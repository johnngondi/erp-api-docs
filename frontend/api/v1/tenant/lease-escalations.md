# Tenant Lease Escalations API

Base route:

`/api/v1/tenant/leases/{lease}/escalations`

These endpoints are read-only for tenant users.

## Endpoints

- `GET /leases/{lease}/escalations`
- `GET /leases/{lease}/escalations/{leaseEscalation}`

## List Escalations

`GET /api/v1/tenant/leases/{lease}/escalations`

Supported query params:

- Filters:
  - `filter[id]`
  - `filter[lease_component_id]`
  - `filter[start_at]`
  - `filter[cycle]`
  - `filter[period]`
  - `filter[rate]`
  - `filter[next_due]`
  - `filter[created_at]`
- Sort:
  - `sort=id,lease_component_id,start_at,cycle,period,rate,next_due,created_at`
- Pagination:
  - `per_page`
  - `page`

## Access Rules

- If `{lease}` is not owned by the authenticated tenant, return `404`.
- If `{leaseEscalation}` does not belong to `{lease}`, return `404`.

