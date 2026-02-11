# Credit Notes API

Domain: `Property Management > Lease Management`

Base route:

`/api/v1/app/{company}/property-management/lease-management/credit-notes`

## Endpoints

- `GET /credit-notes`
- `POST /credit-notes`
- `GET /credit-notes/{creditNote}`
- `PUT/PATCH /credit-notes/{creditNote}`
- `PATCH /credit-notes/{creditNote}/cancel`
- `DELETE /credit-notes/{creditNote}`

## List Credit Notes

`GET /api/v1/app/{company}/property-management/lease-management/credit-notes`

Supported query params:

- Filters:
  - `filter[id]`
  - `filter[lease_id]`
  - `filter[invoice_id]`
  - `filter[credit_note_id]`
  - `filter[due_at]`
  - `filter[cu_reference_number]`
  - `filter[is_credit]`
  - `filter[paid]`
  - `filter[cu_invoice_number]`
  - `filter[status]`
- Sort:
  - `sort=lease_id,invoice_id,credit_note_id,due_at,cu_reference_number,is_credit,paid,cu_invoice_number,status`
- Include: not supported
- Select fields: not supported
- Pagination: `per_page`, `page`

Example:

`GET /api/v1/app/12/property-management/lease-management/credit-notes?filter[lease_id]=101&sort=-due_at`

## Create Credit Note

`POST /api/v1/app/{company}/property-management/lease-management/credit-notes`

Request body:

| Field | Required | Type | Allowed Values / Notes |
|---|---|---|---|
| `lease_id` | Yes | integer | - |
| `due_at` | Yes | date (`YYYY-MM-DD`) | - |
| `notes` | Yes | string | - |
| `items` | Yes | array | At least one item |
| `invoice_id` | No | integer | Optional |
| `cu_reference_number` | No | string | Optional |
| `is_credit` | No | boolean | Defaults to `true` |
| `paid` | No | number | Defaults to `0` |
| `cu_invoice_number` | No | string | Optional |
| `cu_serial_number` | No | string | Optional |
| `cu_credit_note_verify_url` | No | string | Optional |

`items[]` fields:

| Field | Required | Type | Allowed Values / Notes |
|---|---|---|---|
| `lease_item_component_id` | Yes | integer | Must belong to supplied lease |
| `notes` | Yes | string | - |
| `quantity` | Yes | integer | Minimum `1` |
| `amount` | Yes | number | Minimum `0` |
| `tax_id` | Yes | integer | Must exist in `taxes.id` |

Notes:

- `lease_item_component_id` must be unique within the same request.
- Response exposes `cu_invoice_verify_url` in resource output.

Example request:

```json
{
  "lease_id": 101,
  "due_at": "2026-03-01",
  "notes": "Overcharge adjustment",
  "items": [
    {
      "lease_item_component_id": 455,
      "notes": "Base rent correction",
      "quantity": 1,
      "amount": 120,
      "tax_id": 1
    }
  ]
}
```

Example response:

```json
{
  "message": "Credit note created successfully.",
  "credit_note": {
    "id": 410,
    "total": 139.2,
    "status": { "value": "pending", "color": "secondary" }
  }
}
```

## Update Credit Note

`PUT/PATCH /api/v1/app/{company}/property-management/lease-management/credit-notes/{creditNote}`

Use the same payload shape as create.

## Cancel Credit Note

`PATCH /api/v1/app/{company}/property-management/lease-management/credit-notes/{creditNote}/cancel`

No request body required.

Possible status values returned by API resource:

- `pending` (`secondary`)
- `applied` (`success`)
- `cancelled` (`danger`)

## Delete Credit Note

`DELETE /api/v1/app/{company}/property-management/lease-management/credit-notes/{creditNote}`

## Frontend Error Handling

Apply shared rules in `docs/frontend/app/README.md`.
