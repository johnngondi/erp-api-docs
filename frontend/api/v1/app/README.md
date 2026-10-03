# App API Frontend Reference

This reference is for frontend teams integrating app endpoints.

Base URL pattern:

`/api/v1/app/{company}`

Example with company `12`:

`/api/v1/app/12/property-management/lease-management/leases`

## Authentication

Use these headers on authenticated requests:

- `Authorization: Bearer <token>`
- `Accept: application/json`
- `Content-Type: application/json` (for JSON payloads)

## Multi-Tenancy Safety

All app routes are company scoped (`{company}`).

- Always send the currently selected company in the URL.
- Do not reuse cached URLs from a different company session.
- If company changes, clear cached lists/details and refetch.

## List Requests: Filter, Sort, Include, Select Fields

Most list endpoints use Spatie Query Builder:

- Filter: `filter[field]=value`
- Sort: `sort=field,-another_field`
- Include relations: `include=relation1,relation2` (only if endpoint supports includes)
- Select specific fields: `fields[resource]=id,name,status` (only if endpoint supports fields)
- Pagination: `per_page=20&page=2`

Examples:

- Filter + sort:
  - `GET /api/v1/app/12/property-management/lease-management/leases?filter[status]=active&sort=-start_at`
- Include:
  - `GET /api/v1/app/12/property-management/finance/bills?include=items`
- Select fields:
  - `GET /api/v1/app/12/property-management/facilities?fields[facilities]=id,name,status`

If an endpoint does not support `include` or `fields`, sending those params returns a `4xx` error.

## Nested `permissions` on list rows

Every row of a list keeps its own `permissions` map. A resource nested **inside** a
list row (an invoice row's `lease`, a lease row's `facility`, a bill row's
`facility`, a payment voucher row's `debit_bank_account`) does **not** carry
`permissions`, because resolving them for every row was the bulk of the cost of a
list response.

Show responses are unaffected: a nested resource on a single record keeps its
`permissions` (the invoice page can still read `invoice.lease.permissions.view`).

- Opt back in on a list with `?with_permissions=1` when a table genuinely needs
  a nested row's permissions.
- `GET .../leases/{lease}/deposits` and `GET .../leases/{lease}/opening-balances`
  always include nested `invoice` / `credit_note` permissions, because their rows
  link to them.
- Treat a missing nested `permissions` as "unknown", not as `true`; guard with
  `row.lease?.permissions?.view` and fall back to the row's own permissions.

Resources that omit `permissions` when nested in a list row: lease, property
(facility), invoice, ticket, asset, procurement request. Bank accounts and
approval steps omit it whenever nested (list or show); read an approval step's
`can_act` / `is_current`, which are always present.

## List payloads: what nested records carry

To keep list endpoints to a fixed number of queries, some fields appear only where the
record is the subject of the response:

- `approval_steps` — show responses only; `?with_approval_steps=1` adds it to list rows.
- A lease nested inside another list's rows (an invoice's or credit note's `lease`) omits
  `rent_per_month`, `service_charge_per_month`, `current_arrears`, `opening_balance` and
  `total_deposit_amount`. The lease list and lease show keep them, and so does a lease
  nested in a single record (the exit notice page).
- A credit note's `invoice` and a receipt allocation's `invoice` are an **invoice summary**:
  `id`, `notes`, `cu_invoice_number`, `amount`, `total`, `paid`, `balance`, `status`,
  `due_at` (each only when loaded) and `permissions.view` (show responses only). Fetch the
  invoice itself for anything else.
- An upload's `creator` is present only when the endpoint loads it (lease application
  documents do).

## Wording: Supplier and Property

The words the API *says* have changed. **Nothing it takes or returns as data has.**

| Reads as | Used to read |
|---|---|
| Supplier | Vendor |
| Property | Facility |

What this covers: response `message` strings, validation messages, permission labels
(`name`, `resource`, `tag`), notification text and activity log lines.

What is deliberately unchanged, and will stay that way:

- Routes — `/users/vendors`, `/property-management/facilities/...`
- Payload and response field names — `facility_id`, `vendor_id`, `facility_type_id`
- Validation error **keys** — a message about a property is still keyed `facility_id`
- Permission slugs — `create-facility-ticket` is still what a permission check resolves
- Columns, models and enum values

So a client that maps errors by key, checks permissions by slug, or reads `facility_id` off a
response needs no change. Only text shown to a person is different.

### Two consequences worth knowing

**A validation message will not match its key.** `facility_id` can come back with "The property
field is required." That is intended — do not key error display off the words.

**Permission labels are rendered, not stored.** `permissions.name` returns "Create Property
Ticket" while the underlying slug stays `create-facility-ticket`. Match on the slug, never on the
label.

### Some messages dropped an internal prefix

Messages that leaked model names now read as plain English:

| Now | Was |
|---|---|
| `Bill created successfully.` | `FacilityBill created successfully.` |
| `Ticket withdrawn successfully.` | `FacilityTicket withdrawn successfully.` |
| `Expense marked as paid successfully.` | `FacilityExpense marked as paid successfully.` |

If any screen matches on message text rather than status code, it needs updating. Matching on
message text is worth removing anyway.

## Error Handling (Frontend Behavior)

### 4xx errors (show to user)

Treat `4xx` as user-actionable:

- `400`: malformed request/query
- `401`: unauthenticated
- `403`: forbidden
- `404`: resource not found in current company scope
- `409`: conflict/state issue
- `422`: validation/business rule error
- `429`: rate limited

Recommended UI behavior:

- Show backend `message`.
- If available, show field-level `errors`.
- Keep form state where possible.

Typical 422 shape:

```json
{
  "message": "The given data was invalid.",
  "errors": {
    "field_name": [
      "Validation message"
    ]
  }
}
```

### 5xx errors (generic message)

Treat `5xx` as backend/system failure:

- Show a generic message like: `Something went wrong. Please try again.`
- Offer retry.
- Log endpoint + request correlation info client-side for support.
- Do not show raw exception text to users.

## Known API Caveats (As of April 8, 2026)

- `Facility Management > Sensors > Entries` routes exist but backend controller is not implemented yet.
  - Endpoints: `/api/v1/app/{company}/facility-management/sensors/{sensor}/entries...`
- `Access Management` has partial coverage on `user-groups`.
  - `user-groups`: only list is implemented.
- `Users` module has partial CRUD coverage on some resources.
  - `landlords`, `tenants`, `vendors`: update routes exist but are not implemented.
  - `tenants` delete route responds, but deletion logic is not fully wired.
- Integrate frontend for these caveats with graceful fallbacks:
  - Hide unsupported actions in UI where possible.
  - If shown, handle `4xx/5xx` responses with the error strategy above.

## Property Management Docs

- Overview:
  - `docs/frontend/app/property-management/README.md`
- Dashboard:
  - `docs/frontend/app/property-management/dashboard.md`
- Facilities:
  - `docs/frontend/app/property-management/facilities/facilities.md`
  - `docs/frontend/app/property-management/facilities/floor_generation.md`
  - `docs/frontend/app/property-management/facilities/budget-settings.md`
- Tickets:
  - `docs/frontend/app/property-management/tickets.md`
- Procurement:
  - `docs/frontend/app/property-management/procurement.md`
- Finance:
  - `docs/frontend/app/property-management/finance.md`
- Settings:
  - `docs/frontend/app/property-management/settings.md`
- Lease Management:
  - `docs/frontend/app/property-management/lease-management/leases.md`
  - `docs/frontend/app/property-management/lease-management/invoices.md`
  - `docs/frontend/app/property-management/lease-management/credit-notes.md`
  - `docs/frontend/app/property-management/lease-management/receipts.md`
  - `docs/frontend/app/property-management/lease-management/lease-deposits.md`
  - `docs/frontend/app/property-management/lease-management/lease-escalations.md`
  - `docs/frontend/app/property-management/lease-management/lease-opening-balances.md`
  - `docs/frontend/app/property-management/lease-management/charges-imports.md`

## Facility Management Docs

- Overview:
  - `docs/frontend/app/facility-management/README.md`
- Assets:
  - `docs/frontend/app/facility-management/assets.md`
- Fleet:
  - `docs/frontend/app/facility-management/fleet.md`
- Sensors:
  - `docs/frontend/app/facility-management/sensors.md`
- Power Management:
  - `docs/frontend/app/facility-management/power-management.md`
- Utility Management:
  - `docs/frontend/app/facility-management/utility-management.md`
- Settings:
  - `docs/frontend/app/facility-management/settings.md`

## Project Management Docs

- Overview:
  - `docs/frontend/app/project-management/README.md`
- Projects:
  - `docs/frontend/app/project-management/projects.md`

## Access Management Docs

- Overview:
  - `docs/frontend/api/v1/app/access-management/README.md`
- User Groups:
  - `docs/frontend/api/v1/app/access-management/user-groups.md`
- Permissions:
  - `docs/frontend/api/v1/app/access-management/permissions.md`
- Roles:
  - `docs/frontend/api/v1/app/access-management/roles.md`
- Companies:
  - `docs/frontend/api/v1/app/access-management/companies.md`
- Company Users:
  - `docs/frontend/api/v1/app/access-management/company-users.md`

## Alerts, Pending Tasks and Notifications Docs

All three are readable from every portal, are scoped to the signed-in user, and need no permission.
`domain` and `filter[domain]` exist on the App portal only.

- App (staff):
  - `docs/frontend/api/v1/app/alerts.md`
  - `docs/frontend/api/v1/app/pending-tasks.md`
  - `docs/frontend/api/v1/app/notifications.md`
- Tenant:
  - `docs/frontend/api/v1/tenant/alerts.md`
  - `docs/frontend/api/v1/tenant/pending-tasks.md`
  - `docs/frontend/api/v1/tenant/notifications.md`
- Landlord:
  - `docs/frontend/api/v1/landlord/alerts.md`
  - `docs/frontend/api/v1/landlord/pending-tasks.md`
  - `docs/frontend/api/v1/landlord/notifications.md`
- Vendor:
  - `docs/frontend/api/v1/vendor/alerts.md`
  - `docs/frontend/api/v1/vendor/pending-tasks.md`
  - `docs/frontend/api/v1/vendor/notifications.md`

## Users Docs

- Overview:
  - `docs/frontend/app/users/README.md`
- Landlords:
  - `docs/frontend/app/users/landlords.md`
- Tenants:
  - `docs/frontend/app/users/tenants.md`
- Vendors:
  - `docs/frontend/app/users/vendors.md`
- User group/role/permission docs:
  - See `Access Management Docs` section above.
