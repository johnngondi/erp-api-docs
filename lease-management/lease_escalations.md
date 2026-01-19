# Lease Escalations API

Base route:
`/api/v1/crm/lease-management/leases/{lease}/escalations`

## Endpoints

### List lease escalations
`GET /api/v1/crm/lease-management/leases/{lease}/escalations`

Query parameters (Spatie Query Builder v6):
- `filter[id]` (exact)
- `filter[lease_id]` (exact)
- `filter[lease_component_id]` (exact)
- `filter[start_at]` (exact)
- `filter[cycle]` (exact)
- `filter[period]` (exact)
- `filter[rate]` (exact)
- `filter[next_due]` (exact)
- `filter[created_at]` (exact)
- `sort` (comma-separated): `id`, `lease_id`, `lease_component_id`, `start_at`, `cycle`, `period`, `rate`, `next_due`, `created_at` (prefix with `-` for desc)
- `per_page` (int, optional; defaults to `config('app.query.default_per_page')`)
- `page` (int, optional)

Notes:
- Relationships are eagerly loaded by default: `lease`, `lease.user`, `leaseComponent`.
- `include` and `fields` parameters are not enabled for this endpoint.

Example:
`GET /api/v1/crm/lease-management/leases/123/escalations?filter[lease_component_id]=7&sort=-start_at&per_page=10`

### Create a lease escalation
`POST /api/v1/crm/lease-management/leases/{lease}/escalations`

Request body:
- `lease_component_id` (int, required; must exist in `lease_components.id`)
- `start_at` (date, required)
- `cycle` (string, required; one of: `days`, `weeks`, `months`, `years`)
- `period` (int, required)
- `rate` (int, required)
- `next_due` (date, required)

Response:
```json
{
  "message": "Lease escalation created successfully.",
  "lease_escalation": {
    "id": 1,
    "lease_component": {
      "id": 7,
      "name": "Base Rent"
    },
    "start_at": {
      "raw": "2024-06-01T00:00:00.000000Z",
      "formatted": "01 Jun, 2024",
      "diff": "2 days ago"
    },
    "cycle": "months",
    "period": 12,
    "rate": 5,
    "next_due": {
      "raw": "2025-06-01T00:00:00.000000Z",
      "formatted": "01 Jun, 2025",
      "diff": "in 1 year"
    },
    "created_at": {
      "raw": "2024-06-01T10:00:00.000000Z",
      "formatted": "01 Jun, 2024",
      "diff": "2 days ago"
    }
  }
}
```

### Show a lease escalation
`GET /api/v1/crm/lease-management/leases/{lease}/escalations/{leaseEscalation}`

### Update a lease escalation
`PUT /api/v1/crm/lease-management/leases/{lease}/escalations/{leaseEscalation}`

Request body (same as create):
- `lease_component_id` (int, required)
- `start_at` (date, required)
- `cycle` (string, required; one of: `days`, `weeks`, `months`, `years`)
- `period` (int, required)
- `rate` (int, required)
- `next_due` (date, required)

Response:
```json
{
  "message": "Lease escalation updated successfully.",
  "lease_escalation": {
    "id": 1,
    "lease_component": {
      "id": 7,
      "name": "Base Rent"
    },
    "start_at": {
      "raw": "2024-06-01T00:00:00.000000Z",
      "formatted": "01 Jun, 2024",
      "diff": "2 days ago"
    },
    "cycle": "months",
    "period": 12,
    "rate": 6,
    "next_due": {
      "raw": "2025-06-01T00:00:00.000000Z",
      "formatted": "01 Jun, 2025",
      "diff": "in 1 year"
    },
    "created_at": {
      "raw": "2024-06-01T10:00:00.000000Z",
      "formatted": "01 Jun, 2024",
      "diff": "2 days ago"
    }
  }
}
```

### Delete a lease escalation
`DELETE /api/v1/crm/lease-management/leases/{lease}/escalations/{leaseEscalation}`

Response:
```json
{
  "message": "Lease escalation deleted successfully.",
  "lease_escalation": {
    "id": 1,
    "lease_component_id": 7,
    "start_at": "2024-06-01T00:00:00.000000Z",
    "cycle": "months",
    "period": 12,
    "rate": 6,
    "next_due": "2025-06-01T00:00:00.000000Z"
  }
}
```
