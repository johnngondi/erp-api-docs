# Liability Bills API

Domain: `Property Management > Finance`

Base route:

`/api/v1/app/{company}/property-management/finance/bills/liabilities`

Liability bills group the outstanding withholding amounts (`FacilityBillWithholding`) of bills so a vendor's withholding tax can be settled and remitted together. The list returns individual withholding rows with their parent bill loaded; the create endpoint accepts the set of withholding rows being paid under a single withholding tax.

## Endpoints

- `GET /bills/liabilities` — list withholdings of payable bills
- `POST /bills/liabilities` — create a liability bill for selected withholdings

## List Liability Withholdings

`GET /api/v1/app/{company}/property-management/finance/bills/liabilities`

Returns every `FacilityBillWithholding` whose parent bill is **not** `pending` or `cancelled`. Each row includes its withholding tax and the parent bill (with `vendor`, `facility`, `expense_type`, and `notes`).

> This endpoint is **not paginated** — the full filtered set is returned in `data`.

Supported query params:

- Filters (the first four resolve against the parent bill):
  - `filter[facility_id]`, `filter[expense_type_id]`, `filter[expense_category_id]`
  - `filter[withholding_tax_id]`
  - `filter[expense_posted_at]` — **date range** over the parent bill's `expense_posted_at`; filters withholdings whose bill was posted within the period. Accepts either form:
    - **CSV** (`from,to`): `filter[expense_posted_at]=2026-05-01,2026-05-31`
    - **Keyed**: `filter[expense_posted_at][from]=2026-05-01&filter[expense_posted_at][to]=2026-05-31`

    Either bound may be omitted for an open-ended range — e.g. CSV `filter[expense_posted_at]=2026-05-01` (from only) or `filter[expense_posted_at]=,2026-05-31` (to only).
- Sort:
  - `sort=id,bill_id,amount,paid,balance,remitted,created_at,updated_at`

Sample list response (`LiabilityBillWithholdingResource`):

```json
{
  "data": [
    {
      "id": 5012,
      "bill_id": 1201,
      "withholding_tax_id": 3,
      "amount": "160.00",
      "paid": "0.00",
      "balance": "160.00",
      "remitted": "0.00",
      "withholding_tax": { "id": 3, "name": "VAT Withholding", "value": "2.00" },
      "bill": {
        "id": 1201,
        "invoice_number": "INV-1001",
        "tax_invoice_number": "TAX-1001",
        "expense_posted_at": {
          "raw": "2026-05-09T00:00:00.000000Z",
          "formatted": "09 May, 2026",
          "diff": "1 week ago"
        },
        "amount": "1000.00",
        "tax": "160.00",
        "total": "1160.00",
        "notes": "Quarterly maintenance",
        "vendor": { "id": 7, "name": "Acme Vendor" },
        "facility": { "id": 22, "name": "Riverside Plaza" },
        "expense_type": { "id": 4, "name": "Maintenance" },
        "expense_category": { "id": 4, "name": "Rent" },
        "status": { "value": "unpaid", "color": "danger" }
      }
    }
  ]
}
```

`id` is the `FacilityBillWithholding` row id — pass it back as `withholdings[].id` when creating a liability bill to target an existing withholding for update/cancellation.

## Create Liability Bill

`POST /api/v1/app/{company}/property-management/finance/bills/liabilities`

### Payload (`LiabilityBillData`)

| Field | Required | Type | Allowed Values / Notes |
|---|---|---|---|
| `vendor_id` | Yes | integer | Must exist in `users.id` |
| `facility_id` | Yes | integer | Must exist in `facilities.id` |
| `withholding_tax_id` | Yes | integer | Must exist in `withholding_taxes.id` |
| `withholdings` | Yes | array | Non-empty; the withholding rows being paid |

### Withholding object (`LiabilityBillWithholdingData`)

| Field | Required | Type | Allowed Values / Notes |
|---|---|---|---|
| `bill_id` | Yes | integer | Must exist in `facility_bills.id` |
| `id` | Yes | integer | Existing `facility_bill_withholdings.id` being settled |
| `amount` | No | number | The withholding's outstanding amount; defaults to `0` |
| `to_pay` | No | number | Amount being paid now toward this withholding; defaults to `0` |

Sample request body:

```json
{
  "vendor_id": 7,
  "facility_id": 22,
  "withholding_tax_id": 3,
  "withholdings": [
    { "id": 5012, "bill_id": 1201, "amount": 160.0, "to_pay": 160.0 },
    { "id": 5013, "bill_id": 1202, "amount": 80.0, "to_pay": 80.0 }
  ]
}
```

### Behaviour

On create, a single `liability` `FacilityBill` is recorded:

- **Total** = `sum(withholdings[].to_pay)`; it carries no tax and no withholdings of its own.
- **Billable** is the `WithholdingTax` (`withholding_tax_id`) being settled.
- One **line item** is created per selected withholding, each linked (`billable`) back to its
  `FacilityBillWithholding`.
- Each selected `FacilityBillWithholding` has its `paid`/`balance` advanced by `to_pay` and its
  status set to `paid` (balance cleared) or `partially-paid`.

Cancelling the liability bill (via the standard bill cancel endpoint) reverses every settlement:
the linked withholdings have their `paid`/`balance` unwound and their status re-derived.

Sample response (`FacilityBillResource`):

```json
{
  "data": {
    "message": "Liability bill created successfully.",
    "bill": {
      "id": 1300,
      "type": "liability",
      "amount": "240.00",
      "tax": "0.00",
      "total": "240.00",
      "paid": "0.00",
      "balance": "240.00",
      "vendor": { "id": 7, "name": "Acme Vendor" },
      "facility": { "id": 22, "name": "Riverside Plaza" },
      "status": { "value": "unpaid", "color": "danger" }
    }
  }
}
```
