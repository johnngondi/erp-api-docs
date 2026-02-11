# Lease Opening Balances API

Domain: `Property Management > Lease Management`

Base route:

`/api/v1/app/{company}/property-management/lease-management/leases/{lease}/opening-balances`

## Endpoints

- `GET /leases/{lease}/opening-balances`
- `POST /leases/{lease}/opening-balances`
- `GET /leases/{lease}/opening-balances/{leaseOpeningBalance}`
- `PUT/PATCH /leases/{lease}/opening-balances/{leaseOpeningBalance}`
- `DELETE /leases/{lease}/opening-balances/{leaseOpeningBalance}`

## List Lease Opening Balances

`GET /api/v1/app/{company}/property-management/lease-management/leases/{lease}/opening-balances`

Supported query params:

- Filters:
  - `filter[id]`
  - `filter[lease_component_id]`
  - `filter[opening_balance_at]`
  - `filter[created_at]`
- Sort:
  - `sort=id,lease_component_id,opening_balance_at,created_at,amount,billed_amount`
- Include: not supported
- Select fields: not supported
- Pagination: `per_page`, `page`

Example:

`GET /api/v1/app/12/property-management/lease-management/leases/101/opening-balances?sort=-opening_balance_at`

## Create Lease Opening Balance

`POST /api/v1/app/{company}/property-management/lease-management/leases/{lease}/opening-balances`

Request body:

| Field | Required | Type | Allowed Values / Notes |
|---|---|---|---|
| `amount` | Yes | number | - |
| `opening_balance_at` | Yes | date (`YYYY-MM-DD`) | - |
| `lease_component_id` | No | integer | Must exist in `lease_components.id` |
| `notes` | No | string | Optional |

Example request:

```json
{
  "lease_component_id": 9,
  "amount": 200,
  "opening_balance_at": "2026-01-01",
  "notes": "Carried from previous accounting year"
}
```

Example response:

```json
{
  "message": "Lease opening balance created successfully",
  "opening_balance": {
    "id": 88,
    "amount": 200,
    "billed_amount": 0
  }
}
```

## Update Lease Opening Balance

`PUT/PATCH /api/v1/app/{company}/property-management/lease-management/leases/{lease}/opening-balances/{leaseOpeningBalance}`

Use the same payload shape as create.

## Delete Lease Opening Balance

`DELETE /api/v1/app/{company}/property-management/lease-management/leases/{lease}/opening-balances/{leaseOpeningBalance}`

## Frontend Error Handling

Apply shared rules in `docs/frontend/app/README.md`.
