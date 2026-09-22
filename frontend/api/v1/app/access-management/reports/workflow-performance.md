# Workflow Performance

`GET /api/v1/app/{company}/access-management/reports/system/workflow-performance`

`GET /api/v1/app/{company}/access-management/reports/system/workflow-performance/export?format=excel|pdf`

Use `view-workflow-performance-report` for the page and `export-workflow-performance-report` independently for downloads. Both permissions are seeded under Access Management. Company membership and permission checks apply before querying; both approval sources are explicitly restricted to the route company. Add this page beneath Access Management → System Reports. It is not registered in `config/reports.php` or the Property Management discovery endpoint.

Unlike Audit Trail, this endpoint uses the standard `data.header`, `data.report` (array), `data.summary` envelope, with one `overview` bucket. There is no pagination. The header includes period, filters and generation timestamp, but no property or currency. Exports use the same filters and complete result.

## Query parameters

All parameters are flat, not `filter[...]`.

| Parameter | Meaning | Default |
| --- | --- | --- |
| `period_from` | Inclusive first-step prompt date, `Y-m-d` | Start of current month |
| `period_to` | Inclusive first-step prompt date, `Y-m-d` | End of current month |
| `landlord_id` | Property owner ID | All |
| `facility_id` | Property ID belonging to this company | All |
| `facility_type_id` | Property type ID | All |
| `properties_status` | Property status from the shared `Status` enum | All |
| `mechanism` | `approvable` or `procurement` | Both |
| `approval_template_id` | Generic template belonging to this company | All |
| `approvable_type` | Fully qualified class from `config/approvals.php`, e.g. `App\Models\Loo` | All |
| `format` | Export only: `excel` or `pdf` | Required for export |

Setting either `approval_template_id` or `approvable_type` excludes procurement rows. Combining either with `mechanism=procurement` produces an empty report. Unsupported formats, invalid/reversed dates, foreign templates and unsupported mechanisms return 422. All six standard filters are supported, following the ticket rather than the attached Markdown’s narrower filter list. Property filters intersect and apply to the associated property’s current owner, type and status. Invalid IDs/statuses return 422; foreign-company property IDs also return 422. With property filters active, approvals without a surviving linked property are excluded; without those filters their company-scoped history remains included. Date boundaries use the application's timezone.

Example:

```http
GET {{appUrl}}/access-management/reports/system/workflow-performance?period_from=2026-09-01&period_to=2026-09-30&mechanism=approvable
Accept: application/json
Authorization: Bearer {{authToken}}
```

Property resolution:

| Approval document | Property relationship |
| --- | --- |
| Invoice, credit note, tenant exit notice | `lease.facility` |
| Receipt, remittance, contract, LOO | `facility` |
| Procurement LPO | `procurementRequest.facility` |
| Procurement request | `facility` |

## Mechanisms and normalization

Three business scenarios are covered by two underlying mechanisms and one common aggregation:

- **Generic approvals:** `ApprovalStep` in `approval_steps`; action timestamp is `acted_at`.
- **Letter of Offer approvals:** also `ApprovalStep`, with `approvable_type=App\Models\Loo`. They appear as their own template rows with `mechanism=approvable`. The old `loo_approval_steps` stack was removed; there is no separate `loo` mechanism.
- **Procurement approvals:** `FacilityProcurementRequestStep` in `facility_procurement_request_steps`; action timestamp is `reviewed_at`. Their mechanism is `procurement`.

The shared normalizer lives in `App\Services\AccessManagement\Reports\Support`. Every source yields workflow identity, entity type, chain identity, step position/label, prompted/action timestamps, status, final-step flag and cascade flag. A single metrics implementation aggregates these observations.

Generic rows group by approval template and approvable type. Procurement rows group by stored template-group identity. New procurement steps snapshot the group ID and label when attached, preserving attribution after a rename/deletion. Historical steps without that snapshot are matched only when exactly one same-company template group has the same ordered copied step signature. This historical match is an inference from current templates. Ambiguous/unmatched chains are grouped by their copied-step signature and labelled `Historical procurement workflow <reference>`; they are not assigned to the facility's current default template.

## Exact metrics

- **Chain:** generic `(approvable_type, approvable_id, attempt)`; procurement is one request. Generic resubmission creates another chain and preserves the rejected attempt.
- **Period membership:** the currently stored `prompted_at` of the first materialized step. Once a chain qualifies, all its steps are included even if an action occurs outside the period. Chains whose first step was never prompted are excluded. Re-prompting the first step overwrites this timestamp in the existing approval system; the report cannot reconstruct its earlier prompt date.
- **Started:** number of qualifying chains, not number of documents or steps. Auto-approved documents with no materialized approval steps are outside this report.
- **Completed:** chains that reached final-step approval with no remaining open steps, not individual approved steps.
- **Rejected:** chains containing a rejected step, counted once. Generic rejection marks later open steps rejected too; these do not create additional rejected chains.
- **Pending:** nonterminal chains; this includes generic `pending`/`review` and procurement `pending`/`review`/`waiting-quotes`/`quotes-review`. Incomplete historical chains with no recorded terminal outcome also remain pending.
- **Turnaround:** `(acted_at - prompted_at) / 3600`, using `reviewed_at` for procurement. The stored prompt is the latest prompt after review/re-prompt. Only acted-on `approved`, `rejected` or `review` steps with valid nonnegative intervals contribute. Never-prompted, open, missing-action and stale-action-before-latest-prompt steps are excluded; actual zero-hour actions are included.
- **Cascades:** generic `reject()` acts on the earliest open step and cascades to later open steps. Only the first rejected step in chain order contributes rejection turnaround; cascaded steps are excluded even if they have an action timestamp.
- **Average turnaround:** compute each workflow instance’s mean eligible step duration, then average those instance means. Each instance with at least one eligible action has equal weight regardless of its number of steps. Instances with no eligible durations remain in the counts but not the average denominator. This includes valid acted-on steps in still-pending chains. Round only the displayed result to two decimal places.
- **Longest turnaround:** maximum eligible step duration in hours.
- **Slowest step:** step position/role (or procurement title) with the highest average eligible turnaround within the workflow. It is not necessarily the single longest observation.

Every row and total satisfies `started = completed + rejected + pending`. The subtotal/summary recomputes the average from all underlying eligible workflow-instance means, weighting workflow rows by the number of instances with a measurable turnaround. This follows the ticket’s workflow-instance weighting, rather than weighting by the number of steps. It never averages rounded row averages. No eligible durations means `null`, not an invented zero.

## Response and rendering

`data.report[0]` always contains `bucket`, `header`, `fields`, `items`, and `summary`. Its bucket is `overview`. Columns are `workflow`, `mechanism`, `started`, `completed`, `rejected`, `pending`, `avg_turnaround_hours`, `longest_turnaround_hours`, `slowest_step`.

All row cells follow `{ "value": ... }`. Normal rows use `type=normal`; the final total row uses `type=subtotal`, `background_color=secondary`, and a Total cell spanning workflow/mechanism. Field presentation is returned by the backend; `mechanism` has `togglable=true`. Hour columns use the shared decimal format `money` but represent hours, with no currency symbol. Empty results still return all column definitions and a zero-count subtotal, with null duration metrics.

## Deployment and verification

Run the new migration before writing procurement steps, seed `PermissionsSeeder`, clear permission caches, and grant the appropriate role permissions. PDF rendering uses the same Node/Chrome environment as other reports. No historical data is mutated by report reads or by the migration.

Verify generic, LOO and procurement fixtures; a three-step rejection must count once; resubmission must count a second chain; a partially approved chain must remain pending. Compare workflows with different numbers of instances and different step counts per instance to verify that totals use instance weights rather than step weights or unweighted row averages. Verify both export permissions separately, date boundaries, company isolation and invalid-format 422 responses.
