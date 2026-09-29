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
  - `filter[facility_id]`, `filter[vendor_id]`, `filter[expense_type_id]`, `filter[expense_category_id]`, `filter[status]`

**Ids, `status` and `type` match exactly.** They were partial until 2026-09-29, so
`filter[status]=paid` also returned `unpaid` and `partially-paid`, and `filter[facility_id]=6`
returned rows for facility 60. Send the whole value. Date filters are still partial, so
`filter[created_at]=2026-09` means that month.

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

## Create payload (`ExpenseData`)

| Field | Required | Type | Allowed Values / Notes |
|---|---|---|---|
| `vendor_id` | Yes | integer | Must exist in `users.id` |
| `facility_id` | Yes | integer | Must exist in `facilities.id` |
| `transaction_at` | Yes | date | - |
| `expense_type_id` | Yes | integer | Must exist in `expense_types.id` |
| `expense_category_id` | No | integer | Must exist in `expense_categories.id`; when omitted, backend derives from `expense_type_id` |
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

## Update payload (`PUT/PATCH /expenses/{expense}`)

Only these six can be changed after an expense is raised. Everything else is settled at creation
and is ignored if sent.

| Field | Required | Type | Allowed Values / Notes |
|---|---|---|---|
| `notes` | No | string \| null | - |
| `invoice_number` | No | string | Cannot be set to empty when sent |
| `invoice_upload_id` | No | integer \| null | Must exist in `uploads.id` |
| `tax_invoice_number` | No | string \| null | - |
| `transaction_at` | No | date | Must not be in the future, and must fall in the current month |
| `expense_category_id` | No | integer \| null | Must exist in `expense_categories.id` |

Notes:

- **Send only what you are changing.** An omitted field keeps its stored value. Editing one field
  no longer requires resending the other five - doing so previously returned a 500.
- The `transaction_at` restriction applies only when you are **moving** the date. An expense raised
  in an earlier month can still have its other fields corrected; its stored date is left untouched
  and is not re-checked.

