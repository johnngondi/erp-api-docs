# Lease Escalations API

Domain: `Property Management > Lease Management`

Base route:

`/api/v1/app/{company}/property-management/lease-management/leases/{lease}/escalations`

## Endpoints

- `GET /leases/{lease}/escalations`
- `POST /leases/{lease}/escalations`
- `GET /leases/{lease}/escalations/{leaseEscalation}`
- `PUT/PATCH /leases/{lease}/escalations/{leaseEscalation}`
- `DELETE /leases/{lease}/escalations/{leaseEscalation}`

## List Lease Escalations

`GET /api/v1/app/{company}/property-management/lease-management/leases/{lease}/escalations`

Supported query params:

- Filters:
  - `filter[id]`
  - `filter[lease_id]`
  - `filter[lease_component_id]`
  - `filter[start_at]`
  - `filter[cycle]`
  - `filter[period]`
  - `filter[rate]`
  - `filter[next_due]`
  - `filter[created_at]`
- Sort:
  - `sort=id,lease_id,lease_component_id,start_at,cycle,period,rate,next_due,created_at`
- Include: not supported
- Select fields: not supported
- Pagination: `per_page`, `page`

Example:

`GET /api/v1/app/12/property-management/lease-management/leases/101/escalations?filter[lease_component_id]=9&sort=-start_at`

## Create Lease Escalation

`POST /api/v1/app/{company}/property-management/lease-management/leases/{lease}/escalations`

Request body:

| Field | Required | Type | Allowed Values / Notes |
|---|---|---|---|
| `lease_component_id` | Yes | integer | Must exist in `lease_components.id` |
| `start_at` | Yes | date (`YYYY-MM-DD`) | - |
| `cycle` | Yes | string | `days`, `weeks`, `months`, `years` |
| `period` | Yes | integer | Number of cycle units |
| `rate` | Yes | integer | Numeric escalation rate |
| `next_due` | Yes | date (`YYYY-MM-DD`) | - |

Example request:

```json
{
  "lease_component_id": 9,
  "start_at": "2026-03-01",
  "cycle": "months",
  "period": 12,
  "rate": 5,
  "next_due": "2027-03-01"
}
```

Example response:

```json
{
  "message": "Lease escalation created successfully.",
  "lease_escalation": {
    "id": 61,
    "cycle": "months",
    "period": 12,
    "rate": 5
  }
}
```

## Update Lease Escalation

`PUT/PATCH /api/v1/app/{company}/property-management/lease-management/leases/{lease}/escalations/{leaseEscalation}`

Use the same payload shape as create.

## Delete Lease Escalation

`DELETE /api/v1/app/{company}/property-management/lease-management/leases/{lease}/escalations/{leaseEscalation}`

## Frontend Error Handling

Apply shared rules in `docs/frontend/app/README.md`.
