# Bill Credit Notes API

Domain: `Property Management > Finance`

Base route:

`/api/v1/app/{company}/property-management/finance/bills/credit-notes`

A **bill credit note** is a negative bill raised against an existing bill (e.g. for a vendor
overcharge, returned goods, or a discount agreed after invoicing). It is stored in the same
`facility_bills` table as a normal bill but with **negative figures** and `type = "credit-note"`,
and is automatically posted to the expense ledger as a **negative expense**. The credit note's
`billable` is the bill being credited.

Like a normal bill, a credit note starts as **`unpaid`** and becomes **`paid`** once it is included
in a payment voucher (its negative expense is selectable in the voucher flow alongside the vendor's
normal expenses, where it offsets the amount payable).

> **You do not send `type`.** The backend always sets `type = "credit-note"` on this endpoint, and
> derives `vendor_id`, `facility_id`, `currency_id`, and `expense_type_id` from the bill being
> credited. The front end only sends the fields in the payload tables below.

## Endpoints

- `POST   /bills/credit-notes` — create a credit note against a bill
- `PUT/PATCH /bills/credit-notes/{bill}` — update an un-vouchered credit note
- `DELETE /bills/credit-notes/{bill}` — delete (or cancel, if already vouchered)
- `PATCH  /bills/credit-notes/{bill}/cancel` — cancel a credit note (remove from flow)

> **Listing & viewing** use the standard [Bills](./bill.md) endpoints. To list only credit notes,
> filter the bills index by type:
> `GET /bills?filter[type]=credit-note`. A single credit note is fetched via `GET /bills/{bill}`.
> Both return the standard `FacilityBillResource` (see [Bills](./bill.md)).

## Telling whether a bill has credit notes

Each bill row in the [Bills](./bill.md) list/detail response includes a credit-note summary, so the
list can show an indicator without an extra call:

- `has_credit_notes` (boolean) — show a badge when `true`.
- `credit_notes_count` (integer) — how many active credit notes exist.
- `credited_amount` (number) — total credited so far (positive).

Add `?include=creditNotes` to embed the credit notes on each bill, or open the bill via
`GET /bills/{bill}` (its response always embeds them under `credit_notes`).

## When can a bill be credited?

The bill being credited (`bill_id`) must satisfy **both** rules below, or the request is rejected
with a `422` validation error:

1. **Status / remittance:**
   - `paid` or `partially-paid` → always creditable.
   - `unpaid` → creditable **only if the bill has been remitted** (its expense is part of a
     remittance).
   - `pending` or `cancelled` → **not** creditable.
2. **Amount cap:** the credit total (`amount + tax`) cannot exceed the bill's remaining balance,
   **net of any existing active credit notes** already raised against that bill.

The UI should only offer "Credit note" on bills meeting rule 1, and should validate the amount
against the remaining creditable balance before submitting.

## Prefilling the form from the bill

The credit note form should open **pre-filled with the credited bill's figures**, which all come
from the bill you already loaded (`GET /bills/{bill}`):

| Form field | Prefill from bill |
|---|---|
| `bill_id` | `bill.id` |
| `amount` | `bill.amount` (net) — but **clamped to `bill.creditable_amount`**, which is the most that can still be credited |
| `tax_type` | `bill.tax_type` |
| `tax_id` | `bill.items[0].tax_id` (relevant when `tax_type = rate`; load with `?include=items` or via the bill show endpoint) |
| `tax_amount` | `bill.tax` (when `tax_type = fixed`) |
| `notes` / invoice fields | left blank for the user, or seeded from the bill as you prefer |

`bill.creditable_amount` is the hard cap. When the bill has previous credit notes, prefill `amount`
with `min(bill.amount, bill.creditable_amount)` and cap the input at `creditable_amount`. The
backend re-validates this regardless.

## Create Credit Note

`POST /api/v1/app/{company}/property-management/finance/bills/credit-notes`

### Payload (`BillCreditNoteData`)

| Field | Required | Type | Allowed Values / Notes |
|---|---|---|---|
| `bill_id` | Yes | integer | The bill being credited. Must exist in `facility_bills.id` and be creditable (see rules above). |
| `amount` | Yes | number | Credit amount, **net of tax** (tax-exclusive). Send a **positive** number — the backend stores it as negative. |
| `tax_type` | No | string | `none` (default), `rate`, or `fixed`. How the credit tax is captured. |
| `tax_id` | Conditional | integer | Required when `tax_type = rate`. Must exist in `taxes.id`. |
| `tax_amount` | Conditional | number | Used when `tax_type = fixed`. The fixed tax value (positive). |
| `notes` | No | string | Reason for the credit note. |
| `invoice_number` | No | string | Credit note invoice/document number. |
| `invoice_date` | No | date | Credit note date. |
| `tax_invoice_number` | No | string | Credit note tax invoice number. |
| `invoice_upload_id` | No | integer | Must exist in `uploads.id`. Scanned credit note document. |

> The credit total (`amount + tax`) is validated against the bill's remaining creditable balance.
> Send positive values; the backend negates `amount`, `tax`, `total`, and `balance` on storage.

Sample request body:

```json
{
  "bill_id": 1201,
  "amount": 200.0,
  "tax_type": "rate",
  "tax_id": 5,
  "invoice_number": "CN-0001",
  "invoice_date": "2026-06-09",
  "tax_invoice_number": "CN-TAX-0001",
  "notes": "Overcharge on quarterly maintenance"
}
```

### Behaviour

On create, a single `credit-note` `FacilityBill` is recorded:

- **Type** is forced to `credit-note`; `vendor_id`, `facility_id`, `currency_id`, and
  `expense_type_id` are inherited from the bill being credited.
- **Billable** is the `FacilityBill` being credited (`bill_id`).
- **Amount / tax / total / balance** are stored as **negative** figures; `status` is `unpaid`.
- One negative **line item** is created.
- A negative **`FacilityExpense`** is posted automatically (status `unpaid`), so the credit note
  immediately reduces the vendor's outstanding payable and is available to add to a payment voucher.

Sample response (`FacilityBillResource`):

```json
{
  "data": {
    "message": "Bill credit note created successfully.",
    "bill": {
      "id": 1450,
      "type": "credit-note",
      "notes": "Overcharge on quarterly maintenance",
      "invoice_number": "CN-0001",
      "amount": "-200.00",
      "tax": "-32.00",
      "total": "-232.00",
      "paid": "0.00",
      "balance": "-232.00",
      "vendor": { "id": 7, "name": "Acme Vendor" },
      "facility": { "id": 22, "name": "Riverside Plaza" },
      "status": { "value": "unpaid", "color": "danger" }
    }
  }
}
```

## Update Credit Note

`PUT|PATCH /api/v1/app/{company}/property-management/finance/bills/credit-notes/{bill}`

### Payload (`UpdateBillCreditNoteData`)

| Field | Required | Type | Notes |
|---|---|---|---|
| `amount` | No | number | New credit amount (net of tax), positive. Only applied while the credit note is still `unpaid` (not yet vouchered). |
| `tax_type` | No | string | `none`, `rate`, or `fixed`. |
| `tax_id` | Conditional | integer | Required when `tax_type = rate`. |
| `tax_amount` | Conditional | number | Used when `tax_type = fixed`. |
| `notes` | No | string | Reason / description. |
| `invoice_number` | No | string | |
| `invoice_date` | No | date | |
| `tax_invoice_number` | No | string | |
| `invoice_upload_id` | No | integer (`uploads.id`) | |

### Behaviour

- Invoice details and `notes` can always be updated.
- `amount` / `tax` (and the negative line item and linked negative expense) are **only**
  recomputed while the credit note is still `unpaid`. Once it has been added to a payment voucher,
  amount changes are ignored — cancel and re-create instead.
- The amount cap rule is re-validated on update.

## Cancel Credit Note

`PATCH /api/v1/app/{company}/property-management/finance/bills/credit-notes/{bill}/cancel`

Cancels the credit note and **removes it from flow**: the credit note is set to `cancelled` and its
negative expense is reversed/cancelled so it no longer affects the vendor's payable or appears in
voucher options. Returns the updated `FacilityBillResource`.

## Delete Credit Note

`DELETE /api/v1/app/{company}/property-management/finance/bills/credit-notes/{bill}`

- If the credit note has **not** yet been added to a voucher (still `unpaid`), it is hard-deleted
  along with its line item and negative expense.
- Otherwise it is **cancelled** (same effect as the cancel endpoint).

Returns the `FacilityBillResource` of the affected credit note.
