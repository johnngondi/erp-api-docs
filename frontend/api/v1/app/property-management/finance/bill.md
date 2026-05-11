# Bills API

Domain: `Property Management > Finance`

Base route:

`/api/v1/app/{company}/property-management/finance/bills`

## Endpoints

- `GET /bills`
- `POST /bills`
- `GET /bills/{bill}`
- `PUT/PATCH /bills/{bill}`
- `DELETE /bills/{bill}`
- `PATCH /bills/{bill}/cancel`
- `PUT /bills/{bill}/post-bill`
- `PUT /bills/{bill}/reject-invoice`

## List Bills

`GET /api/v1/app/{company}/property-management/finance/bills`

Supported query params:

- Filters:
  - `filter[search]` (Scout-backed search; supports CSV IDs and invoice numbers)
  - `filter[facility_id]`, `filter[vendor_id]`, `filter[created_at]`, `filter[type]`, `filter[status]`
- Sort:
  - `sort=id,invoice_number,tax_invoice_number,amount,tax,total,paid,balance,invoice_date,invoice_uploaded_at,expense_posted_at,created_at,updated_at`
- Include:
  - `include=items`
- Pagination:
  - `per_page`, `page`

Enum filter options:

- `filter[type]`: `lpo`, `contract`, `liability`, `other` (from `FacilityBillData`)
- `filter[status]`: `pending`, `unpaid`, `partially-paid`, `paid`, `cancelled` (from `BillStatus` enum)

Sample list response (`FacilityBillResource`):

```json
{
  "data": [
    {
      "id": 1201,
      "invoice_number": "INV-1001",
      "tax_invoice_number": "TAX-1001",
      "invoice_date": {
        "raw": "2026-05-01T00:00:00.000000Z",
        "formatted": "01 May, 2026",
        "diff": "1 week ago"
      },
      "type": "lpo",
      "notes": "Quarterly maintenance",
      "tax_regime": "vat",
      "amount": "1000.00",
      "tax": "160.00",
      "total": "1160.00",
      "paid": "0.00",
      "balance": "1160.00",
      "vendor": { "id": 7, "name": "Acme Vendor" },
      "facility": { "id": 22, "name": "Riverside Plaza" },
      "status": { "value": "pending", "color": "warning" },
      "created": {
        "raw": "2026-05-09T10:10:10.000000Z",
        "formatted": "09 May, 2026",
        "diff": "1 minute ago"
      },
      "updated": {
        "raw": "2026-05-09T10:10:10.000000Z",
        "formatted": "09 May, 2026",
        "diff": "1 minute ago"
      }
    }
  ]
}
```

## Create payload (`BillData`)

| Field | Required | Type | Allowed Values / Notes |
|---|---|---|---|
| `vendor_id` | Yes | integer | Must exist in `users.id` |
| `facility_id` | Yes | integer | Must exist in `facilities.id` |
| `expense_type_id` | Yes | integer | Must exist in `expense_types.id` |
| `type` | No | string | `lpo`, `contract`, `liability`, `other` |
| `items` | No | array | Bill item objects |
| `currency_id` | No | integer | Must exist in `currencies.id` |
| `billable_type` | No | string | Optional |
| `billable_id` | No | integer | Optional |
| `notes` | No | string | Optional |
| `invoice_number` | No | string | Optional |
| `invoice_date` | No | date | Optional |
| `invoice_upload_id` | No | integer | Must exist in `uploads.id` |
| `tax_invoice_number` | No | string | Optional |

### Bill item object

| Field | Required | Type | Allowed Values / Notes |
|---|---|---|---|
| `tax_id` | No | integer | Must exist in `taxes.id` |
| `quantity` | No | integer | Defaults to `1` |
| `cost` | No | number | Defaults to `0` |
| `title` | No | string | Optional |
| `notes` | No | string | Optional |

## Upload/post payload (`UploadInvoiceBillData`)

Used by `post-bill`, `reject-invoice`, and some updates.

| Field | Required | Type |
|---|---|---|
| `invoice_number` | Yes | string |
| `invoice_date` | Yes | date |
| `tax_invoice_number` | Yes | string |
| `invoice_upload_id` | No | integer (`uploads.id`) |
| `notes` | No | string |
