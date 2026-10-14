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
  - `filter[lease_component_id]` (exact)
  - `filter[lease_item_component_id]`
  - `filter[created_at]`
- Sort:
  - `sort=id,lease_component_id,lease_item_component_id,created_at`
- Pagination:
  - `per_page`
  - `page`

Each row carries **`lease_component`** - what the deposit is held against (Rent, Service Charge,
Parking). Show that as the deposit's type: `lease_item_component` is the lease line it was billed
on, which for a deposit taken since row 143 is the lease's single Deposit line.

## Access Rules

- If `{lease}` is not owned by the authenticated tenant, return `404`.
- If `{leaseDeposit}` does not belong to `{lease}`, return `404`.

