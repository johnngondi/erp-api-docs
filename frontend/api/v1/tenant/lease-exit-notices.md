# Tenant Lease Exit Notices API

These endpoints allow tenant users to initiate and track lease exit notices.
They are exposed two ways:

- **Nested** under a specific lease: `/api/v1/tenant/leases/{lease}/exit-notices`
- **Top-level** across all the authenticated tenant's leases: `/api/v1/tenant/exit-notices`
  (mirrors the property-management `exit-notices` resource)

Both are backed by the same controller and apply the same access rules. A tenant
can only ever see or act on notices for leases they own.

---

## Nested resource

Base route:

`/api/v1/tenant/leases/{lease}/exit-notices`

## Endpoints

- `GET /leases/{lease}/exit-notices`
- `POST /leases/{lease}/exit-notices`
- `GET /leases/{lease}/exit-notices/{tenantExitNotice}`
- `PUT/PATCH /leases/{lease}/exit-notices/{tenantExitNotice}`

## List Exit Notices

`GET /api/v1/tenant/leases/{lease}/exit-notices`

Supported query params:

- Filters:
  - `filter[search]`
  - `filter[id]`
  - `filter[exit_date]`
  - `filter[status]`
  - `filter[created_at]`
  - `filter[initiated_by_id]`
  - `filter[lease.user_id]`
  - `filter[lease.facility_id]`
- Sort:
  - `sort=id,exit_date,created_at,deposit_held,arrears,repair_damages_cost`
- Pagination:
  - `per_page`
  - `page`

Example:

`GET /api/v1/tenant/leases/101/exit-notices?filter[search]=relocation&sort=-created_at`

## Initiate Exit Notice

`POST /api/v1/tenant/leases/{lease}/exit-notices`

Request body:

| Field | Required | Type | Notes |
|---|---|---|---|
| `exit_date` | Yes | date string | Format: `YYYY-MM-DD` |
| `reason_for_exit` | Yes | string | Reason provided by tenant |
| `supporting_upload_id` | No | integer | Must exist in `uploads.id` |

Notes:

- `deposit_held` is computed automatically from paid lease deposit invoices.
- `arrears` is computed automatically from unpaid/partially paid lease invoices.

## Update Exit Notice

`PUT/PATCH /api/v1/tenant/leases/{lease}/exit-notices/{tenantExitNotice}`

Tenant can update only:

- `exit_date`
- `supporting_upload_id`

## Access Rules

- Tenant can only access notices for leases they own.
- If `{lease}` is not owned by authenticated tenant, API returns `404`.
- If `{tenantExitNotice}` does not belong to `{lease}`, API returns `404`.

---

## Top-level resource

Base route:

`/api/v1/tenant/exit-notices`

Same flow as the nested resource, but not scoped to a single lease in the URL.
Useful for listing or managing exit notices across every lease the tenant owns.

## Endpoints

- `GET /exit-notices`
- `POST /exit-notices`
- `GET /exit-notices/{tenantExitNotice}`
- `PUT/PATCH /exit-notices/{tenantExitNotice}`

## List Exit Notices

`GET /api/v1/tenant/exit-notices`

Returns every exit notice belonging to the authenticated tenant's leases. Each
item includes a `lease` object (with `facility` and `currency`) so the notice can
be identified without a lease in the URL.

Supported query params:

- Filters:
  - `filter[search]`
  - `filter[id]`
  - `filter[lease_id]`
  - `filter[exit_date]`
  - `filter[status]`
  - `filter[created_at]`
  - `filter[initiated_by_id]`
  - `filter[lease.user_id]`
  - `filter[lease.facility_id]`
- Sort:
  - `sort=id,lease_id,exit_date,created_at,deposit_held,arrears,repair_damages_cost`
- Pagination:
  - `per_page`
  - `page`

Example:

`GET /api/v1/tenant/exit-notices?filter[lease_id]=101&sort=-created_at`

## Initiate Exit Notice

`POST /api/v1/tenant/exit-notices`

Request body:

| Field | Required | Type | Notes |
|---|---|---|---|
| `lease_id` | Yes | integer | Must exist in `leases.id` and be owned by the tenant |
| `exit_date` | Yes | date string | Format: `YYYY-MM-DD` |
| `reason_for_exit` | Yes | string | Reason provided by tenant |
| `supporting_upload_id` | No | integer | Must exist in `uploads.id` |

Notes:

- `deposit_held` and `arrears` are computed automatically.
- If `lease_id` is not owned by the authenticated tenant, API returns `404`.

## Show Exit Notice

`GET /api/v1/tenant/exit-notices/{tenantExitNotice}`

## Update Exit Notice

`PUT/PATCH /api/v1/tenant/exit-notices/{tenantExitNotice}`

Tenant can update only:

- `exit_date`
- `supporting_upload_id`

## Access Rules (top-level)

- Listing only ever returns notices for leases the authenticated tenant owns.
- If `{tenantExitNotice}` belongs to a lease the tenant does not own, API returns `404`.
