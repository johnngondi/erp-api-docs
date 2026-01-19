# Lease Deposits API

Base route:
`/api/v1/crm/lease-management/leases/{lease}/deposits`

## Endpoints

### List lease deposits
`GET /api/v1/crm/lease-management/leases/{lease}/deposits`

Query parameters (Spatie Query Builder v6):
- `filter[id]` (exact)
- `filter[lease_item_component_id]` (exact)
- `filter[created_at]` (exact)
- `sort` (comma-separated): `id`, `lease_item_component_id`, `created_at` (prefix with `-` for desc)
- `per_page` (int, optional; defaults to `config('app.query.default_per_page')`)
- `page` (int, optional)

Notes:
- Relationships are eagerly loaded by default: `leaseItemComponent`, `leaseItemComponent.leaseComponent`, `invoice`.
- `include` and `fields` parameters are not enabled for this endpoint.

Example:
`GET /api/v1/crm/lease-management/leases/123/deposits?filter[lease_item_component_id]=45&sort=-created_at&per_page=10`

### Create a lease deposit
`POST /api/v1/crm/lease-management/leases/{lease}/deposits`

Request body:
- `lease_item_component_id` (int, required; must exist in `lease_item_components.id`)
- `amount` (float, required)
- `billed` (bool, optional; defaults to `false` when omitted)

Response:
```json
{
  "message": "Lease deposit created successfully",
  "lease_deposit": {
    "id": 1,
    "lease_item_component": {
      "id": 45,
      "lease_component": {
        "id": 7,
        "name": "Security Deposit"
      }
    },
    "amount": 1000,
    "billed": false,
    "invoice": {
      "id": 9001
    },
    "created_at": {
      "raw": "2024-06-01T10:00:00.000000Z",
      "formatted": "01 Jun, 2024",
      "diff": "2 days ago"
    }
  }
}
```

### Show a lease deposit
`GET /api/v1/crm/lease-management/leases/{lease}/deposits/{leaseDeposit}`

### Update a lease deposit
`PUT /api/v1/crm/lease-management/leases/{lease}/deposits/{leaseDeposit}`

Request body (same as create):
- `lease_item_component_id` (int, required)
- `amount` (int, required)
- `billed` (bool, optional; defaults to `false` when omitted)

Response:
```json
{
  "message": "Lease deposit updated successfully",
  "lease_deposit": {
    "id": 1,
    "lease_item_component": {
      "id": 45,
      "lease_component": {
        "id": 7,
        "name": "Security Deposit"
      }
    },
    "amount": 1200,
    "billed": true,
    "invoice": {
      "id": 9001
    },
    "created_at": {
      "raw": "2024-06-01T10:00:00.000000Z",
      "formatted": "01 Jun, 2024",
      "diff": "2 days ago"
    }
  }
}
```

### Delete a lease deposit
`DELETE /api/v1/crm/lease-management/leases/{lease}/deposits/{leaseDeposit}`

Response:
```json
{
  "message": "Lease deposit deleted successfully",
  "lease_deposit": {
    "id": 1,
    "lease_item_component_id": 45,
    "amount": 1200,
    "billed": true
  }
}
```
