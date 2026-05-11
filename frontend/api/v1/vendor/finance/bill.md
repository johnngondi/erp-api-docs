# Vendor Bills API

Domain: `Vendor Portal > Finance`

Base route:

`/api/v1/vendor/finance/bills`

## Endpoints

- `GET /bills`
- `GET /bills/{bill}`
- `POST /bills/{bill}/upload-invoice`

## List Bills

`GET /api/v1/vendor/finance/bills`

Supported query params:

- Filters:
  - `filter[search]` (Scout-backed search; supports CSV IDs and invoice numbers)
  - `filter[facility_id]`, `filter[vendor_id]`, `filter[created_at]`, `filter[type]`, `filter[status]`
- Sort:
  - `sort=id,invoice_number,tax_invoice_number,amount,tax,total,paid,balance,invoice_date,invoice_uploaded_at,expense_posted_at,created_at,updated_at`
- Include:
  - `include=items,withholdings`
- Pagination:
  - `per_page`, `page`

Enum filter options:

- `filter[type]`: `lpo`, `contract`, `liability`, `other` (from `FacilityBillData`)
- `filter[status]`: `pending`, `unpaid`, `partially-paid`, `paid`, `cancelled` (from `BillStatus` enum)

Sample list response (`Vendor\\FacilityBillResource`):

```json
{
  "data": [
    {
      "id": 2201,
      "notes": "Service contract billing",
      "type": "contract",
      "invoice_number": "V-INV-2201",
      "tax_invoice_number": "V-TAX-2201",
      "invoice_date": {
        "raw": "2026-05-01T00:00:00.000000Z",
        "formatted": "01 May, 2026",
        "diff": "1 week ago"
      },
      "amount": "5000.00",
      "tax": "800.00",
      "total": "5800.00",
      "paid": "1000.00",
      "balance": "4800.00",
      "currency": { "id": 1, "code": "KES" },
      "vendor": { "id": 7, "name": "Acme Vendor" },
      "facility": { "id": 22, "name": "Riverside Plaza" },
      "status": { "value": "unpaid", "color": "danger" },
      "created": {
        "raw": "2026-05-09T10:22:00.000000Z",
        "formatted": "09 May, 2026",
        "diff": "moments ago"
      }
    }
  ]
}
```

## Upload invoice payload

`POST /api/v1/vendor/finance/bills/{bill}/upload-invoice`

| Field | Required | Type |
|---|---|---|
| `invoice_number` | Yes | string |
| `invoice_date` | Yes | date |
| `tax_invoice_number` | Yes | string |
| `invoice_upload_id` | No | integer (`uploads.id`) |
| `notes` | No | string |
