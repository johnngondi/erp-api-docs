# Credit Notes API

Domain: `Property Management > Lease Management`

Base route:

`/api/v1/app/{company}/property-management/lease-management/credit-notes`

## Endpoints

- `GET /credit-notes`
- `POST /credit-notes`
- `GET /credit-notes/{creditNote}`
- `PUT/PATCH /credit-notes/{creditNote}`
- `PATCH /credit-notes/{creditNote}/cancel`
- `POST /credit-notes/{creditNote}/sign`
- `POST /credit-notes/{creditNote}/dispute`
- `DELETE /credit-notes/{creditNote}/dispute`
- `DELETE /credit-notes/{creditNote}`

## List Credit Notes

`GET /api/v1/app/{company}/property-management/lease-management/credit-notes`

Supported query params:

- Filters:
  - `filter[search]` — free text
  - **Exact** (`=`): `filter[lease_id]`, `filter[status]`, `filter[facility_id]` (through the lease)
  - **Partial** (`LIKE`): `filter[due_at]`, `filter[created_at]`
  - `filter[reversed]=true|false`: only full reversals, or only everything else
- Sort:
  - `sort=id,due_at,created_at,amount,tax,total,paid,balance`
- Include: not supported
> **Exact versus partial matters here.** `lease_id` and `status` are exact. They were declared as
> bare strings, which Spatie turns into a *partial* match — so `filter[lease_id]=6` returned leases
> 6, 16, 26, 60-69 and anything else whose id contains a 6, putting another tenant's billing on a
> lease's page. `filter[status]=applied` also matched `unapplied` and `partially applied`.
>
> Dates stay partial on purpose, so `filter[created_at]=2026-09` means "that month".

- Select fields: not supported
- Pagination: `per_page`, `page`

Example:

`GET /api/v1/app/12/property-management/lease-management/credit-notes?filter[lease_id]=101&sort=-due_at`

## CU number validation

Every invoice, credit note and bill carries a verdict on whether its CU number is one KRA actually
has. Two fields, the same shape as the bill's invoice-document check:

| Field | Type |
|---|---|
| `cu_validation_status` | `{ "value": "pending", "color": "warning" }` |
| `cu_validation_failure_reason` | string \| null — set only on `failed` |

| Status | Colour | Means |
|---|---|---|
| `pending` | `warning` | Not checked yet. **The default on every document.** |
| `passed` | `success` | KRA knows the number and agrees with our figures |
| `failed` | `danger` | KRA does not know it, or the figures disagree — `cu_validation_failure_reason` says which |
| `skipped` | `secondary` | No CU number on the document to check |

Filter the listings with `filter[cu_validation_status]=failed` to find the ones worth a look.

### When it runs

On its own, after the fact. A document is signed to the fiscal device and issued first; the check
is queued and the verdict lands afterwards. **Nothing is ever blocked by it** — KRA's checker is a
portal we do not control, and a reading from it is not authoritative enough to refuse a document
over. Treat `failed` as "someone should look", not "this is wrong".

That also means a document sits at `pending` until the queue gets to it, and stays `pending` — not
`failed` — for as long as KRA cannot be reached. Do not read `pending` as a problem.

**An hourly sweep picks up anything left behind.** A document on `pending` or `skipped` that
now carries a CU number is queued for the check again every hour, so a number added after the
fact, or a check lost to a queue outage or a long KRA outage, still gets a verdict without anyone
asking. Nothing is needed from the frontend; the status simply moves on its own.

### What `failed` can say

```json
{
  "cu_validation_status": { "value": "failed", "color": "danger" },
  "cu_validation_failure_reason": "Total on the document is 2234.93 but KRA has 2500.00."
}
```

Three kinds of reason, all rendered as plain sentences you can show directly:

- **KRA does not recognise the number** — *"Please enter valid Middleware Invoice Number"*, KRA's
  own wording.
- **The figures disagree** — total, amount or tax, named and quoted both ways. Differences of a
  single cent are forgiven, since KRA's figures are read back off a fiscal receipt.
- **Wrong kind of document** — *"KRA has this number registered as a credit note, not an
  invoice."* KRA records each number as a `Tax Invoice` or a `Credit Note`, so a credit note
  carrying an invoice's number is caught here and nowhere else.

## Create Credit Note

`POST /api/v1/app/{company}/property-management/lease-management/credit-notes`

Request body:

| Field | Required | Type | Allowed Values / Notes |
|---|---|---|---|
| `lease_id` | Yes | integer | - |
| `due_at` | Yes | date (`YYYY-MM-DD`) | - |
| `notes` | Yes | string | - |
| `items` | Yes | array | At least one item |
| `invoice_id` | No | integer | The invoice this credit note reverses. See [One invoice per credit note](#one-invoice-per-credit-note) |
| `cu_reference_number` | No | string | Optional. Not used for signing: the ETR reference always comes from the linked invoice |
| `is_credit` | No | boolean | Defaults to `true` |
| `paid` | No | number | Defaults to `0` |
| `cu_invoice_number` | No | string | Optional |
| `cu_serial_number` | No | string | Optional |
| `cu_credit_note_verify_url` | No | string | Optional |

`items[]` fields:

| Field | Required | Type | Allowed Values / Notes |
|---|---|---|---|
| `lease_item_component_id` | Yes | integer | Must belong to supplied lease |
| `notes` | Yes | string | - |
| `quantity` | Yes | integer | Minimum `1` |
| `amount` | Yes | number | Minimum `0` |
| `tax_id` | Yes | integer | Must exist in `taxes.id` |

Notes:

- `lease_item_component_id` must be unique within the same request.
- Response exposes `cu_invoice_verify_url` in resource output.

Example request:

```json
{
  "lease_id": 101,
  "due_at": "2026-03-01",
  "notes": "Overcharge adjustment",
  "items": [
    {
      "lease_item_component_id": 455,
      "notes": "Base rent correction",
      "quantity": 1,
      "amount": 120,
      "tax_id": 1
    }
  ]
}
```

Example response:

```json
{
  "message": "Credit note created successfully.",
  "credit_note": {
    "id": 410,
    "total": 139.2,
    "status": { "value": "pending", "color": "secondary" }
  }
}
```

### Validation (`422`) on `invoice_id`

| Condition | Message |
|---|---|
| Invoice does not exist | Invoice not found |
| Invoice has no balance left | Invoice is already paid for fully |
| Invoice is cancelled / rejected | Invoice is already cancelled / Invoice was rejected |
| Invoice is on a receipt that is awaiting approval | Invoice #9001 is already being paid in receipt #2500 (RCPT-1004), which is awaiting approval. |
| Items total more than the invoice balance | Credit note total exceeds invoice balance |

The pending-receipt rule is the lock described under
[Invoices → already being paid](./invoices.md#invoices-already-being-paid-by-a-pending-receipt).
The invoice list still returns such an invoice (with `pending_receipt` set and
`permissions.issueCreditNote: false`), so the picker should show it as unavailable rather than
let the request fail. It applies to credit notes raised through this endpoint; credits the system
raises itself (invoice reversal, opening balances, exit notices) are not held up by it.

## One invoice per credit note

Every credit note reverses exactly one invoice, and `invoice_id` is the only record of which one.

- **Raised against an invoice** (`invoice_id` sent): it is applied to that invoice when it is
  processed. Its total cannot exceed the invoice's balance.
- **Raised open** (no `invoice_id`, e.g. an overpayment brought forward or a goodwill credit): it
  comes to rest `unapplied`. When the next invoice is raised on the lease, the credit note is
  applied to it automatically and **that invoice becomes its `invoice_id`**. If the credit is
  larger than the invoice, the rest stays `partially applied` and is drawn down by later invoices,
  but `invoice_id` keeps pointing at the first one.
- **Signing needs the invoice.** An open credit note cannot be signed on the ETR: the device needs
  the CU number of the invoice it reverses. See [Sign Credit Note](#sign-credit-note-etr).

### Signs

A credit note's `amount`, `tax`, `total`, `paid` and `balance` are all stored and returned
**negative** (or zero). `balance` is the credit still available to the tenant: an `unapplied`
credit note of 8,000 has `paid: 0, balance: -8000`; after 3,000 of it is drawn down it has
`paid: -3000, balance: -5000`. Show the absolute value where the screen reads "available credit".

### Full reversal or partial credit

A credit note is a **reversal** only when it covers the invoice's full total. A partial credit
still carries `invoice_id`, but it is not a reversal.

| | Full reversal | Partial credit |
|---|---|---|
| Invoice `reversal_credit_note_id` | this credit note's id | unchanged |
| Credit note `reversed_invoice_id` | the invoice id | `null` |
| `filter[reversed]=true` (on either list) | included | excluded |

A reversed invoice's balance drops to zero, so its status reads `paid` even though nothing was
paid. Use `reversal_credit_note_id` / `filter[reversed]` to tell the two apart (see the invoices
doc).

## Update Credit Note

`PUT/PATCH /api/v1/app/{company}/property-management/lease-management/credit-notes/{creditNote}`

Use the same payload shape as create.

## Cancel Credit Note

`PATCH /api/v1/app/{company}/property-management/lease-management/credit-notes/{creditNote}/cancel`

No request body required.

What happens depends on whether the credit note has been signed on the ETR (`etr_signed_at` /
`cu_invoice_number` is set).

**Not signed.** The credit note is withdrawn:

- its lease billing entries are removed
- the amount goes back onto the invoice's balance. If this credit note was the invoice's reversal,
  `reversal_credit_note_id` is cleared and the invoice is open again
- the tenant statement gets a `Cancelled Credit Note # {id} - {notes}` debit line

A queued signing job for the credit note does nothing once it has been cancelled.

**Signed.** KRA already holds the credit note, so it cannot be withdrawn. The credit is **re-billed
with a new invoice** instead:

- a new invoice is raised for **the same amount as the credit note**. It copies the credit note's
  lines (component, quantity, cost, tax), is posted straight away without approval, and is queued
  for ETR signing
- the new invoice's `reissued_from_credit_note_id` is set to this credit note's id. Its notes, and
  its tenant statement line, read
  `INV#{id} - Re-issued for cancelled ETR signed CN#{creditNoteId} - {credit note notes}`
- the credit note's own billing entries and statement credit stay, and so does the original
  invoice's `reversal_credit_note_id`. On the statement, the tenant sees the credit and then the
  new invoice's debit
- the credit note still counts as a negative line in Output VAT, because KRA has it
- the response's credit note carries `reissued_invoice_id`

Possible status values returned by API resource:

- `pending` (`secondary`)
- `unapplied` (`info`)
- `partially applied` (`warning`)
- `applied` (`success`)
- `cancelled` (`danger`)
- `rejected` (`danger`) — the approval chain was rejected. The credit note was never applied, so
  it touches no invoice, statement or VAT report. It can only be viewed or deleted (delete removes
  it outright, with no reversal); every other `permissions` flag is `false` and the endpoints
  refuse it

`cancelled` is only ever set by this endpoint now; a rejection never produces it.

## Sign Credit Note (ETR)

`POST /api/v1/app/{company}/property-management/lease-management/credit-notes/{creditNote}/sign`

Signs the credit note on the property's ETR (KRA fiscal) device and stores the CU details on the credit note. No request body.

The device comes from the property's `esd_type` / `esd_config` (see Facilities). A property with no device set uses the system default (Incotex). The seller PIN sent to the device is always the **landlord's KRA PIN**, and the buyer PIN is the tenant's.

When to show the action: `permissions.sign` on the credit note resource is `true`. It is `false` when:

- the user lacks the `sign-facility-credit-note` permission
- the credit note is `pending` (awaiting approval), `cancelled` or `rejected`
- the credit note is already signed (`cu_invoice_number` is set)
- it has no invoice yet (an open credit note), or its invoice is not signed yet (sign the invoice first). A typed `cu_reference_number` does not stand in for the invoice

Signing runs as a queued job. On a synchronous queue (the default locally) the request returns after the device answers; on a real queue it returns straight away and the CU fields fill in once the job runs. Refetch the credit note (or poll `GET /credit-notes/{creditNote}`) until `etr_signed_at` or `etr_error` is set.

Success response (`200`):

```json
{
  "message": "Credit note signed successfully",
  "credit_note": {
    "id": 9001,
    "cu_invoice_number": "0010000042",
    "cu_serial_number": "KRAMW017202209000001",
    "cu_credit_note_verify_url": "https://itax.kra.go.ke/KRA-Portal/invoiceChk.htm?actionCode=loadPage&invoiceNo=0010000042",
    "etr_signed_at": {
      "raw": "2026-09-22T10:15:00.000000Z",
      "formatted": "22 Sep, 2026 10:15",
      "diff": "1 second ago"
    },
    "etr_error": null,
    "permissions": { "sign": false }
  }
}
```

When the job is queued rather than run inline, `message` is `"Credit note queued for ETR signing"` and the CU fields are still `null`.

Errors:

| Status | When | Body |
|---|---|---|
| `403` | `permissions.sign` is `false` | standard forbidden response |
| `422` | The device refused the credit note, could not be reached, or the credit note cannot be signed as it stands | `{ "message": "...", "errors": { "etr": ["<reason>"] } }` |

Typical `errors.etr` reasons:

- `The landlord of this property has no KRA PIN.`: add the PIN on the landlord, then sign again.
- `No URL is configured for the incotex ETR device.`: set the property's `esd_config.url` (or ask for the default device to be configured).
- `The incotex ETR device rejected the document (HTTP 4xx): ...`: the device's own message follows.
- `The credit note has no signed invoice to reference.`: sign the original invoice first.

The last failure is also saved on the credit note as `etr_error`, so show it next to the Sign button. Signing again clears it.

ETR fields on the credit note resource:

| Field | Type | Notes |
|---|---|---|
| `cu_invoice_number` | string\|null | CU invoice number from the device |
| `cu_serial_number` | string\|null | CU serial number |
| `cu_credit_note_verify_url` | string\|null | KRA verification URL (render as a QR code) |
| `etr_signed_at` | object\|null | `{ raw, formatted, diff }` when the system signed it |
| `etr_error` | string\|null | Reason the last signing attempt failed |

Reversal fields on the credit note resource:

| Field | Type | Notes |
|---|---|---|
| `reversed_invoice_id` | integer\|null | The invoice id when this credit note fully reverses it, otherwise `null`. See [Full reversal or partial credit](#full-reversal-or-partial-credit) |
| `reissued_invoice_id` | integer\|null | The invoice raised when this signed credit note was cancelled |

## Delete Credit Note

`DELETE /api/v1/app/{company}/property-management/lease-management/credit-notes/{creditNote}`

A credit note signed on the ETR cannot be deleted, because KRA has it. Cancel it instead, which
re-bills the credit with a new invoice. Deleting a signed one returns `422` with
`errors.status: ["A credit note signed on the ETR cannot be deleted. Cancel it instead."]`.

## Dispute Credit Note

`POST /api/v1/app/{company}/property-management/lease-management/credit-notes/{creditNote}/dispute`

Request body:

| Field | Required | Type | Notes |
|---|---|---|---|
| `details` | Yes | string | Dispute description |
| `ticket_upload_id` | No | integer | Optional attachment upload id (`uploads.id`) |

## Withdraw Credit Note Dispute

`DELETE /api/v1/app/{company}/property-management/lease-management/credit-notes/{creditNote}/dispute`

## Frontend Error Handling

Apply shared rules in `docs/frontend/app/README.md`.
