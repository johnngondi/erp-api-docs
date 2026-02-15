# Invoices API

Domain: `Property Management > Lease Management`

Base route:

`/api/v1/app/{company}/property-management/lease-management/invoices`

## Endpoints

- `GET /invoices`
- `POST /invoices`
- `GET /invoices/{invoice}`
- `PUT/PATCH /invoices/{invoice}`
- `PATCH /invoices/{invoice}/cancel`
- `DELETE /invoices/{invoice}`

## List Invoices

`GET /api/v1/app/{company}/property-management/lease-management/invoices`

Supported query params:

- Filters:
  - `filter[lease_id]`
  - `filter[due_at]`
  - `filter[cu_reference_number]`
  - `filter[is_credit]`
  - `filter[paid]`
  - `filter[cu_invoice_number]`
  - `filter[cu_serial_number]`
  - `filter[id]`
  - `filter[status]`
  - `filter[user_id]` (maps to lease tenant)
- Sort:
  - `sort=lease_id,due_at,cu_reference_number,is_credit,paid,cu_invoice_number,cu_serial_number`
- Include: not supported
- Select fields: not supported
- Pagination: `per_page`, `page`

Examples:

- `GET /api/v1/app/12/property-management/lease-management/invoices?filter[lease_id]=101&sort=-due_at&per_page=10`
- `GET /api/v1/app/12/property-management/lease-management/invoices?filter[status]=pending`

## Create Invoice

`POST /api/v1/app/{company}/property-management/lease-management/invoices`

Request body:

| Field | Required | Type | Allowed Values / Notes |
|---|---|---|---|
| `lease_id` | Yes | integer | - |
| `due_at` | Yes | date (`YYYY-MM-DD`) | - |
| `notes` | Yes | string | - |
| `items` | Yes | array | At least one item |
| `cu_reference_number` | No | string | Optional |
| `is_credit` | No | boolean | Defaults to `false` |
| `paid` | No | number | Defaults to `0` |
| `cu_invoice_number` | No | string | Optional |
| `cu_serial_number` | No | string | Optional |
| `cu_invoice_verify_url` | No | string | Optional |

`items[]` fields:

| Field | Required | Type | Allowed Values / Notes |
|---|---|---|---|
| `lease_item_component_id` | Yes | integer | Must belong to the supplied lease |
| `notes` | Yes | string | - |
| `quantity` | Yes | integer | Minimum `1` |
| `amount` | Yes | number | Minimum `0` |
| `tax_id` | Yes | integer | Must exist in `taxes.id` |

Notes:

- `lease_item_component_id` must be unique within the same request.

Example request:

```json
{
  "lease_id": 101,
  "due_at": "2026-03-01",
  "notes": "March 2026 billing",
  "cu_reference_number": "INV-REF-3001",
  "items": [
    {
      "lease_item_component_id": 455,
      "notes": "Base rent",
      "quantity": 1,
      "amount": 950,
      "tax_id": 1
    }
  ]
}
```

Example response:

```json
{
  "message": "Invoice created successfully",
  "invoice": {
    "id": 9001,
    "lease": { "id": 101 },
    "amount": 950,
    "tax": 152,
    "total": 1102,
    "status": { "value": "pending", "color": "secondary" }
  }
}
```

## Update Invoice

`PUT/PATCH /api/v1/app/{company}/property-management/lease-management/invoices/{invoice}`

Use the same payload shape as create.

## Cancel Invoice

`PATCH /api/v1/app/{company}/property-management/lease-management/invoices/{invoice}/cancel`

No request body required.

Possible status values returned by API resource:

- `pending` (`secondary`)
- `unpaid` (`warning`)
- `partially paid` (`primary`)
- `paid` (`success`)
- `cancelled` (`danger`)

## Delete Invoice

`DELETE /api/v1/app/{company}/property-management/lease-management/invoices/{invoice}`

## Frontend Error Handling

Apply shared rules in `docs/frontend/app/README.md`.
