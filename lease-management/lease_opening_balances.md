# Lease Opening Balances API

Base route:
`/api/v1/crm/lease-management/leases/{lease}/opening-balances`

## Endpoints

### List lease opening balances
`GET /api/v1/crm/lease-management/leases/{lease}/opening-balances`

Query parameters (Spatie Query Builder v6):
- `filter[id]` (exact)
- `filter[lease_component_id]` (exact)
- `filter[opening_balance_at]` (exact)
- `filter[created_at]` (exact)
- `sort` (comma-separated): `id`, `lease_component_id`, `opening_balance_at`, `created_at`, `amount`, `billed_amount` (prefix with `-` for desc)
- `per_page` (int, optional; defaults to `config('app.query.default_per_page')`)
- `page` (int, optional)

Notes:
- Relationships are eagerly loaded by default: `leaseComponent`, `invoice`, `creditNote`.
- `include` and `fields` parameters are not enabled for this endpoint.

Example:
`GET /api/v1/crm/lease-management/leases/123/opening-balances?filter[lease_component_id]=7&sort=-opening_balance_at`

### Create a lease opening balance
`POST /api/v1/crm/lease-management/leases/{lease}/opening-balances`

Request body:
- `lease_component_id` (int, optional; must exist in `lease_components.id`)
- `amount` (number, required)
- `opening_balance_at` (date, required)
- `notes` (string, optional)

Response:
```json
{
  "message": "Lease opening balance created successfully",
  "opening_balance": {
    "id": 1,
    "lease_component": {
      "id": 7,
      "name": "Base Rent"
    },
    "invoice": {
      "id": 9001,
      "total": 1050
    },
    "credit_note": {
      "id": 3001,
      "total": 100
    },
    "amount": 1200,
    "billed_amount": 0,
    "notes": "Opening balance carried forward",
    "opening_balance_at": {
      "raw": "2024-06-01T00:00:00.000000Z",
      "formatted": "01 Jun, 2024",
      "diff": "2 days ago"
    },
    "created_at": {
      "raw": "2024-06-01T10:00:00.000000Z",
      "formatted": "01 Jun, 2024",
      "diff": "2 days ago"
    }
  }
}
```

### Show a lease opening balance
`GET /api/v1/crm/lease-management/leases/{lease}/opening-balances/{leaseOpeningBalance}`

### Update a lease opening balance
`PUT /api/v1/crm/lease-management/leases/{lease}/opening-balances/{leaseOpeningBalance}`

Request body (same as create):
- `lease_component_id` (int, optional)
- `amount` (number, required)
- `opening_balance_at` (date, required)
- `notes` (string, optional)

Response:
```json
{
  "message": "Lease opening balance updated successfully",
  "opening_balance": {
    "id": 1,
    "amount": 1300,
    "billed_amount": 200
  }
}
```

### Delete a lease opening balance
`DELETE /api/v1/crm/lease-management/leases/{lease}/opening-balances/{leaseOpeningBalance}`

Response:
```json
{
  "message": "Lease opening balance deleted successfully",
  "opening_balance": {
    "id": 1
  }
}
```
