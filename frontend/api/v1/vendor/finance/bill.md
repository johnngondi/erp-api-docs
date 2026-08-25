# Vendor Bills API

Domain: `Vendor Portal > Finance`

Base route:

`/api/v1/vendor/finance/bills`

## Endpoints

- `GET /bills`
- `GET /bills/{bill}`
- `POST /bills/{bill}/upload-invoice`
- `POST /bills/merge`

## List Bills

`GET /api/v1/vendor/finance/bills`

Supported query params:

- Filters:
  - `filter[search]` (Scout-backed search; supports CSV IDs and invoice numbers)
  - `filter[facility_id]`, `filter[vendor_id]`, `filter[expense_category_id]`, `filter[type]`, `filter[status]`
  - `filter[contract_id]` — bills raised against a given contract (`facility_contracts.id`)
  - Date-range filters: `filter[invoice_date]`, `filter[created_at]`, `filter[invoice_uploaded_at]`, `filter[expense_posted_at]` — see [Date-range filtering](#date-range-filtering)
- Sort:
  - `sort=id,invoice_number,tax_invoice_number,amount,tax,total,paid,balance,invoice_date,invoice_uploaded_at,expense_posted_at,created_at,updated_at`
- Include:
  - `include=items,withholdings`
- Pagination:
  - `per_page`, `page`

Enum filter options:

- `filter[type]`: `lpo`, `contract`, `liability`, `other` (from `FacilityBillData`)
- `filter[status]`: `pending`, `unpaid`, `partially-paid`, `paid`, `cancelled` (from `BillStatus` enum)

### Date-range filtering

`invoice_date`, `created_at`, `invoice_uploaded_at`, and `expense_posted_at` are range filters.
Each accepts an inclusive `from`/`to` pair (dates, `Y-m-d`); either bound may be omitted for an
open-ended range. Both shapes are supported:

- Bracket form: `filter[created_at][from]=2026-05-01&filter[created_at][to]=2026-05-31`
- CSV form: `filter[created_at]=2026-05-01,2026-05-31`

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
| `expense_category_id` | No | integer (`expense_categories.id`) |
| `notes` | No | string |

## Merge bills

`POST /api/v1/vendor/finance/bills/merge`

Combines several of **your own** pending bills for the same property into one bill (the "super
bill"), so a single invoice can be raised against it. Each merged bill becomes one line item on the
super bill — carrying its amount, tax and notes — and then disappears from your bill list.

Access is the same as uploading an invoice on a bill: any vendor user who may upload an invoice on
the selected bills may merge them. Every selected bill must belong to your own vendor account
(`403` otherwise).

### Constraints

Rejected with a `422` (message under `errors.message`) unless:

- At least **two** distinct bills are selected and all of them exist.
- Every bill is `pending`. Bills that are unpaid, partly paid, paid or cancelled cannot be merged.
- No bill carries an invoice number, tax invoice number or has been posted to expenses.
- No bill is a credit note, and none has already been merged into another bill.
- All bills share the same property (`facility_id`) and currency.
- **All bills already share the same expense category, expense type and bill type.** Unlike the
  landlord's portal, the vendor portal cannot change how an expense is classified, so mismatched
  bills must be merged by the landlord instead. The message names the field, e.g.
  `All bills must have the same expense category to be merged.`

### Payload

| Field | Required | Type | Notes |
|---|---|---|---|
| `bill_ids` | Yes | integer[] | At least 2, distinct; each must exist in `facility_bills.id` and belong to you |
| `notes` | No | string | Overrides the notes composed from the merged bills |

`type`, `expense_type_id` and `expense_category_id` are accepted by the request but **ignored** on
this route — the merged bills must already agree on them.

### What the merged bill looks like

- `amount`, `tax`, `total` = the sums of the merged bills; `paid = 0`, `balance = total`,
  `status = pending`, no invoice number yet — upload the invoice on the merged bill as usual.
- One item per merged bill, keeping that bill's title, notes and figures.
- `notes`: your `notes` if supplied; otherwise the shared note when every merged bill has the same
  one, else `Merged {n} bills: {note}; {note}; …`.

Merging can be undone by the landlord: cancelling the merged bill while it is still pending restores
the original bills exactly as they were.

Sample response:

```json
{
  "message": "2 bills merged successfully.",
  "bill": {
    "id": 2310,
    "notes": "Merged 2 bills: Lobby cleaning; Car park cleaning",
    "type": "lpo",
    "amount": "1500.00",
    "tax": "240.00",
    "total": "1740.00",
    "status": { "value": "pending", "color": "warning" },
    "items": [
      { "id": 6001, "title": "Lobby cleaning", "notes": "Lobby cleaning", "amount": "1000.00", "tax": "160.00", "total": "1160.00" },
      { "id": 6002, "title": "Car park cleaning", "notes": "Car park cleaning", "amount": "500.00", "tax": "80.00", "total": "580.00" }
    ],
    "...": "standard Vendor\\FacilityBillResource"
  }
}
```
