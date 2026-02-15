# Tenant Lease Opening Balances API

Base route:

`/api/v1/tenant/leases/{lease}/opening-balances`

These endpoints are read-only for tenant users.

## Endpoints

- `GET /leases/{lease}/opening-balances`
- `GET /leases/{lease}/opening-balances/{leaseOpeningBalance}`

## List Opening Balances

`GET /api/v1/tenant/leases/{lease}/opening-balances`

Supported query params:

- Filters:
  - `filter[id]`
  - `filter[lease_component_id]`
  - `filter[opening_balance_at]`
  - `filter[created_at]`
- Sort:
  - `sort=id,lease_component_id,opening_balance_at,created_at,amount,billed_amount`
- Pagination:
  - `per_page`
  - `page`

## Access Rules

- If `{lease}` is not owned by the authenticated tenant, return `404`.
- If `{leaseOpeningBalance}` does not belong to `{lease}`, return `404`.

