# Expenses API

Domain: `Property Management > Finance`

Base route:

`/api/v1/app/{company}/property-management/finance/expenses`

## Endpoints

- `GET /expenses`
- `POST /expenses`
- `GET /expenses/{expense}`
- `PUT/PATCH /expenses/{expense}`
- `DELETE /expenses/{expense}`
- `PATCH /expenses/{expense}/mark-paid`
- `PATCH /expenses/{expense}/cancel`

## List Expenses

`GET /api/v1/app/{company}/property-management/finance/expenses`

Supported query params:

- Filters:
  - `filter[search]` (Scout-backed search; supports CSV IDs and invoice numbers)
  - `filter[facility_id]`, `filter[vendor_id]`, `filter[expense_type_id]`, `filter[status]`
  - `filter[created_at]`, `filter[transaction_at]`
- Sort:
  - `sort=id,invoice_number,amount,tax,total,paid,balance,transaction_at,created_at,updated_at`
- Include:
  - `include=taxType,invoiceUpload,creatingUser,bill`
- Pagination:
  - `per_page`, `page`

Enum filter options:

- `filter[status]`: `pending`, `unpaid`, `partially-paid`, `paid`, `cancelled` (from `ExpenseStatus` enum)

Sample list response (`FacilityExpenseResource`):

```json
{
  "data": [
    {
      "id": 501,
      "notes": "Electrical repairs",
      "invoice_number": "EXP-2026-001",
      "tax_invoice_number": "TEXP-2026-001",
      "amount": "800.00",
      "tax_amount": "128.00",
      "total": "928.00",
      "paid": "100.00",
      "balance": "828.00",
      "transaction_at": {
        "raw": "2026-05-04T00:00:00.000000Z",
        "formatted": "04 May, 2026",
        "diff": "5 days ago"
      },
      "vendor": { "id": 7, "name": "Acme Vendor" },
      "facility": { "id": 22, "name": "Riverside Plaza" },
      "status": { "value": "partially-paid", "color": "primary" },
      "created": {
        "raw": "2026-05-09T10:12:00.000000Z",
        "formatted": "09 May, 2026",
        "diff": "moments ago"
      },
      "updated": {
        "raw": "2026-05-09T10:12:00.000000Z",
        "formatted": "09 May, 2026",
        "diff": "moments ago"
      }
    }
  ]
}
```

## Create/Update payload (`ExpenseData`)

| Field | Required | Type | Allowed Values / Notes |
|---|---|---|---|
| `vendor_id` | Yes | integer | Must exist in `users.id` |
| `facility_id` | Yes | integer | Must exist in `facilities.id` |
| `transaction_at` | Yes | date | - |
| `expense_type_id` | Yes | integer | Must exist in `expense_types.id` |
| `invoice_number` | Yes | string | - |
| `amount` | Yes | number | - |
| `status` | No | string | `pending`, `unpaid`, `partially-paid`, `paid`, `cancelled` |
| `creating_user_id` | No | integer | Must exist in `users.id` |
| `notes` | No | string | Optional |
| `bill_id` | No | integer | Must exist in `bills.id` |
| `invoice_upload_id` | No | integer | Must exist in `uploads.id` |
| `tax_invoice_number` | No | string | Optional |
| `currency_id` | No | integer | Must exist in `currencies.id` |
| `tax_id` | No | integer | Must exist in `taxes.id` |
| `paid` | No | number | Default `0` |
