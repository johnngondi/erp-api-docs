# Tenant Lease Exit Notices API

Base route:

`/api/v1/tenant/leases/{lease}/exit-notices`

These endpoints allow tenant users to initiate and track lease exit notices.

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
