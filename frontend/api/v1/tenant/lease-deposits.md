# Tenant Lease Deposits API

Base route:

`/api/v1/tenant/leases/{lease}/deposits`

These endpoints are read-only for tenant users.

## Endpoints

- `GET /leases/{lease}/deposits`
- `GET /leases/{lease}/deposits/{leaseDeposit}`

## List Deposits

`GET /api/v1/tenant/leases/{lease}/deposits`

Supported query params:

- Filters:
  - `filter[id]`
  - `filter[lease_item_component_id]`
  - `filter[created_at]`
- Sort:
  - `sort=id,lease_item_component_id,created_at`
- Pagination:
  - `per_page`
  - `page`

## Access Rules

- If `{lease}` is not owned by the authenticated tenant, return `404`.
- If `{leaseDeposit}` does not belong to `{lease}`, return `404`.

