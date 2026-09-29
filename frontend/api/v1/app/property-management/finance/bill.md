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
- `PATCH /bills/{bill}/backdate`: change the posting date. See [Change Bill Date (Backdate)](#change-bill-date-backdate)
- `PUT /bills/{bill}/post-bill`
- `PUT /bills/{bill}/reject-invoice`: reject the vendor's submitted invoice. See [Reject invoice](#reject-invoice)
- `PUT /bills/bulk-post-bill`
- `POST /bills/merge`
- `POST /bills/bulk-settlement` — see [Bulk bills payment](bulk-bills-payment.md)

## List Bills

`GET /api/v1/app/{company}/property-management/finance/bills`

Supported query params:

- Filters:
  - `filter[search]` (Scout-backed search; supports CSV IDs and invoice numbers)
  - `filter[facility_id]`, `filter[vendor_id]`, `filter[expense_type_id]`, `filter[expense_category_id]`, `filter[type]`, `filter[status]`

**Ids, `status` and `type` match exactly.** They were partial until 2026-09-29, so
`filter[status]=paid` also returned `unpaid` and `partially-paid`, and `filter[facility_id]=6`
returned rows for facility 60. Send the whole value. Date filters are still partial, so
`filter[created_at]=2026-09` means that month.

  - `filter[landlord_id]` — bills whose facility belongs to the given landlord (`users.id`)
  - `filter[contract_id]` — bills raised against a given contract (`facility_contracts.id`)
  - `filter[payable]` — boolean. When true, returns only bills that can still be paid: status
    `unpaid` or `partially-paid` **and** a non-zero `payable_amount`. This is what the "Pay Bills"
    table lists. Off by default, so the unfiltered index is unchanged.
  - `filter[invoice_validation_status]`, `filter[cu_validation_status]` — exact. `failed` gives
    the bills someone needs to look at
  - Date-range filters: `filter[invoice_date]`, `filter[created_at]`, `filter[invoice_uploaded_at]`, `filter[expense_posted_at]` — see [Date-range filtering](#date-range-filtering)

### Invoice and CU validation

A posted bill is checked against the invoice document attached to it - invoice number, invoice
date, amount, **tax** and total. It **records, it never blocks**: the bill is already posted by the
time the check runs, and a figure read off a scan is not authoritative enough to refuse a posting
over.

Four fields carry the result, each `{ "value", "color" }` for the status and free text for the
reason:

`invoice_validation_status`, `invoice_validation_failure_reason`,
`cu_validation_status`, `cu_validation_failure_reason`

| `value` | `color` | Means |
|---|---|---|
| `pending` | `warning` | Queued, the answer has not landed yet |
| `passed` | `success` | The document agrees with the bill |
| `failed` | `danger` | Something disagrees; the reason names each figure |
| `skipped` | `secondary` | No invoice document to read. Not a failure - nothing disagreed |

### The two checks answer different questions

They are separate on purpose, and a bill can pass one and fail the other:

- **`invoice_validation_*`** — does the document the supplier gave us match the bill we keyed in?
  Read out of the attached file.
- **`cu_validation_*`** — is the CU number on it one KRA has actually issued, for these amounts?
  Asked of KRA directly.

A forged document with real figures passes the first and fails the second. A genuine document keyed
in wrongly does the reverse.

**`cu_validation_status` now starts at `pending` rather than empty.** It was never written before,
so the column sat null and read as "fine"; every bill, invoice and credit note now begins at
`pending`, which says plainly that nobody has checked it yet. A bill stays `pending` — not
`failed` — for as long as KRA cannot be reached, so don't render `pending` as a problem.

The CU check reads the number from `tax_invoice_number` on a bill, which is where the supplier's
fiscal number is kept. Our own invoices and credit notes keep theirs in `cu_invoice_number`.

A document the extraction could read **nothing** from - an unrelated image, or a scan too poor to
read - comes back `failed`, not `passed`. Absence of one field is not a discrepancy, since plenty
of invoices carry no separate tax line; absence of every field means the document was never read,
and reporting that as a match would be worse than saying nothing.

`null` on all four means the bill has not been posted, or predates this. Neither needs attention.

**It is queued**, so posting does not wait on it and the ledger does not depend on the document
pipeline being reachable. That also means **a queue worker has to be running** - without one the
status sits on `pending` and the feature looks stuck rather than working.

**Correcting a posted bill re-runs it.** Changing the invoice number, invoice date, CU number or
the attached document puts the status back to `pending`, clears the old reason and queues a fresh
check - otherwise a bill would keep reporting a discrepancy that had already been fixed. A posted
bill only accepts invoice details, so its amount cannot be corrected at that point. An edit that
touches nothing the check looks at leaves the verdict alone.

**An hourly sweep picks up anything left behind.** A posted bill whose `invoice_validation_status`
is `pending` or `skipped` and that now has a document attached, or whose `cu_validation_status` is
`pending` or `skipped` and that now has a `tax_invoice_number`, is queued for that check again
every hour. A document or CU number added after posting, or a check lost to a queue or KRA outage,
still gets a verdict without anyone re-saving the bill. Unposted bills are left alone, because
posting runs both checks.

**The CU check is a shell.** It needs a KRA lookup that is not built, so `cu_validation_status`
stays `null`. A bill's CU number comes from the supplier's own fiscal device, so unlike an invoice
we sign, there is no verification URL to link to yet.

**Suppliers do not see any of this.** The vendor bill endpoints use their own resource, which
carries none of these fields - it is an internal control, not something to put in front of the
party being checked.

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
      "withheld_amount": 60,
      "payable_amount": 1100,
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

### Payable amount fields

The list and show endpoints always eager-load `withholdings`, so every bill carries what is still
owed to the supplier:

| Field | Type | Notes |
|---|---|---|
| `withheld_amount` | number | Tax still withheld on the bill — the sum of withholdings with a positive `balance`. A withholding that has already been settled via a liability bill has also reduced `balance`, so it is not counted again here. |
| `payable_amount` | number | `balance − withheld_amount`. What the supplier can actually be paid, and what the bulk payment popup prefills its "To Pay" input with. Negative on a credit note. |

Both are omitted when `withholdings` is not loaded (e.g. a resource rendered from a create/update
response).

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
| `post_direct` | No | boolean | Post the bill straight to the expense ledger at creation |
| `invoice_number` | Conditional | string | **Required when `post_direct` is true** |
| `invoice_date` | Conditional | date | **Required when `post_direct` is true** |
| `invoice_upload_id` | Conditional | integer | Must exist in `uploads.id`. **Required when `post_direct` is true** |
| `tax_invoice_number` | Conditional | string | **Required when `post_direct` is true** |

### The invoice date cannot be in the future

An invoice dated tomorrow has not been issued yet, and posting it puts the expense in a tax period
that has not happened. Refused on all four paths — create with `post_direct`, `post-bill`,
`bulk-post-bill` and editing a bill — with `422` on `invoice_date`:

```json
{ "errors": { "invoice_date": ["The invoice date cannot be in the future."] } }
```

Setting the picker's maximum to today will keep users out of it.

**One carve-out worth knowing.** A bill that *already* carries a future date — some were created
before this rule — can still be saved as long as the date is unchanged. Without that, the edit form
resends the stored date, the rule refuses it, and edits that have nothing to do with the date become
impossible. Moving such a bill to a *different* future date is still refused; correcting it to today
or earlier works.

### The invoice document is required to post

Nothing reaches the expense ledger without the supplier's invoice document. That applies to every
path a person can post through:

| Endpoint | Rule |
|---|---|
| `POST .../bills` with `post_direct: true` | all four invoice fields required |
| `PUT .../bills/{bill}/post-bill` | `invoice_upload_id` required, unless the bill already carries one |
| `PUT .../bills/bulk-post-bill` | `invoice_upload_id` required on the shared payload |
| `POST /api/v1/vendor/finance/bills/{bill}/upload-invoice` | `invoice_upload_id` required |
| `PUT .../bills/{bill}` on an **already posted** bill | the document may be replaced, not removed |

All of them return `422` keyed `invoice_upload_id`:

```json
{ "errors": { "invoice_upload_id": ["A bill cannot be posted to expenses without its invoice document."] } }
```

and the posted-bill edit returns

```json
{ "errors": { "invoice_upload_id": ["The invoice document cannot be removed. Replace it with another instead."] } }
```

**Saving a pending bill is unchanged** — the document stays optional right up until the bill posts,
and a pending bill's document can still be removed as well as replaced.

> Note: `post_direct` is the trigger on create, not the presence of an invoice number. Sending
> invoice fields without `post_direct: true` raises an ordinary pending bill and requires nothing.
> Previously the three non-document invoice fields were only enforced when sent as an explicit
> `null`, so omitting them posted a bill with no invoice details at all; they are now enforced
> either way.

**Bills raised before this rule stay editable.** A posted bill that never had a document can still
be saved without one — that is how a missing document gets attached. Only taking away a document
that is there is refused.

**System-raised bills are not affected.** Utility billing, LPO close, tenant exit notices,
remittances and the data migration raise (and sometimes post) bills with no supplier invoice to
give, and they keep working.

### Bill item object

| Field | Required | Type | Allowed Values / Notes |
|---|---|---|---|
| `tax_id` | No | integer | Must exist in `taxes.id` |
| `quantity` | No | integer | Defaults to `1` |
| `cost` | No | number | Defaults to `0` |
| `title` | No | string | Optional |
| `notes` | No | string | Optional |

## Upload/post payload (`UploadInvoiceBillData`)

Used by `post-bill` and some updates. It is **not** used by `reject-invoice`; see [Reject invoice](#reject-invoice).

| Field | Required | Type |
|---|---|---|
| `invoice_number` | Yes | string |
| `invoice_date` | Yes | date |
| `tax_invoice_number` | Yes | string |
| `invoice_upload_id` | Conditional | integer (`uploads.id`) — required when posting, unless the bill already carries one; on a posted bill it may be replaced but not set to `null` |
| `expense_category_id` | No | integer (`expense_categories.id`) |
| `notes` | No | string |

Editing `invoice_date` on a posted bill changes **only** the invoice date. It no longer moves
`expense_posted_at` or the expense's dates. The invoice date records when the invoice was
generated. To move the posting date, use [Change Bill Date](#change-bill-date-backdate).

## Reject invoice

`PUT /api/v1/app/{company}/property-management/finance/bills/{bill}/reject-invoice`

This rejects the invoice a vendor submitted against a **pending** bill. It clears the invoice so the
vendor can submit it again. It never posts the bill.

Show a **Reject Invoice** item in the bills list row actions and in the view page header actions,
but only when `bill.permissions.validate === true`. The flag needs the `validate-bill-invoice`
permission and is `false` for any bill that is not `pending`. The reject flow is separate from Post
Bill and uses its own modal.

Request body:

| Field | Required | Type | Notes |
|---|---|---|---|
| `reason` | Yes | string (max 1000) | Sent to the vendor and kept on the bill. |

```json
{ "reason": "The CU number does not match the attached invoice." }
```

What happens:

- These fields are set to `null`: `invoice_number`, `invoice_date`, `tax_invoice_number` (the CU
  invoice number), `invoice_upload_id`, `invoice_uploaded_at`, `invoice_validation_status`,
  `invoice_validation_failure_reason`, `cu_validation_status` and `cu_validation_failure_reason`.
- The uploaded invoice file is deleted.
- `invoice_rejected_at`, `invoice_rejected_by` and `invoice_rejection_reason` record the rejection.
- The bill stays `pending`. No expense or vendor statement entry is written.
- The vendor is notified in the app, by email, by SMS and by WhatsApp. The notification carries
  the reason and links to the bill.

New bill resource fields:

| Field | Type | Notes |
|---|---|---|
| `invoice_rejected_at` | `{raw, formatted, diff}` or `null` | When the invoice was last rejected. |
| `invoice_rejection_reason` | string or `null` | Why it was rejected. |
| `invoice_rejected_by` | `{id, name}` | Present when loaded (it is loaded on this endpoint's response). |

These fields keep the last rejection even after the vendor submits again. Show them in the Invoice
Details card whenever `invoice_rejected_at` is set.

Success (`200`) returns `{ data: { message, bill } }`.

Errors:

- `422` `errors.reason`: the reason is missing or longer than 1000 characters. Show it under the
  reason field.
- `422` `errors.bill`: the bill has no invoice to reject, or it stopped being pending after the
  screen loaded. Show it as a toast and close the modal.
- `403`: the user lacks `validate-bill-invoice`, or the bill is not pending.

## Change Bill Date (Backdate)

`PATCH /api/v1/app/{company}/property-management/finance/bills/{bill}/backdate`

This moves a posted bill to a different posting date. Show a **Change Date** item in the bills
list row actions and in the view page header actions, but only when
`bill.permissions.backdate === true`. That flag is already `false` for bills that are pending,
cancelled, never posted, or whose expense has been remitted. Credit notes are included.

Request body:

| Field | Required | Type | Notes |
|---|---|---|---|
| `new_bill_posted_date` | Yes | string (`YYYY-MM-DD`) | Today or earlier. It may fall before `invoice_date`. Cap the date picker at today. |

```json
{ "new_bill_posted_date": "2026-08-15" }
```

Behaviour:

- `expense_posted_at` and the expense's `transaction_at` and `created_at` move to the new date.
  The original time of day is kept.
- `invoice_date` and `posted_at` never change.
- If a non-advance landlord remittance for the property covers the chosen month, the request is
  refused. The user has to pick a date in a different month.

Success (`200`) returns `{ data: { message, bill } }`. Upsert the bill with it.

Errors:

- `422` `errors.new_bill_posted_date`: the date is missing, invalid, in the future, or in a month
  that has already been remitted. Show the message under the date picker.
- `422` `errors.bill`: the bill's expense was remitted after the screen loaded. Show it as a toast
  and close the modal.
- `403`: the user lacks `backdate-facility-bill`, or the bill is no longer eligible.

Backend reference: `docs/backend/api/v1/app/bill-backdate.md`.

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
| `invoice_upload_id` | Yes | integer | Must exist in `uploads.id`. One document covers the whole batch |
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
