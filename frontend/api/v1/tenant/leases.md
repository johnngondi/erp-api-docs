# Tenant Leases API

Base route:

`/api/v1/tenant/leases`

These endpoints are read-only for tenant users.

## Endpoints

- `GET /leases`
- `GET /leases/{lease}`

## List Leases

`GET /api/v1/tenant/leases`

Supported query params:

- Filters:
  - `filter[facility_id]`
  - `filter[status]`
  - `filter[created_at]`
  - `filter[start_at]`
  - `filter[end_at]`
  - `filter[next_due_at]`
  - `filter[billing_cycle]`
  - `filter[user_id]`
- Sort:
  - `sort=user_id,status,created_at,start_at,end_at,next_due_at,billing_cycle`
- Pagination:
  - `per_page`
  - `page`

## Show Lease

`GET /api/v1/tenant/leases/{lease}`

If `{lease}` does not belong to the authenticated tenant, the API returns `404`.

