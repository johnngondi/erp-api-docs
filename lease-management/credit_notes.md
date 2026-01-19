# Credit Notes API

Base route:
`/api/v1/crm/lease-management/credit-notes`

## Endpoints

### List credit notes
`GET /api/v1/crm/lease-management/credit-notes`

Query parameters (Spatie Query Builder v6):
- `filter[lease_id]` (exact)
- `filter[invoice_id]` (exact)
- `filter[credit_note_id]` (exact)
- `filter[due_at]` (exact)
- `filter[cu_reference_number]` (exact)
- `filter[is_credit]` (exact)
- `filter[paid]` (exact)
- `filter[cu_invoice_number]` (exact)
- `filter[cu_serial_number]` (exact)
- `sort` (comma-separated): `lease_id`, `invoice_id`, `credit_note_id`, `due_at`, `cu_reference_number`, `is_credit`, `paid`, `cu_invoice_number`, `cu_serial_number` (prefix with `-` for desc)
- `per_page` (int, optional; defaults to `config('app.query.default_per_page')`)
- `page` (int, optional)

Notes:
- Relationships are eagerly loaded by default: `creditNoteItems`, `lease` (with `currency`, `user`), `creditNoteItems.leaseItemComponent` (with `leaseComponent`, `leaseItem.facilitySpace`).
- `include` and `fields` parameters are not enabled for this endpoint.

Example:
`GET /api/v1/crm/lease-management/credit-notes?filter[lease_id]=123&sort=-due_at&per_page=10`

### Create a credit note
`POST /api/v1/crm/lease-management/credit-notes`

Request body:
- `lease_id` (int, required)
- `due_at` (date, required)
- `cu_reference_number` (string, optional)
- `is_credit` (bool, optional; defaults to `true` when omitted)
- `paid` (float, optional; defaults to `0` when omitted)
- `notes` (string, required)
- `cu_invoice_number` (string, optional)
- `cu_serial_number` (string, optional)
- `cu_credit_note_verify_url` (string, optional)
- `items` (array, required)
  - `lease_item_component_id` (int, required; must exist in `lease_item_components.id` and belong to `lease_id`)
  - `notes` (string, required)
  - `quantity` (int, required; min `1`)
  - `amount` (float, required; min `0`)
  - `tax_id` (int, required; must exist in `taxes.id`)

Notes:
- Each `lease_item_component_id` in `items` must be unique.
- Responses use `cu_invoice_verify_url` (resource field) even if the request uses `cu_credit_note_verify_url`.

Response:
```json
{
  "message": "Credit note created successfully.",
  "credit_note": {
    "id": 1,
    "due_at": {
      "raw": "2024-06-30T00:00:00.000000Z",
      "formatted": "30 Jun, 2024",
      "diff": "in 3 weeks"
    },
    "amount": 500,
    "tax": 25,
    "total": 525,
    "paid": 0,
    "balance": 525,
    "cu_invoice_number": "CN-0001",
    "cu_serial_number": "SER-123",
    "cu_invoice_verify_url": "https://example.test/verify/CN-0001",
    "status": "pending",
    "credit_note_items": [
      {
        "id": 10,
        "notes": "Rent adjustment",
        "quantity": 1,
        "cost": 500,
        "amount": 500,
        "tax": 25,
        "total": 525
      }
    ],
    "created_at": {
      "raw": "2024-06-01T10:00:00.000000Z",
      "formatted": "01 Jun, 2024",
      "diff": "2 days ago"
    }
  }
}
```

### Show a credit note
`GET /api/v1/crm/lease-management/credit-notes/{creditNote}`

### Update a credit note
`PUT /api/v1/crm/lease-management/credit-notes/{creditNote}`

Request body (same as create):
- `lease_id` (int, required)
- `due_at` (date, required)
- `cu_reference_number` (string, optional)
- `is_credit` (bool, optional; defaults to `true` when omitted)
- `paid` (int, optional; defaults to `0` when omitted)
- `notes` (string, required)
- `cu_invoice_number` (string, optional)
- `cu_serial_number` (string, optional)
- `cu_credit_note_verify_url` (string, optional)
- `items` (array, required; same rules as create)

Response:
```json
{
  "message": "Credit note updated successfully",
  "invoice": {
    "id": 1,
    "paid": 100,
    "balance": 425,
    "status": "partially paid"
  }
}
```

### Cancel a credit note
`PATCH /api/v1/crm/lease-management/credit-notes/{creditNote}/cancel`

Notes:
- If the credit note is already cancelled, the API returns a validation error.
- Cancelling deletes related lease billing entries, sets status to `cancelled`, and records a tenant statement debit equal to the credit note total.

Response:
```json
{
  "message": "Credit note cancelled successfully",
  "credit-note": {
    "id": 1,
    "status": "cancelled"
  }
}
```

### Delete a credit note
`DELETE /api/v1/crm/lease-management/credit-notes/{creditNote}`

Response:
```json
{
  "message": "Credit note deleted successfully",
  "invoice": {
    "id": 1
  }
}
```
