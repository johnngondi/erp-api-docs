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
- `PUT /bills/bulk-post-bill`

## List Bills

`GET /api/v1/app/{company}/property-management/finance/bills`

Supported query params:

- Filters:
  - `filter[search]` (Scout-backed search; supports CSV IDs and invoice numbers)
  - `filter[facility_id]`, `filter[vendor_id]`, `filter[expense_type_id]`, `filter[expense_category_id]`, `filter[type]`, `filter[status]`
  - `filter[landlord_id]` — bills whose facility belongs to the given landlord (`users.id`)
  - `filter[contract_id]` — bills raised against a given contract (`facility_contracts.id`)
  - Date-range filters: `filter[invoice_date]`, `filter[created_at]`, `filter[invoice_uploaded_at]`, `filter[expense_posted_at]` — see [Date-range filtering](#date-range-filtering)
- Sort:
  - `sort=id,invoice_number,tax_invoice_number,amount,tax,total,paid,balance,invoice_date,invoice_uploaded_at,expense_posted_at,created_at,updated_at`
- Include:
  - `include=items`, `include=withholdings`, `include=withholdings.withholdingTax`
  - `include=creditNotes` — embeds the credit notes raised against each bill
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

Open-ended examples: `filter[invoice_date][from]=2026-05-01` (on/after) or
`filter[expense_posted_at][to]=2026-05-31` (on/before).

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
      "credit_notes_count": 1,
      "has_credit_notes": true,
      "credited_amount": 232,
      "creditable_amount": 928,
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

### Credit-note indicator fields

Every bill in the list/detail response carries a summary of the **active** (non-cancelled) credit
notes raised against it (see [Bill Credit Notes](./bill-credit-note.md)):

| Field | Type | Notes |
|---|---|---|
| `credit_notes_count` | integer | Number of active credit notes against this bill. |
| `has_credit_notes` | boolean | `true` when `credit_notes_count > 0`. Use this for a list badge/indicator. |
| `credited_amount` | number | Total value credited so far (absolute, positive). |
| `creditable_amount` | number | Value still creditable (`total − credited_amount`, floored at 0). Use as the credit note form's max. |

To also pull the credit notes themselves into the row, add `?include=creditNotes`. The bill **show**
endpoint always embeds them under `credit_notes` (each is a standard `FacilityBillResource` with
negative figures).

## Create payload (`BillData`)

| Field | Required | Type | Allowed Values / Notes |
|---|---|---|---|
| `vendor_id` | Yes | integer | Must exist in `users.id` |
| `facility_id` | Yes | integer | Must exist in `facilities.id` |
| `expense_type_id` | Yes | integer | Must exist in `expense_types.id` |
| `expense_category_id` | No | integer | Must exist in `expense_categories.id`; when omitted, backend derives from `expense_type_id` |
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
| `expense_category_id` | No | integer (`expense_categories.id`) |
| `notes` | No | string |

## Bulk post to expenses (`BulkUploadInvoiceFacilityBillData`)

`PUT /api/v1/app/{company}/property-management/finance/bills/bulk-post-bill`

Posts several **pending** bills against a single invoice in one transaction. Each bill is posted
to expenses exactly as the per-bill `post-bill` endpoint would (each gets its own `FacilityExpense`),
sharing the invoice fields below.

Constraints (the whole request is rejected with a `422` if any fails):

- Every id in `bill_ids` must exist and resolve to a bill the caller may update.
- All bills must belong to the **same vendor**.
- All bills must belong to facilities under the **same landlord** (facilities themselves may differ).
- Each bill must still be `pending` (enforced per bill when posting).

| Field | Required | Type | Notes |
|---|---|---|---|
| `bill_ids` | Yes | integer[] | Non-empty, distinct; each must exist in `facility_bills.id` |
| `invoice_number` | Yes | string | Applied to every bill |
| `invoice_date` | Yes | date | Applied to every bill |
| `tax_invoice_number` | Yes | string | Applied to every bill |
| `invoice_upload_id` | No | integer | Must exist in `uploads.id` |
| `expense_category_id` | No | integer | Must exist in `expense_categories.id` |
| `notes` | No | string | Optional |

Sample response:

```json
{
  "message": "3 bills posted to Expenses successfully.",
  "bills": [
    { "id": 1201, "...": "standard FacilityBillResource" }
  ]
}
```
