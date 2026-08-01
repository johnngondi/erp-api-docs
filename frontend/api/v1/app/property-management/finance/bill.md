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
- `POST /bills/merge`

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
  - `include=mergedChildren` — embeds the bills merged into this one (see [Merge bills](#merge-bills))
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

### Merge indicator fields

| Field | Type | Notes |
|---|---|---|
| `merged_into_bill_id` | integer \| null | Set on a bill that was merged into a super bill. Merged bills are soft deleted, so they do not appear in the list at all — this only surfaces on a restored/`withTrashed` read. |
| `is_merged` | boolean | `true` when `merged_into_bill_id` is set. |
| `merged_bills_count` | integer | On a super bill, how many bills were merged into it (only present when the count is loaded). |
| `merged_bills` | array | The merged bills themselves, when `?include=mergedChildren` is used. |

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

## Merge bills

`POST /api/v1/app/{company}/property-management/finance/bills/merge`

Combines several **pending** bills from the same supplier for the same property into one new bill
(the "super bill"). Each merged bill becomes a single line item on the super bill — carrying its
amount, tax and notes — and is then soft deleted, so it disappears from the bill list.

Permission: the same as creating a bill (`create-facility-bill`).

Vendors can merge their own bills too, from
[`POST /api/v1/vendor/finance/bills/merge`](../../../vendor/finance/bill.md#merge-bills). That route
applies the same rules but is stricter: it accepts no classification overrides, so the selected bills
must already share the same `type`, `expense_type_id` and `expense_category_id`.

### Constraints

The whole request is rejected with a `422` (message under `errors.message`) if any of these fail:

- At least **two** distinct bills, all of which must exist.
- Every bill is `pending`. A bill in any other status cannot be merged.
- No bill has been invoiced or posted (`invoice_number`, `tax_invoice_number`, `expense_posted_at`
  must all be empty and no expense may exist).
- No bill is a `credit-note`, and none has already been merged into another bill.
- All bills share the same `vendor_id`, `facility_id` and `currency_id`.

### Choosing the effective type / category

`type`, `expense_type_id` and `expense_category_id` may legitimately differ between the selected
bills. When they do and no override is sent, the request fails with a message naming the field and
listing the candidate values, e.g.

```json
{
  "message": "The given data was invalid.",
  "errors": {
    "message": ["The selected bills have different expense type values (3, 7). Choose the one that applies to the merged bill."]
  }
}
```

The UI should then prompt the user to pick one and resend with the corresponding field set. An
override is always honoured, even when the bills agree.

### Payload (`MergeFacilityBillsData`)

| Field | Required | Type | Allowed Values / Notes |
|---|---|---|---|
| `bill_ids` | Yes | integer[] | At least 2, distinct; each must exist in `facility_bills.id` |
| `type` | No | string | `lpo`, `contract`, `liability`, `other`. Required only when the selected bills disagree |
| `expense_type_id` | No | integer | `facility_expense_types.id`. Required only when the selected bills disagree |
| `expense_category_id` | No | integer | `expense_categories.id`. Derived from the expense type when omitted |
| `expense_sub_type_id` | No | integer | `facility_expense_sub_types.id` |
| `notes` | No | string | Overrides the notes the backend composes (see below) |

### What the super bill looks like

- `amount`, `tax`, `total` = the sums of the merged bills; `paid = 0`, `balance = total`,
  `status = pending`, `tax_type = fixed` (the merged taxes are already resolved amounts).
- One item per merged bill: `title` from that bill's first item (falling back to `Bill#{id}`),
  `notes` = the merged bill's notes, `amount`/`tax`/`total` copied across, and `billable` pointing at
  the merged bill.
- `notes`: the client's `notes` if supplied; otherwise the shared note when every merged bill has the
  same one, else `Merged {n} bills: {note}; {note}; …`.
- `withholding_tax_ids` = the union of the merged bills' withholding taxes, recalculated against the
  combined totals.
- No invoice number: the super bill is invoiced and posted to expenses like any other bill.

### Undoing a merge

Cancelling the super bill (`PATCH /bills/{bill}/cancel`) **while it is still pending** restores every
merged bill exactly as it was — pending, with its own items and withholdings intact — and clears
their merge markers. Deleting a pending super bill does the same before removing it.

Once the super bill has left `pending` (posted to expenses, partially paid, paid), the merge is
locked in: cancelling it follows the normal bill-cancellation path (expense cancellation and, where
applicable, the negative reversal bill) and the merged bills stay deleted.

Sample response:

```json
{
  "message": "2 bills merged successfully.",
  "bill": {
    "id": 1310,
    "notes": "Merged 2 bills: Lobby cleaning; Car park cleaning",
    "amount": "1500.00",
    "tax": "240.00",
    "total": "1740.00",
    "status": { "value": "pending", "color": "warning" },
    "items": [
      { "id": 5001, "title": "Lobby cleaning", "notes": "Lobby cleaning", "amount": "1000.00", "tax": "160.00", "total": "1160.00" },
      { "id": 5002, "title": "Car park cleaning", "notes": "Car park cleaning", "amount": "500.00", "tax": "80.00", "total": "580.00" }
    ],
    "...": "standard FacilityBillResource"
  }
}
```
