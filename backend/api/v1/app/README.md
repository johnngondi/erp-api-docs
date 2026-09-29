# App API Docs (Backend Reference)

Scope: `app` routes only.

Base prefix:
```text
/api/v1/app/{company}
```

This docs set is a technical reference generated from routes, DTOs, and resources for Property Management, Facility Management, Project Management, and App Users modules.

## Contents
- `docs/backend/api/v1/app/routes-catalog.md`: full route catalog (method + URI + controller).
- `docs/backend/api/v1/app/query-capabilities.md`: list endpoint query options (`filter`, `sort`, `include`, `fields`).
- `docs/backend/api/v1/app/request-contracts.md`: request DTO field contracts (required/optional + enum inputs + validation hints).
- `docs/backend/api/v1/app/enums.md`: enum values and color mappings from code.
- `docs/backend/api/v1/app/receipt-reallocation.md`: moving a confirmed receipt's money between a tenant's invoices.
- `docs/backend/api/v1/app/receipt-backdate.md`: changing a confirmed receipt's date (receipt, tenant statement, cashbook and collection dates) and the remittance rules that block it.
- `docs/backend/api/v1/app/bill-backdate.md`: changing a posted bill's posting date (bill `expense_posted_at`, expense `transaction_at`/`created_at`) and the remittance rules that block it.
- `docs/backend/api/v1/app/invoice-backdate.md`: changing an issued invoice's ledger date (tenant statement `transaction_at`/`created_at`, lease billing `transaction_date`/`created_at`).
- `docs/backend/api/v1/app/tenants.md`: tenant CRUD — group-membership elevation on create, deactivation on delete.
- `docs/backend/api/v1/app/inbox-and-permissions.md`: counts-only inbox summary (every portal) and the light `me/permissions` read.

## Auth and Headers
Required headers for almost all app routes:
- `Authorization: Bearer <token>`
- `Accept: application/json`
- `Content-Type: application/json` (for body requests)

## Company Scope (Multi-Tenancy)
All app routes are company-scoped through `{company}`.

Frontend rule:
- Always call endpoints with the current company id.
- Never reuse URLs cached from another company session.
- If company context changes, clear cached list/detail responses and reload.

## Request Contract Rules
Use `docs/backend/api/v1/app/request-contracts.md` as source of truth for payload fields.

Conventions:
- `Required = Yes`: field must be sent.
- `Required = No`: optional field.
- `Allowed Values (Enum)`: accepted enum string values for input.
- `Rules` column may include conditional requirements such as `RequiredIf`, `RequiredUnless`.

Example (enum input):
```json
{
  "status": "active"
}
```

## List Querying (filter/sort/include/fields)
Use `docs/backend/api/v1/app/query-capabilities.md` to check exactly which endpoint supports what.

### Filter
```http
GET /api/v1/app/12/property-management/lease-management/leases?filter[status]=active
```

### Sort
```http
GET /api/v1/app/12/facility-management/fleet/vehicles?sort=-created_at,license_plate
```

### Include
```http
GET /api/v1/app/12/property-management/finance/bills?include=items
```

### Sparse Fields
```http
GET /api/v1/app/12/property-management/facilities?fields[facilities]=id,name,status
```

### Pagination
```http
GET /api/v1/app/12/project-management/projects?per_page=20&page=2
```

Notes:
- If a list endpoint does not advertise `include`/`fields` in the matrix, do not send those params.
- Invalid filter/sort/include/fields will typically return a `4xx` validation-style response.

## Static Example Responses
These are realistic static examples based on current API resources.

### Project item (status enum with color)
```json
{
  "id": 44,
  "title": "Parking Expansion",
  "status": {
    "value": "in-progress",
    "color": "primary"
  },
  "estimated_start_date": {
    "raw": "2026-01-10T00:00:00.000000Z",
    "formatted": "10 Jan, 2026",
    "diff": "1 month ago"
  }
}
```

### Ticket item (priority/status enums with color)
```json
{
  "id": 901,
  "title": "Water leak at Block A",
  "ticket_type": "facility",
  "priority": {
    "value": "high",
    "color": "danger"
  },
  "status": {
    "value": "open",
    "color": "primary"
  }
}
```

### Lease item (status enum with color)
```json
{
  "id": 101,
  "billing_cycle": "monthly",
  "status": {
    "value": "active",
    "color": "success"
  },
  "next_due_at": {
    "raw": "2026-03-01T00:00:00.000000Z",
    "formatted": "01 Mar, 2026",
    "diff": "in 2 weeks"
  }
}
```

## Nested instance permissions

`App\Http\Resources\Concerns\ResolvesInstancePermissions` offers three ways to
emit a resource's `permissions` map:

| Method | Emitted |
|---|---|
| `resolveInstancePermissions()` | always |
| `resolveInstancePermissionsUnlessListed()` | unless the resource was built inside a row of a top-level list |
| `resolveInstancePermissionsUnlessNested()` | only when the resource is the response subject (a list row or a show record) |

Both conditional variants honour `?with_permissions=1` and
`ResourceNesting::keepNestedPermissions($request)`, which an endpoint calls when its
client reads nested permissions from list rows (lease deposits and opening balances
do). Nesting is detected only through resources that use the trait: a parent that
does not use it renders its children as if they were the subject.

- `UnlessListed`: `LeaseResource`, `FacilityResource`, `FacilityInvoiceResource`,
  `FacilityTicketResource`, `FacilityAssetResource`, `FacilityProcurementRequestResource`.
- `UnlessNested`: `BankAccountResource`, `ApprovalStepResource`, `FacilitySpaceResource`,
  `LeaseItemResource`, `LeaseItemComponentResource`, `LooResource`, `LooTemplateResource`.

### Gate checks and Spatie's `before` hook

Spatie registers a `Gate::before` hook that calls `User::checkPermissionTo($ability)`
for every ability, policy abilities (`view`, `sign`, ...) included. `User` overrides
`checkPermissionTo()` to answer a name that is not a permission from an index of
permission names (built once per loaded Spatie permission collection) instead of
letting `findByName()` scan the collection and throw `PermissionDoesNotExist`.
Permission names still go through `hasPermissionTo()`, which memoizes its answers
per guard, name and company. `hasPermissionTo()` itself still throws for an unknown
name.

## Keeping list resources query-free

- `ResolvesInstancePermissions::resolveApprovalStepsForSubject()` renders `approval_steps`
  only for the response subject (not a list row, not nested) or with
  `?with_approval_steps=1`. Every approvable resource uses it instead of calling
  `getApprovalSteps()` per row.
- `FacilityInvoice::hasAllocatedPayments()` reads `has_allocated_payments` when the query
  added `withExists('payments as has_allocated_payments')` (invoice index and show do);
  both `FacilityInvoiceResource` and `FacilityInvoicePolicy::delete` use it.
- `LeaseResource` emits the five lease figures inside another list's rows only when
  `Lease::withTotals()` preloaded them (`Lease::hasPreloadedTotals()`).
- `PropertyAllocation::allows()` answers a one-hop path (`lease.facility_id`) from a loaded
  relation that carries the column, so narrow eager loads include the allocation key:
  `lease:...,facility_id`, `ticket:...,facility_id`, `asset:...,facility_id`,
  `procurementRequest:...,facility_id`, `receiptAllocations.invoice:id,lease_id,...`.
- `FacilityInvoiceSummaryResource` renders an invoice referenced from a credit note or a
  receipt allocation.
- `Approvable::currentApprovalStepPreferLoaded()` answers from a loaded `approvalSteps`
  relation; the remittance index preloads it for the `cancel` policy.
- Invoice and credit-note item spaces preload `leaseItemComponents` through
  `FacilitySpace::withAllocatedComponents()`.

## Error Handling Guidelines (Frontend)

### 4xx (show to user)
Treat as user-actionable whenever possible.

- `400 Bad Request`: malformed request/query params.
- `401 Unauthorized`: not authenticated. Redirect to login / refresh token flow.
- `403 Forbidden`: authenticated but not allowed (permission/company/portal mismatch).
- `404 Not Found`: missing/invalid resource id in current scope.
- `409 Conflict`: state conflict (if returned by endpoint behavior).
- `422 Unprocessable Entity`: validation/business-rule error.
- `429 Too Many Requests`: throttled, retry later.

Recommended UI behavior:
- Render `message` and field-level `errors` (for validation).
- Keep form data when possible.
- Do not show generic crash toast for 4xx.

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

### 5xx (generic message)
Treat as backend/system failure.

Recommended UI behavior:
- Show generic message: `Something went wrong. Please try again.`
- Optionally provide retry action.
- Log request id / endpoint / payload fingerprint for debugging.
- Do not expose raw backend exception text to end users.

## Enum Colors
For all enum badge/chip colors, use:
- `docs/backend/api/v1/app/enums.md`

If an endpoint returns `{ "value": "...", "color": "..." }`, prefer the backend-provided `color` directly.
