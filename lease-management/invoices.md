# Invoices API

Base route:
`/api/v1/crm/lease-management/invoices`

## Endpoints

### List invoices
`GET /api/v1/crm/lease-management/invoices`

Query parameters (Spatie Query Builder v6):
- `filter[lease_id]` (exact)
- `filter[due_at]` (exact)
- `filter[cu_reference_number]` (exact)
- `filter[is_credit]` (exact)
- `filter[paid]` (exact)
- `filter[cu_invoice_number]` (exact)
- `filter[cu_serial_number]` (exact)
- `sort` (comma-separated): `lease_id`, `due_at`, `cu_reference_number`, `is_credit`, `paid`, `cu_invoice_number`, `cu_serial_number` (prefix with `-` for desc)
- `per_page` (int, optional; defaults to `config('app.query.default_per_page')`)
- `page` (int, optional)

Notes:
- Relationships are eagerly loaded by default: `invoiceItems`, `lease` (with `user`, `currency`), `invoiceItems.leaseItemComponent` (with `leaseComponent`, `leaseItem.facilitySpace`, `leaseItem.lease.user`).
- `include` and `fields` parameters are not enabled for this endpoint.

Example:
`GET /api/v1/crm/lease-management/invoices?filter[lease_id]=123&sort=-due_at&per_page=10`

### Create an invoice
`POST /api/v1/crm/lease-management/invoices`

Request body:
- `lease_id` (int, required)
- `due_at` (date, required)
- `cu_reference_number` (string, optional)
- `is_credit` (bool, optional; defaults to `false` when omitted)
- `paid` (float, optional; defaults to `0` when omitted)
- `notes` (string, required)
- `cu_invoice_number` (string, optional)
- `cu_serial_number` (string, optional)
- `cu_invoice_verify_url` (string, optional)
- `items` (array, required)
  - `lease_item_component_id` (int, required; must exist in `lease_item_components.id` and belong to `lease_id`)
  - `notes` (string, required)
  - `quantity` (int, required; min `1`)
  - `amount` (float, required; min `0`)
  - `tax_id` (int, required; must exist in `taxes.id`)

Notes:
- Each `lease_item_component_id` in `items` must be unique.

Response:
```json
{
  "message": "Invoice created successfully",
  "invoice": {
    "id": 1,
    "due_at": {
      "raw": "2024-06-30T00:00:00.000000Z",
      "formatted": "30 Jun, 2024",
      "diff": "in 3 weeks"
    },
    "lease": {
      "id": 123,
      "user": {
        "id": 77,
        "name": "John Doe"
      },
      "currency": {
        "id": 1,
        "code": "USD",
        "name": "US Dollar"
      }
    },
    "cu_reference_number": "REF-0001",
    "notes": "June rent",
    "amount": 1000,
    "tax": 50,
    "total": 1050,
    "paid": 0,
    "balance": 1050,
    "cu_invoice_number": "INV-0001",
    "cu_serial_number": "SER-123",
    "cu_invoice_verify_url": "https://example.test/verify/INV-0001",
    "status": "pending",
    "invoice_items": [
      {
        "id": 10,
        "notes": "Rent",
        "quantity": 1,
        "cost": 1000,
        "amount": 1000,
        "tax": 50,
        "total": 1050,
        "paid": 0,
        "balance": 1050
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

### Show an invoice
`GET /api/v1/crm/lease-management/invoices/{invoice}`

### Update an invoice
`PUT /api/v1/crm/lease-management/invoices/{invoice}`

Request body (same as create):
- `lease_id` (int, required)
- `due_at` (date, required)
- `cu_reference_number` (string, optional)
- `is_credit` (bool, optional; defaults to `false` when omitted)
- `paid` (float, optional; defaults to `0` when omitted)
- `notes` (string, required)
- `cu_invoice_number` (string, optional)
- `cu_serial_number` (string, optional)
- `cu_invoice_verify_url` (string, optional)
- `items` (array, required; same rules as create)

Response:
```json
{
  "message": "Invoice updated successfully",
  "invoice": {
    "id": 1,
    "paid": 200,
    "balance": 850,
    "status": "partially paid"
  }
}
```

### Cancel an invoice
`PATCH /api/v1/crm/lease-management/invoices/{invoice}/cancel`

Notes:
- If the invoice is already cancelled, the API returns a validation error.
- Cancelling deletes related lease billing entries, sets status to `cancelled`, and records a tenant statement credit equal to the invoice total.

Response:
```json
{
  "message": "Invoice cancelled successfully",
  "invoice": {
    "id": 1,
    "status": "cancelled"
  }
}
```

### Delete an invoice
`DELETE /api/v1/crm/lease-management/invoices/{invoice}`

Response:
```json
{
  "message": "Invoice deleted successfully",
  "invoice": {
    "id": 1
  }
}
```
