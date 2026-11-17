# System Audit Trail

**This endpoint paginates and does not return the standard report envelope.** It returns `data` as a list, plus Laravel `links` and `meta`. There are no top-level `header`, `report`, `bucket`, `fields`, or `items` keys.

## Endpoints and authorization

- `GET /api/v1/app/{company}/access-management/reports/system/audit-trail`
- `GET /api/v1/app/{company}/access-management/reports/system/audit-trail/export?format=excel|pdf`
- `GET /api/v1/app/{company}/access-management/reports/system/audit-trail/subject-types` — options for the Subject Type filter
- `GET /api/v1/app/{company}/access-management/reports/system/audit-trail/users` — options for the User filter

Routes are named `app.access-management.reports.system.audit-trail` and `.export`. These use the PROP-241 ticket's `reports/system` path; the initial milestone document's shorter `reports/audit-trail` path is not an alias.

Use Sanctum bearer authentication, an active staff account, and access to the route company. Listing and both filter-option endpoints require `view-audit-trail-report`; export requires `export-audit-trail-report`, independently. Missing permissions return 403. The old Property Management `/property-management/reports/system/audit-trail` endpoint does not exist.

## Query parameters

Use **nested `filter[...]` parameters**, not flat Property Management report filters.

| Parameter | Meaning | Default |
| --- | --- | --- |
| `filter[user_id]` | One or more user IDs of the causer (not the subject's user), comma-separated (`1,130`). Anything but positive whole numbers returns 422 on `filter.user_id`. Options: [`/users`](#filter-options) | All causers |
| `filter[subject_type]` | One or more stored morph types, comma-separated, e.g. `App\Models\FacilityInvoice,App\Models\Lease`. Options: [`/subject-types`](#filter-options) | All subject types |
| `filter[event]` | One or more of `created`, `updated`, `deleted`, `restored`, comma-separated (`updated,created`). Any other value returns 422 on `filter.event` | All events |
| `filter[search]` | Free text, up to 100 characters. See [Searching](#searching) | No search |
| `filter[resource_type]` + `filter[resource_id]` | The history of one record and its lines, as a row's `resource_type` + `resource_id` name it (`FacilityInvoice` + `220`). Send both; an unknown type returns 422 on `filter.resource_type`. See [History of one record](#history-of-one-record) | All records |
| `filter[created_between]` | Two real dates `Y-m-d,Y-m-d`, start must not follow end | Unbounded |
| `sort` | `created_at` or `-created_at` | `-created_at` |
| `per_page` | Integer 1–1000 | `config(app.query.default_per_page)` |
| `page` | Positive integer | 1 |

The date window includes the entire end date, in the application timezone (`Africa/Nairobi` by default). Invalid date ranges return 422 with `errors["filter.created_between"]`. Timestamp ties use descending activity ID so page order is stable. Pagination links preserve the active query parameters. Unrecognized QueryBuilder filters/sorts return 400.

Postman example (enter query parameters in the Params tab so backslashes and brackets are encoded correctly):

```text
GET {{appUrl}}/access-management/reports/system/audit-trail
filter[event] = updated,created
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
    "resource_type": "FacilityInvoice",
    "resource_id": 9911,
    "resource_kind": null,
    "changes": {
      "old": {"notes": "Water", "facility_id": 4},
      "new": {"notes": "Water charge", "facility_id": 5},
      "labels": {"notes": "Notes", "facility_id": "Facility"}
    },
    "created": {"raw": "2026-09-18T08:02:19.000000Z", "formatted": "18 Sep, 2026 11:02", "diff": "1 hour ago"}
  }],
  "links": {"first": "...", "last": "...", "prev": null, "next": null},
  "meta": {"current_page": 1, "from": 1, "last_page": 1, "per_page": 25, "to": 1, "total": 1}
}
```

IDs use the existing activity table's integer primary key. `causer` may be null for system work or a deleted actor. A queued change retains the person who dispatched the job; company context is captured at dispatch as well.

Descriptions come from the existing model describer services. Labels are captured when the activity is logged, so renaming or deleting the subject does not rewrite its historical label. Older records fall back to the model's readable class name when a label is unavailable.

`changes.old` and `changes.new` are objects, empty as `{}` when that side is absent. `changes.labels` names every key that appears in either side (`facility_id` is "Facility", `cu_invoice_number` is "CU Invoice Number"), so a drawer can show labels instead of column names. Values are not translated: `facility_id` stays the id. Values are the recorded values, not current subject values; decimal strings, IDs, booleans and arrays retain their stored shape. Ignored secrets and timestamp noise are excluded, on old rows as well as new (a description written before a column was hidden reads "duplicate token hash changed", without the value): passwords and tokens, the receipt `duplicate_token_hash` (a one-time confirmation hash) and the reconciliation `statement_token` never appear in the drawer or the export. Render descriptions as text, not HTML.

**Opening the record: use `resource_type` + `resource_id`.** They name the record whose page a row opens, in the same form as notifications: the short model name and its id. A child change points at its parent: an invoice-item row carries `FacilityInvoice` and the invoice id, not the item. Both are null when the record has no page, so show no link then. Older rows carry them too, including lines deleted since (linked from what their earlier rows recorded).

| A change on | Opens (`resource_type`) |
| --- | --- |
| Invoice, invoice item | `FacilityInvoice` |
| Bill, bill item | `FacilityBill` |
| Receipt, receipt allocation | `FacilityReceipt` |
| Payment voucher | `FacilityPaymentVoucher`, with `resource_kind` |
| Remittance | `FacilityRemittance` |
| Ticket, ticket entry | `FacilityTicket` |
| Exit notice, exit notice item | `TenantExitNotice` |
| Lease, lease item, lease item component | `Lease` |
| Expense · Budget (and its lines) · Customer invoice (and its lines) · Property · User | `FacilityExpense` · `FacilityBudget` · `CustomerInvoice` · `Facility` · `User` |

**`resource_kind`** is set only for payment vouchers, which open different pages by payee: the voucher's `payable_as`, one of `vendor` (supplier), `landlord`, `tenant` or `other`. It is null on every other row.

`resource_url` is kept for compatibility. It is relative, with no host, scheme, leading slash or API/company prefix, and is a backend path that does not necessarily match a frontend route; prefer `resource_type` + `resource_id`. It may be null where the record has no mapped page.

A child edit records the child activity and touches the parent's `updated_at`. The existing foundation deliberately suppresses timestamp-only parent log entries; it does not fabricate a second row merely because the parent was touched.

## Searching

`filter[search]` is a contains-match, not case sensitive, over:

- the **description**, which names the record and spells out what changed ("Updated Invoice INV-0012: notes from Water to Water charge");
- the **subject label** (the record's number or name, e.g. `INV-0012`);
- the **user's name**.

A search made only of digits also matches the activity id and the subject id exactly. Commas and `%` are plain text, not lists or wildcards. The search combines with every other filter (user, subject type, event, dates, record), and the export takes it too, so the file holds exactly the rows on screen.

The raw `changes` values are not searched. Update descriptions list up to a handful of changes and end "and N more", so a value in a long change set may not be found. Narrow by date for the fastest search.

```text
GET {{appUrl}}/access-management/reports/system/audit-trail?filter[search]=INV-0012
GET {{appUrl}}/access-management/reports/system/audit-trail?filter[search]=Wanjiru&filter[event]=updated
```

## History of one record

`filter[resource_type]=FacilityInvoice&filter[resource_id]=220` returns everything that happened to invoice 220: rows about the invoice itself and rows about its lines (invoice items), whichever link the row carries. Use it for a "History" link from a record's page: pass the same `resource_type` + `resource_id` a row returns. Types that have a page are listed in the table under [Row shape](#row-shape). A line deleted before it was linked to its record may not appear.

```text
GET {{appUrl}}/access-management/reports/system/audit-trail?filter[resource_type]=FacilityBill&filter[resource_id]=38
```

## Filter options

**Subject types:** `GET …/audit-trail/subject-types` lists every type this company's trail holds, sorted by label. Send `value` back unchanged in `filter[subject_type]` (comma-separate several). Where two types read the same, the class name is added to tell them apart.

```json
{"data": [
  {"value": "App\\Models\\FacilityInvoiceItem", "label": "Invoice Item"},
  {"value": "App\\Models\\LeaseComponent", "label": "Lease Component (LeaseComponent)"},
  {"value": "App\\Models\\LeaseItemComponent", "label": "Lease Component (LeaseItemComponent)"},
  {"value": "App\\Models\\Facility", "label": "Property"}
]}
```

**Users:** `GET …/audit-trail/users` lists the people who appear as the actor in this company's trail, from any portal (staff, tenants, landlords, vendors), sorted by name and paginated (`links` / `meta`, `per_page`, `page`). `filter[search]` matches name, email or phone. `portals` lists the user groups the person belongs to. Send `id` as `filter[user_id]`, comma-separated for several.

```json
{"data": [
  {"id": 130, "name": "Wanjiru Njoroge", "email": "wanjiru@example.com", "portals": ["tenant"]},
  {"id": 7, "name": "Test Reviewer", "email": "reviewer@example.com", "portals": ["app"]}
], "links": {...}, "meta": {...}}
```

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
