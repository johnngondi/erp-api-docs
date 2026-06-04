# Lease Exit Notices API

Domain: `Property Management > Lease Management`

Base route:

`/api/v1/app/{company}/property-management/lease-management/exit-notices`

These endpoints allow staff users to manage all lease exit notices across leases.

## Endpoints

- `GET /exit-notices`
- `POST /exit-notices`
- `GET /exit-notices/{tenantExitNotice}`
- `PUT/PATCH /exit-notices/{tenantExitNotice}`
- `DELETE /exit-notices/{tenantExitNotice}`

## List Exit Notices

`GET /api/v1/app/{company}/property-management/lease-management/exit-notices`

Supported query params:

- Filters:
  - `filter[search]`
  - `filter[id]`
  - `filter[lease_id]`
  - `filter[initiated_by_id]`
  - `filter[exit_date]`
  - `filter[status]`
  - `filter[created_at]`
  - `filter[tenant_id]` (mapped to `lease.user_id`)
  - `filter[facility_id]` (mapped to `lease.facility_id`)
  - `filter[lease.user_id]`
  - `filter[lease.facility_id]`
- Sort:
  - `sort=id,lease_id,exit_date,created_at,updated_at,deposit_held,arrears,repair_damages_cost`
- Pagination:
  - `per_page`
  - `page`

Examples:

- `GET /api/v1/app/12/property-management/lease-management/exit-notices?filter[lease.user_id]=77`
- `GET /api/v1/app/12/property-management/lease-management/exit-notices?filter[search]=vacating&sort=-created_at`

## Initiate Exit Notice

`POST /api/v1/app/{company}/property-management/lease-management/exit-notices`

Request body:

| Field | Required | Type | Notes |
|---|---|---|---|
| `lease_id` | Yes | integer | Must exist in `leases.id` |
| `exit_date` | Yes | date string | Format: `YYYY-MM-DD` |
| `reason_for_exit` | Yes | string | Exit reason |
| `supporting_upload_id` | No | integer | Must exist in `uploads.id` |

Notes:

- `deposit_held` and `arrears` are computed automatically.

## Update Exit Notice

`PUT/PATCH /api/v1/app/{company}/property-management/lease-management/exit-notices/{tenantExitNotice}`

Request body may include:

- `exit_date`
- `reason_for_exit`
- `supporting_upload_id`
- `repair_damages_cost`
- `repairs_cost_quote_upload_id`

`repair_damages_cost` and `repairs_cost_quote_upload_id` are intended for staff-side evaluation updates.

## Delete Exit Notice

`DELETE /api/v1/app/{company}/property-management/lease-management/exit-notices/{tenantExitNotice}`

Deletes the notice and associated approval artifacts.
