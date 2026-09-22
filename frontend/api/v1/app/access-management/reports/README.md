# Access Management Reports

Access Management owns company-wide audit and approval workflow reporting. These endpoints are not registered in the Property Management report registry or its discovery response.

## System Reports navigation

Add **System Reports → Audit Trail** under Access Management. Use the explicit `view-audit-trail-report` permission to show the page and `export-audit-trail-report` independently for its download action. Read the current caller's granted permission names from `GET /api/v1/app/{company}/access-management` → `data.permissions`. The permission catalog under `/access-management/permissions` is for role configuration, not a list of the caller's grants. The new permissions belong to resource `Audit Trail` and tag `Access Management`. A user with export only may download without gaining list access.

| Entry | Method and URL | View permission | Export permission |
| --- | --- | --- | --- |
| [Audit Trail](audit-trail.md) | `GET /api/v1/app/{company}/access-management/reports/system/audit-trail` | `view-audit-trail-report` | `export-audit-trail-report` |
| [Workflow Performance](workflow-performance.md) | `GET /api/v1/app/{company}/access-management/reports/system/workflow-performance` | `view-workflow-performance-report` | `export-workflow-performance-report` |

Audit Trail is a **paginated list with a custom diff view**, not a standard report-envelope page. Do not send it through the generic report/bucket renderer. The server-side Access Management registry is separate from the Property Management registry; this does not add a public discovery endpoint. The frontend should configure the navigation entry above explicitly.

Workflow Performance uses the standard report-envelope renderer with one `overview` bucket. Show its page and export action using their independent permissions, just as for Audit Trail.

The dashboard's existing `recent_activity` is a separate Telescope summary. Use the new Audit Trail endpoint for persisted activity records and diffs.

## Backend registry and permissions

`config/access-management-reports.php` declares Access Management reports. The corresponding
`App\Services\AccessManagement\Reports\Support\ReportRegistry` lists report keys, route names,
response types and independent view/export permissions. `AuthServiceProvider::registerReportAbilities()`
reads both module registries and defines their gates through the same registration loop.

Audit Trail declares `response: paginated`; registry membership does not turn its response into a
report envelope. Its view/export permission names remain unchanged and are still seeded in
`storage/app/seeders/permissions.json`. Grant them to the appropriate company roles after seeding.
A new Access Management report should add its definition here, its routes and its seeded permissions;
do not add standalone gates to the provider. Workflow Performance declares `response: report`.
Route names refer to the list endpoint; exports use the same name with `.export` appended.
After adding registry entries, clear or rebuild Laravel’s cached configuration (`php artisan config:clear`
or the deployment’s `config:cache` step) so the new definitions are loaded.

This separate registry follows the PM's review clarification and supersedes the milestone MD's
instruction to define Access Management gates outside `registerReportAbilities()`. Audit Trail
remains outside `config/reports.php` and the Property Management discovery response.
