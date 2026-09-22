# System Audit Trail

**This endpoint paginates and does not return the standard report envelope.** It returns `data` as a list, plus Laravel `links` and `meta`. There are no top-level `header`, `report`, `bucket`, `fields`, or `items` keys.

## Endpoints and authorization

- `GET /api/v1/app/{company}/access-management/reports/system/audit-trail`
- `GET /api/v1/app/{company}/access-management/reports/system/audit-trail/export?format=excel|pdf`

Routes are named `app.access-management.reports.system.audit-trail` and `.export`. These use the PROP-241 ticket's `reports/system` path; the initial milestone document's shorter `reports/audit-trail` path is not an alias.

Use Sanctum bearer authentication, an active staff account, and access to the route company. Listing requires `view-audit-trail-report`; export requires `export-audit-trail-report`, independently. Missing permissions return 403. The old Property Management `/property-management/reports/system/audit-trail` endpoint does not exist.

## Query parameters

Use **nested `filter[...]` parameters**, not flat Property Management report filters.

| Parameter | Meaning | Default |
| --- | --- | --- |
| `filter[user_id]` | Exact user ID of the causer (not the subject's user) | All causers |
| `filter[subject_type]` | Exact stored morph type, normally a fully qualified class such as `App\Models\FacilityInvoiceItem` | All subject types |
| `filter[event]` | `created`, `updated`, `deleted`, or `restored` | All events |
| `filter[created_between]` | Two real dates `Y-m-d,Y-m-d`, start must not follow end | Unbounded |
| `sort` | `created_at` or `-created_at` | `-created_at` |
| `per_page` | Integer 1–1000 | `config(app.query.default_per_page)` |
| `page` | Positive integer | 1 |

The date window includes the entire end date, in the application timezone (`Africa/Nairobi` by default). Invalid date ranges return 422 with `errors["filter.created_between"]`. Timestamp ties use descending activity ID so page order is stable. Pagination links preserve the active query parameters. Unrecognized QueryBuilder filters/sorts return 400.

Postman example (enter query parameters in the Params tab so backslashes and brackets are encoded correctly):

```text
GET {{appUrl}}/access-management/reports/system/audit-trail
filter[event] = updated
filter[subject_type] = App\Models\FacilityInvoiceItem
filter[created_between] = 2026-09-01,2026-09-30
sort = -created_at
per_page = 25
page = 1
```

## Row shape

```json
{
  "data": [{
    "id": 123,
    "event": "updated",
    "causer": {"id": 7, "name": "Test Reviewer"},
    "subject_type": "App\\Models\\FacilityInvoiceItem",
    "subject_id": 5512,
    "subject_label": "Invoice Item",
    "description": "Updated the water charge line on Invoice INV-0012",
    "resource_url": "lease-management/billing/invoices/9911",
    "changes": {"old": {"notes": "Water"}, "new": {"notes": "Water charge"}},
    "created": {"raw": "2026-09-18T08:02:19.000000Z", "formatted": "18 Sep, 2026 11:02", "diff": "1 hour ago"}
  }],
  "links": {"first": "...", "last": "...", "prev": null, "next": null},
  "meta": {"current_page": 1, "from": 1, "last_page": 1, "per_page": 25, "to": 1, "total": 1}
}
```

IDs use the existing activity table's integer primary key. `causer` may be null for system work or a deleted actor. A queued change retains the person who dispatched the job; company context is captured at dispatch as well.

Descriptions come from the existing model describer services. Labels are captured when the activity is logged, so renaming or deleting the subject does not rewrite its historical label. Older records fall back to the model's readable class name when a label is unavailable.

`changes.old` and `changes.new` are objects, empty as `{}` when that side is absent. Values are the recorded values, not current subject values; decimal strings, IDs, booleans and arrays retain their stored shape. Ignored secrets and timestamp noise are excluded. Render descriptions as text, not HTML.

`resource_url` is relative, with no host, scheme, leading slash or API/company prefix. It may be null where the record has no mapped page. A child's URL points to its configured parent (e.g. an invoice item's invoice). The frontend resolves it using its existing application URL mapping.

A child edit records the child activity and touches the parent's `updated_at`. The existing foundation deliberately suppresses timestamp-only parent log entries; it does not fabricate a second row merely because the parent was touched.

## Company isolation and history

The report covers all modules, including Access Management. Every query requires the persisted `activity_log.company_id` to equal the route company. It never infers access from the causer's memberships. Null-company records never appear.

At logging time, a subject's company takes precedence, including the company reached through its configured audit parent. A Company record attributes to itself. For subjects without company ownership, the current route/queued company context is the fallback. A background change without either source remains unattributed. The saved attribution remains available after the subject is deleted.

## Exports

Append `/export?format=excel` or `/export?format=pdf`, and supply the same nested filters and sorting. Export includes **all matching company rows**, ignoring the selected page and page size. Excel includes the activity ID, timestamp, actor, subject, readable description, resource URL and both change objects (serialized as JSON text). PDF uses the same columns and shared report renderer.

Unsupported or missing `format` returns 422 keyed to `format`. Export is a file response, never a JSON report envelope. The shared file writers use an internal presentation envelope only; this does not change the list API's contract. Large exports still depend on the shared writers' memory and PDF rendering limits.

## Deployment / local setup

After checking the intended database connection:

```bash
php artisan migrate
php artisan db:seed --class=PermissionsSeeder
php artisan permission:cache-reset
php artisan audit:backfill-companies
php artisan queue:restart
```

PDF export also requires the project's Node dependencies and Chrome. On the local Ubuntu setup used for this ticket, run `npm ci --ignore-scripts` and use the installed binaries:

```dotenv
DOCUMENT_TEMPLATES_NODE_BINARY=/usr/bin/node
DOCUMENT_TEMPLATES_NPM_BINARY=/usr/bin/npm
DOCUMENT_TEMPLATES_CHROME_PATH=/usr/bin/google-chrome
```

Use actual paths on each server; do not copy local paths into staging without checking them. Clear cached configuration and restart queue workers after changing these settings.

Grant the new permissions to the intended company roles; seeding permission names does not automatically grant them to existing roles. The repeatable backfill reads surviving subjects (including soft-deleted records) and configured parents in batches. It never guesses from today's request context or actor memberships. Missing or unresolvable subjects remain null and hidden. No historical record is deleted by this command.

The six 14.5.1 indexes remain; a new `(company_id, created_at)` index supports this endpoint. Retention/archival policy is a follow-up ticket, outside PROP-241.
