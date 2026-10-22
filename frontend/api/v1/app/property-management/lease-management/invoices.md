# Invoices API

Domain: `Property Management > Lease Management`

Base route:

`/api/v1/app/{company}/property-management/lease-management/invoices`

## Endpoints

- `GET /invoices`
- `POST /invoices`
- `GET /invoices/{invoice}`
- `PUT/PATCH /invoices/{invoice}`
- `PATCH /invoices/{invoice}/cancel`
- `PATCH /invoices/{invoice}/backdate`: change the invoice's ledger date. See [Change Invoice Date (Backdate)](#change-invoice-date-backdate)
- `PUT/PATCH /invoices/{invoice}/supporting-document`: attach, replace or remove just the supporting document. See [Supporting document](#supporting-document)
- `POST /invoices/{invoice}/sign`
- `POST /invoices/{invoice}/dispute`
- `DELETE /invoices/{invoice}/dispute`
- `DELETE /invoices/{invoice}`
- `GET /leases/{lease}/invoice-components`: the components a manual invoice on that lease can
  bill. See [Components a manual invoice can bill](#components-a-manual-invoice-can-bill)

## List Invoices

`GET /api/v1/app/{company}/property-management/lease-management/invoices`

Supported query params:

- Filters:
  - `filter[search]` — free text
  - **Exact** (`=`): `filter[lease_id]`, `filter[status]`, `filter[facility_id]` (through the
    lease), `filter[user_id]` (the lease's tenant)
  - **Partial** (`LIKE`): `filter[due_at]`, `filter[created_at]`
  - **Date range**: `filter[created_between]`, `filter[due_between]` — see below
  - **Fiscal**: `filter[etr_status]`, `filter[cu_validation_status]` — see below
- Sort:
  - `sort=id,due_at,created_at,amount,tax,total,paid,balance`
- Include: not supported
> **Exact versus partial matters here.** `lease_id` and `status` are exact. They were declared as
> bare strings, which Spatie turns into a *partial* match — so `filter[lease_id]=6` returned leases
> 6, 16, 26, 60-69 and anything else whose id contains a 6, putting another tenant's billing on a
> lease's page. `filter[status]=paid` also matched `unpaid` and `partially paid`.
>
> Dates stay partial on purpose, so `filter[created_at]=2026-09` means "that month".

### Fiscal status: signed and validated

Two different questions, each with its own filter.

`filter[etr_status]` — has our control unit stamped the document?

| Value | Colour | Meaning |
| --- | --- | --- |
| `signed` | `success` | It carries a CU invoice number. |
| `failed` | `danger` | It has no CU number, and the last signing attempt failed (`etr_error` is set). |
| `not signed` | `secondary` | It has no CU number and no recorded failure. |

Returned on each record as `etr_status`, the usual `{ value, color }` pair. Note the space in
`not signed`; URL-encode it in the filter. Every document has exactly one of the three, so the
three counts add up to the unfiltered total, and an unrecognised value returns an empty list rather
than an error.

A CU number outranks an error: a document signed after an earlier failed attempt is `signed`.
Signing again clears `etr_error`, so a retried document leaves `failed` as soon as the attempt
starts.

A document counts as signed once it carries a CU invoice number, whether our own signer wrote it or
it came across from another system. `etr_signed_at` sits beside it but is not the test: it records
that we ran the step, while the number is the evidence KRA holds the document. Read `etr_status`
rather than deriving the state from the timestamp.

`emailed_at` — when the invoice was first emailed to the tenant, and `last_reminded_at` — when the
latest payment reminder went. Each is `{ raw, formatted, diff }`, or `null` when it has not happened.
A taxed invoice is only emailed once it is ETR signed.

`filter[cu_validation_status]` — has KRA accepted that number? `pending | passed | failed |
skipped`, with `cu_validation_failure_reason` carrying the text when it is `failed`.

**These are sequential, not interchangeable.** Signing is where a CU number comes from; validating
it is queued immediately afterwards. So a freshly issued document is normally `signed` with
`cu_validation_status` still `pending`, and stays that way until KRA answers. A single combined
"fiscal status" badge will misreport every new document for that whole window.

### Each row names its property

The lease on every invoice carries the property it belongs to:

```json
"lease": {
  "id": 101,
  "facility": { "id": 60, "name": "Kahawa House" },
  "user": { "id": 42, "name": "Acme Traders Ltd" },
  "currency": { "id": 1, "code": "KES" }
}
```

Read `lease.facility.name` for a Property column, and `lease.facility.id` to confirm
`filter[facility_id]` did what you asked. It is eager loaded, so it costs one query for the page
rather than one per row.

The show, create and update responses carry the same `lease.facility`, so the invoice view can
name the property and keep the spaces (`invoice_items[].lease_item_component.lease_item.facility_space`)
as secondary text.

**An empty result from `filter[facility_id]` is usually scoping, not a broken filter.** Staff see
only the invoices of properties they are allocated to, so filtering to a property outside that set
returns nothing — correctly. Compare against an unfiltered page before reporting it as a bug.

### Date ranges

`filter[created_between]=2026-09-01,2026-09-30`
`filter[due_between]=2026-09-01,2026-09-30`

Two dates as `Y-m-d,Y-m-d`. The **end date is inclusive of its whole day**, so an invoice created
at 23:40 on the 30th is in the range above. Dates are read in Nairobi time, which is what
timestamps are stored in, so send plain `Y-m-d` and do no conversion.

They are separate names rather than an extension of `filter[created_at]`, which keeps working
exactly as documented: a single value there is still a partial match, so `filter[created_at]=2026-09`
still means that month. Use whichever suits the control you are building — a month picker or a
range picker.

A range that is backwards, malformed, incomplete, or names a date that does not exist is a **422**
keyed to the filter itself:

```json
{ "message": "Use two valid dates in ascending order: Y-m-d,Y-m-d.",
  "errors": { "filter.created_between": ["Use two valid dates in ascending order: Y-m-d,Y-m-d."] } }
```

Same shape as the audit trail report's `filter[created_between]`.

> Other parts of the API express ranges as `filter[x][from]` and `filter[x][to]` — bills and
> statements, for instance. Those are unchanged and do not accept `_between`.

- Select fields: not supported
- Pagination: `per_page`, `page`

Examples:

- `GET /api/v1/app/12/property-management/lease-management/invoices?filter[lease_id]=101&sort=-due_at&per_page=10`
- `GET /api/v1/app/12/property-management/lease-management/invoices?filter[status]=pending`

### Amounts held by documents awaiting approval

A receipt or credit note that is still **awaiting approval** has posted nothing, so the
invoice's `balance` does not move until it is approved. Its amount is **held** instead: other
receipts and credit notes may use what is left, but not the held part. An invoice is never
locked as a whole.

```
available_balance = balance - (pending allocations of pending receipts + pending credit notes)
```

List rows and the show payload carry:

```json
{
  "id": 9001,
  "status": { "value": "unpaid", "color": "warning" },
  "balance": "1000.00000",
  "held_amount": 800,
  "available_balance": 200,
  "pending_holds": [
    { "type": "receipt", "id": 2500, "reference": "RCPT-1004", "amount": 600 },
    { "type": "credit_note", "id": 77, "reference": null, "amount": 200 }
  ],
  "permissions": { "pay": true, "issueCreditNote": true }
}
```

- `held_amount`, `available_balance` and each hold's `amount` are in the invoice (lease)
  currency. A receipt paid in another currency is converted at its transaction date.
- `pending_holds[].type` is `receipt` or `credit_note`. `reference` is the receipt's
  `transaction_number`, or the credit note's CU invoice number (`null` until it is signed).
- A hold ends when its document leaves `pending`: approval posts it into `balance`, and a
  rejection or cancellation releases it. Nothing needs to be done by hand.
- The invoice still reads `unpaid` / `partially paid` and still appears in
  `filter[status]=unpaid,partially paid`.
- **Receipts may go above `available_balance`.** A new receipt, an edit of a pending one or a
  reallocation that puts more on a held invoice than is available is accepted. It overpays the
  invoice once posted, and the overpaid part carries forward to the lease's next invoice (see
  [Receipts → Overpaying an invoice](./receipts.md#overpaying-an-invoice)). Documents already
  awaiting approval still post in full when approved, even onto an invoice that is paid by then.
- **A credit note raised by a user may not go above `available_balance`** (`422`), because a
  credit note may never credit more than is billed and unpaid. A credit note already awaiting
  approval is applied in full when approved, even if receipts have paid the invoice meanwhile;
  the invoice then goes below zero and carries forward like any overpayment.
- `permissions.pay` is `false` only once the invoice `balance` is zero or below.
  `permissions.issueCreditNote` is also `false` when something is held and nothing is available.
- In receipt pickers, show the available amount ("200.00 available, 800.00 held by pending
  documents") and **warn** when the input is above it; do not cap it. A pending receipt being
  edited holds money of its own: its available amount on an invoice is `available_balance` plus
  what it already has there. In credit-note forms, cap the input at `available_balance`.

### Where the paid amount came from

`GET /invoices/{invoice}` explains `paid` line by line, including money carried over from another
invoice.

```json
{
  "id": 9010,
  "total": "5000.00000",
  "paid": "5000.00000",
  "balance": "0.00000",
  "payments": [
    {
      "id": 3301,
      "amount": "2000.00000",
      "receipt": { "id": 2501, "transaction_number": "RCPT-1005" },
      "carried_from_invoice": { "id": 9001, "total": "5000.00000", "status": { "value": "paid", "color": "success" } },
      "carried_forward": []
    },
    {
      "id": 3310,
      "amount": "0.00000",
      "receipt": { "id": 2600, "transaction_number": "RCPT-1100" },
      "carried_from_invoice": null,
      "carried_forward": [
        { "id": 3320, "amount": "1200.00000", "invoice": { "id": 9022, "total": "5000.00000" } }
      ]
    }
  ],
  "credit_applications": [
    {
      "id": 502,
      "amount": "3000.00000",
      "applied_at": { "raw": "2026-11-01T00:00:00Z", "formatted": "01 Nov, 2026", "diff": "now" },
      "credit_note": { "id": 77, "notes": "Opening balance", "total": "-8000.00000", "status": { "value": "applied", "color": "success" } }
    }
  ]
}
```

- **`payments[]`** are receipt allocations on this invoice.
  - **`carried_from_invoice`** is set when the money is an overpayment that moved here from
    another invoice when this one was processed (see
    [Receipts → Overpaying an invoice](./receipts.md#overpaying-an-invoice)). Show it as "Receipt
    RCPT-1005, carried from Invoice #9001". The receipt's collections stay on the original
    invoice, where the money was received.
  - **`carried_forward[]`** goes the other way. It lists the parts of this allocation that later
    moved on to other invoices, with each part's amount and the invoice it went to. The
    allocation's own `amount` is what stayed here. An allocation whose whole amount moved on
    reads `0` here and keeps its `carried_forward` rows, so the audit trail is complete.
  - Carry-overs made before 2026-10-21 have `carried_from_invoice: null` and no
    `carried_forward`: the source was not recorded then.
- **`credit_applications[]`** list every credit note that paid part of this invoice, with the
  amount it paid here, including credit notes raised on another invoice or raised open (see
  [Credit notes → Applications](./credit-notes.md#applications-which-invoices-a-credit-note-paid)).
  `credits` is unchanged. It still lists only the credit notes whose `invoice_id` is this invoice.
- `paid` = sum of `payments[].amount` + sum of `credit_applications[].amount` (within rounding).
  Invoices paid before 2026-10-21 may not add up exactly; see the notes above.

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

## Create Invoice

`POST /api/v1/app/{company}/property-management/lease-management/invoices`

Request body:

| Field | Required | Type | Allowed Values / Notes |
|---|---|---|---|
| `lease_id` | Yes | integer | - |
| `due_at` | Yes | date (`YYYY-MM-DD`) | - |
| `notes` | Yes | string | - |
| `items` | Yes | array | At least one item |
| `cu_reference_number` | No | string | Optional |
| `is_credit` | No | boolean | Defaults to `false` |
| `paid` | No | number | Defaults to `0` |
| `cu_invoice_number` | No | string | Optional |
| `cu_serial_number` | No | string | Optional |
| `cu_invoice_verify_url` | No | string | Optional |
| `supporting_upload_id` | No | integer | One upload id (`uploads.id`) backing the invoice: a quote, a work order, an agreed charge. Optional. See [Supporting document](#supporting-document) |

`items[]` fields:

| Field | Required | Type | Allowed Values / Notes |
|---|---|---|---|
| `lease_component_id` | One of the two | integer | A catalogue component (`lease_components.id`). The API picks the lease line to bill — see below |
| `lease_item_component_id` | One of the two | integer | A line already on the lease. Must belong to the supplied lease. Wins when both are sent |
| `notes` | Yes | string | - |
| `quantity` | Yes | integer | Minimum `1` |
| `amount` | Yes | number | Minimum `0` |
| `tax_id` | Yes | integer | Must exist in `taxes.id` |

Notes:

- Two items may not end up on the same lease line. `lease_item_component_id` must be unique within
  the request, and that includes the lines the API picks for `lease_component_id`.
- The create screen sends `lease_component_id`. `lease_item_component_id` stays for the screens and
  imports that already know the exact line.

### Billing a catalogue component

When an item names a `lease_component_id`, the API finds the lease line to bill:

| The component is… | The lease carries it | Result |
|---|---|---|
| Autobilled, or a rent, service charge, parking or signage component | On any of its spaces | Billed on that line — the first space that carries it. Rent held on space A and service charge on space B bill on A and B respectively |
| Autobilled, or a rent, service charge, parking or signage component | No | **422** |
| Not autobilled (electricity, water, deposit, excess service charge, …) | On any of its spaces | Billed on that line |
| Not autobilled | No | A line is added to the lease's first space with a cost of `0`, and the item is billed on it |

A line added this way costs `0`, so the billing cycle never bills it. Rent, service charge, parking
and signage are never added, even when a company has turned their autobilling off, because those
lines decide which lease occupies a space.

The 422 is keyed to the item:

```json
{
  "message": "Service Charge is billed automatically and is not on this lease, so it cannot be invoiced here.",
  "errors": {
    "items.1.lease_component_id": [
      "Service Charge is billed automatically and is not on this lease, so it cannot be invoiced here."
    ]
  }
}
```

A lease with no spaces at all answers 422 on `lease_id`, since there is nothing to bill against. An
inactive catalogue component answers 422 on its item.

### Which leases can be invoiced

A manual invoice can be raised on an `active`, `suspended` or `terminated` lease. A `pending` or
`rejected` lease answers **422** on `lease_id`, and nothing is written:

```json
{
  "message": "This lease is pending. Invoices can only be raised on active, suspended or terminated leases.",
  "errors": {
    "lease_id": [
      "This lease is pending. Invoices can only be raised on active, suspended or terminated leases."
    ]
  }
}
```

A pending lease is billed only once it is approved or activated. At that point its deposits and
opening balances are raised, and the billing cycle picks it up. The billing cycle and
`POST .../leases/{lease}/generate-invoice-for-next-period` stay limited to `active` leases. Hide or disable
**Raise Invoice** while the lease is `pending` or `rejected`.

### Components a manual invoice can bill

`GET /api/v1/app/{company}/property-management/lease-management/leases/{lease}/invoice-components`

Feeds the component dropdown on the create screen. It returns every active catalogue component,
not paginated: every component that can be invoiced first, then every one that cannot, each in
`sort_order` (see Lease Components in the settings doc), and works out the table above for this lease, so the screen can
disable what the API would refuse.

Requires permission to create invoices and to view the lease; otherwise `403`.

```json
{
  "data": [
    {
      "id": 3,
      "name": "Rent",
      "tax_id": 1,
      "is_autobilled": true,
      "lease_item_component_id": 455,
      "facility_space": { "id": 88, "name": "Shop 4" },
      "amount": 950,
      "selectable": true,
      "reason": null
    },
    {
      "id": 4,
      "name": "Service Charge",
      "tax_id": 1,
      "is_autobilled": true,
      "lease_item_component_id": null,
      "facility_space": null,
      "amount": null,
      "selectable": false,
      "reason": "Service Charge is billed automatically and is not on this lease, so it cannot be invoiced here."
    },
    {
      "id": 9,
      "name": "Water",
      "tax_id": 2,
      "is_autobilled": false,
      "lease_item_component_id": null,
      "facility_space": null,
      "amount": null,
      "selectable": true,
      "reason": null
    }
  ]
}
```

- `lease_item_component_id`, `facility_space` and `amount` describe the lease line the item would
  be billed on. They are `null` when the lease does not carry the component yet.
- `amount` is that one line's monthly cost. A lease carrying the component on several spaces
  still bills it as one item, so the create screen prefills the item with the sum of
  `cost_per_month` over every line of the lease that carries the component (from the lease show
  payload), and the quantity with the months in the lease's billing cycle. A line added at `0` has
  nothing to prefill, so leave the amount for the user.
- `tax_id` is the line's tax when the lease carries the component, otherwise the catalogue's.
- `selectable: false` comes with a `reason`. Show the option disabled with the reason as a hint.
  The create endpoint enforces the same rule, so a stale screen still gets the 422 above.

Example request:

```json
{
  "lease_id": 101,
  "due_at": "2026-03-01",
  "notes": "March 2026 billing",
  "cu_reference_number": "INV-REF-3001",
  "items": [
    {
      "lease_component_id": 3,
      "notes": "Base rent",
      "quantity": 1,
      "amount": 950,
      "tax_id": 1
    }
  ]
}
```

Example response:

```json
{
  "message": "Invoice created successfully",
  "invoice": {
    "id": 9001,
    "lease": { "id": 101 },
    "amount": 950,
    "tax": 152,
    "total": 1102,
    "status": { "value": "pending", "color": "secondary" }
  }
}
```

## Update Invoice

`PUT/PATCH /api/v1/app/{company}/property-management/lease-management/invoices/{invoice}`

Body: the header - `due_at`, `notes`, `cu_invoice_number`, `cu_serial_number`,
`cu_invoice_verify_url` - and, optionally, `items`. **A field left out keeps its value** (row 138;
before, the CU fields and an opening balance's `transaction_at` were cleared by any edit that did not
send them), so send just what the form shows. `lease_id` may be left out; if sent it must be the
invoice's own (`422` - an invoice cannot be moved to another lease). `lease_id` is returned on the
invoice.

- **`items` sent:** they are the invoice's new lines, complete. The old lines are replaced and
  `amount`, `tax`, `total` and `balance` worked out again, as on create. Each item names its lease
  line with `lease_item_component_id` (or a catalogue `lease_component_id`, resolved as on create),
  with `notes`, `quantity`, `amount` (unit cost) and `tax_id`. A lease line may appear as often as
  the invoice already bills it - a generated invoice whose period straddles an escalation has one
  line per rate on the same lease line - and otherwise once.
- **`items` left out:** the lines stay as they are; only the header changes.
- **Lines that cannot be replaced here** (`422` on `items`, header edits still work): an invoice that
  bills a deposit ("Change the deposit instead.") or an opening balance ("Change the opening balance
  instead."), a utility invoice (its lines carry meter readings), and one with payments or credit
  notes against it.
- **`permissions.updateLines`** says up front whether `items` will be accepted: `true` when the
  invoice may be edited and its lines are not one of the kinds above. When it is `false` (and
  `permissions.update` is `true`), show the lines read-only and send the header without `items`.
- `paid` is never taken from the body (it used to be written, defaulting to 0).

Before row 138 this endpoint saved the header only and dropped whatever lines it was sent.

**Only a `pending` invoice is edited** (row 138). Once approved (or cancelled, rejected) the edit is
refused with `403` `"Only a pending invoice can be edited."`, whatever the caller's permissions, and `permissions.update` is `false`.
Hide the Edit action unless it is `true`. An invoice returned for changes is still `pending`, so its creator
can answer it, and the edit sends it round its chain again. Who may edit a pending one: the update
permission, the creator, whoever it was returned to, and an approver whose current step lends editing
rights (limited to the step's fields when the step names them).

An approver whose step names fields saves them through `PATCH invoices/{invoice}/step-fields` - see
[approvals](../../access-management/approvals.md#saving-only-the-steps-fields-step-fields-row-138).

## Supporting document

An invoice raised by hand can carry **one** supporting document (row 171), the way a credit note does.
It is **optional**. Invoices the system raises itself (monthly billing, deposits, utility billing,
imports) have none.

- **Create:** upload the file through the uploads endpoint, then send its id as `supporting_upload_id`
  with `POST /invoices`. Only an upload **you** made can be attached: anyone else's answers `422`,
  `errors.supporting_upload_id: ["Attach a document you uploaded yourself."]`. An unknown id answers
  `422` on the same key.
- **Edit while pending:** in `PUT /invoices/{invoice}`, leave `supporting_upload_id` out to keep the
  document. Send an id to replace it, or `null` to remove it.
- **Its own endpoint:** `PUT/PATCH /invoices/{invoice}/supporting-document` with
  `{ "supporting_upload_id": 512 }` (or `null` to remove), without resending the rest of the invoice.
  The key must be sent. Saving here resubmits an invoice that was sent back for changes, as a full
  edit does. Response: `{ "data": { "message": "Supporting document updated successfully.", "invoice": { ... } } }`.
- **Once approved it is fixed:** an approved invoice is no longer edited (row 138), so both the edit and
  the document endpoint answer `403` "Only a pending invoice can be edited."
- **Read it** as `supporting_upload_id` and `supporting_upload` (an upload object, with `file_name`
  and `source_url`) on the invoice: in create, show and update responses.
- **The tenant sees it** on their invoice in the tenant portal. It is **not** attached to the invoice
  email.

## Cancel Invoice

`PATCH /api/v1/app/{company}/property-management/lease-management/invoices/{invoice}/cancel`

No request body required.

`permissions.cancel` is **false** on an invoice that cannot be cancelled — one already
cancelled, a rejected one (which can only be deleted), and a draft (`pending`, where
cancelling would delete it outright rather than withdraw it). Drive the button off the flag;
a direct call on any of those returns **403**.

On an invoice with payments allocated, `permissions.cancel` is still `true`, but cancelling is
refused with **422** and `status: ["This invoice has payments allocated to it. Reallocate them to
another invoice before removing it."]`. Reallocate the payments first, then cancel.

Possible status values returned by API resource:

- `pending` (`secondary`)
- `unpaid` (`warning`)
- `partially paid` (`primary`)
- `paid` (`success`)
- `cancelled` (`danger`)
- `rejected` (`danger`) — the approval chain was rejected. The invoice was never issued: it posts
  nothing to the tenant's statement and counts in no balance, ageing or VAT report. It can only
  be viewed or deleted (delete removes it outright, with no reversal); `cancel`, `sign`,
  `dispute`, `update` and every other `permissions` flag except `view`/`delete` are `false`, and
  the endpoints refuse it

`cancelled` is only ever set by this endpoint now; a rejection never produces it.

## Change Invoice Date (Backdate)

`PATCH /api/v1/app/{company}/property-management/lease-management/invoices/{invoice}/backdate`

This moves an issued invoice to a different date in the tenant's ledger. Show a **Change Date**
item in the invoices list row actions and in the view page header actions, but only when
`invoice.permissions.backdate === true`. That flag is already `false` for invoices that are
pending, cancelled, rejected, or reversed by a credit note.

Request body:

| Field | Required | Type | Notes |
|---|---|---|---|
| `new_invoice_date` | Yes | string (`YYYY-MM-DD`) | Today or earlier. Cap the date picker at today. |

```json
{ "new_invoice_date": "2026-08-15" }
```

Behaviour:

- The invoice's tenant statement lines (`transaction_at`, `created_at`) and its lease billings
  (`transaction_date`, `created_at`) move to the new date. Statement lines keep their time of day.
- The invoice's own `created_at` and `due_at` never change, so the list's date column stays the
  same. The move shows on the tenant statement and in billing reports.

Success (`200`) returns `{ data: { message, invoice } }`. Upsert the invoice with it.

Errors:

- `422` `errors.new_invoice_date`: the date is missing, invalid, or in the future. Show the
  message under the date picker.
- `403`: the user lacks `backdate-facility-invoice`, or the invoice is no longer eligible.

Backend reference: `docs/backend/api/v1/app/invoice-backdate.md`.

## Sign Invoice (ETR)

`POST /api/v1/app/{company}/property-management/lease-management/invoices/{invoice}/sign`

Signs the invoice on the property's ETR (KRA fiscal) device and stores the CU details on the invoice. No request body.

The device comes from the property's `esd_type` / `esd_config` (see Facilities). A property with no device set uses the system default (Incotex). The seller PIN sent to the device is always the **landlord's KRA PIN**, and the buyer PIN is the tenant's.

When to show the action: `permissions.sign` on the invoice resource is `true`. It is `false` when:

- the user lacks the `sign-facility-invoice` permission
- the invoice is `pending` (awaiting approval), `cancelled` or `rejected`
- the invoice is already signed (`cu_invoice_number` is set)

Signing runs as a queued job. On a synchronous queue (the default locally) the request returns after the device answers; on a real queue it returns straight away and the CU fields fill in once the job runs. Refetch the invoice (or poll `GET /invoices/{invoice}`) until `etr_signed_at` or `etr_error` is set.

Success response (`200`):

```json
{
  "message": "Invoice signed successfully",
  "invoice": {
    "id": 9001,
    "cu_invoice_number": "0010000042",
    "cu_serial_number": "KRAMW017202209000001",
    "cu_invoice_verify_url": "https://itax.kra.go.ke/KRA-Portal/invoiceChk.htm?actionCode=loadPage&invoiceNo=0010000042",
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

When the job is queued rather than run inline, `message` is `"Invoice queued for ETR signing"` and the CU fields are still `null`.

Errors:

| Status | When | Body |
|---|---|---|
| `403` | `permissions.sign` is `false` | standard forbidden response |
| `422` | The device refused the invoice, could not be reached, or the invoice cannot be signed as it stands | `{ "message": "...", "errors": { "etr": ["<reason>"] } }` |

Typical `errors.etr` reasons:

- `The landlord of this property has no KRA PIN.`: add the PIN on the landlord, then sign again.
- `No URL is configured for the incotex ETR device.`: set the property's `esd_config.url` (or ask for the default device to be configured).
- `The incotex ETR device rejected the document (HTTP 4xx): ...`: the device's own message follows.

The last failure is also saved on the invoice as `etr_error`, so show it next to the Sign button. Signing again clears it.

ETR fields on the invoice resource:

| Field | Type | Notes |
|---|---|---|
| `cu_invoice_number` | string\|null | CU invoice number from the device |
| `cu_serial_number` | string\|null | CU serial number |
| `cu_invoice_verify_url` | string\|null | KRA verification URL (render as a QR code) |
| `etr_signed_at` | object\|null | `{ raw, formatted, diff }` when the system signed it |
| `etr_error` | string\|null | Reason the last signing attempt failed |

## Delete Invoice

`DELETE /api/v1/app/{company}/property-management/lease-management/invoices/{invoice}`

One endpoint, four outcomes, decided by the invoice.

| Invoice | Outcome |
|---|---|
| **Payments allocated** | Refused. `permissions.delete` is `false`, so hide the action; a direct call returns **403**. The money is reallocated to another invoice first |
| **Issued and carrying a CU number** | **Reversed**, not deleted. A credit note for the full value is raised and signed to ETR. The invoice survives |
| **Live and unfiled** (unpaid, partially paid) | Refused — **cancel it first**. `permissions.delete` is `false`; a direct call returns **403** with `"Cancel this invoice before deleting it."` |
| **Cancelled, pending or rejected** | Removed, along with its tenant statement entries and lease billings |

### Cancel before delete

Deleting is not a stronger cancelling. **Cancel** keeps the invoice and posts an offsetting line to
the tenant statement; **Delete** removes the invoice *and* those entries, so an invoice the tenant
has already had would leave nothing behind saying it existed.

So an invoice that reached the tenant has to be cancelled first — the cancellation is that record.
In practice deleting is now a two-step action for most invoices, and the Delete button simply will
not appear on a live one.

A **pending** invoice is still removed outright, even if it carries a CU number, and so is a
**rejected** one: neither reached the tenant, so there is nothing to withdraw. Cancelling is
refused on both — a rejected invoice *"can only be deleted"*, and cancelling a pending one deletes
it — so requiring cancellation first would leave them undeletable for good.

Response:

| Field | Notes |
|---|---|
| `action` | `"cancelled"` or `"reversed"`. Always present — switch on this |
| `credit_note_id` | The reversing credit note. **Absent** when nothing was reversed, because null values are stripped from responses |

### Reversed invoices read as "paid"

Applying a full credit note drives the balance to zero, and the status follows the balance — so a
reversed invoice comes to rest as `paid` when nobody paid anything.

`reversal_credit_note_id` on the invoice is what tells the two apart, and
**`filter[reversed]=true|false`** filters on it. Combine it to get real revenue:

```
filter[status]=paid&filter[reversed]=false
```

An invoice counts as reversed only when a credit note covers its **full total**, whether the
credit note came from deleting or cancelling a filed invoice or was raised by hand. A partial credit
note leaves `reversal_credit_note_id` unchanged. Cancelling an **unsigned** reversing credit note
clears the field and reopens the invoice.

The same filter exists on the credit notes list, and a reversing credit note carries
`reversed_invoice_id` (`null` on a partial credit). The credit note's `invoice_id` is the only
stored link. `reversed_invoice_id` is worked out from it and the invoice's
`reversal_credit_note_id`.

The statement line reads `CN#{id} - Reversal for INV#{id} - {invoice notes}`.

### Invoices re-issued from a cancelled credit note

A credit note signed on the ETR cannot be withdrawn. Cancelling one raises a new invoice for the
same amount instead (see the credit notes doc). That invoice carries
**`reissued_from_credit_note_id`** (`null` on every other invoice), and
**`filter[reissued]=true|false`** filters on it. Its notes and statement line read
`INV#{id} - Re-issued for cancelled ETR signed CN#{creditNoteId} - {credit note notes}`.

## Dispute Invoice

`POST /api/v1/app/{company}/property-management/lease-management/invoices/{invoice}/dispute`

**`permissions.dispute` says whether this invoice can be disputed at all**, and it now follows the
invoice's status. Read it per record rather than per user: holding `dispute-facility-invoice` is
necessary but not sufficient.

| Status | `permissions.dispute` |
| --- | --- |
| `unpaid`, `partially paid` | `true` |
| `pending`, `paid`, `cancelled`, `rejected` | `false` |

A paid invoice is refused alongside the closed ones — there is nothing outstanding left to argue
with.

The endpoint enforces the same rule, so a `false` flag and a `422` keyed `invoice` are the same
decision reached twice. Previously the flag said `true` on paid, cancelled and rejected invoices
while the endpoint refused them, so the button appeared and then failed; that is fixed, and the two
now read one rule.

Request body:

| Field | Required | Type | Notes |
|---|---|---|---|
| `details` | Yes | string | Dispute description |
| `ticket_upload_id` | No | integer | Optional attachment upload id (`uploads.id`) |

## Withdraw Invoice Dispute

`DELETE /api/v1/app/{company}/property-management/lease-management/invoices/{invoice}/dispute`

## Public Invoice Page

Every invoice has a public link that anyone holding it can open without signing in, for example
from a reminder SMS or email. Invoice responses carry it as `public_url`:

```json
{
  "id": 178,
  "public_url": "https://api.example.com/invoices/Yx3...40 random chars"
}
```

The link ends in a random 40-character token, not the invoice id, so invoice numbers cannot be
guessed. The token is issued when the invoice is created; invoices that existed before this
feature were given one by the migration.

These are web pages served by the backend, not JSON endpoints:

| Route | Returns |
|---|---|
| `GET /invoices/{token}` | HTML page. The left column shows the invoice number, dates, currency and status; the ETR (CU invoice number, CU serial number, KRA verify link and QR); the tenant, landlord and property; the line items; the receipts and credit notes allocated to the invoice; a small ageing analysis of the balance; and a footer with Amount, Tax, Total, Paid and Balance. The right column shows how to pay (the M-Pesa paybill and account number, and the bank accounts that collect the invoiced charges) and a **Reprint** button |
| `GET /invoices/{token}/pdf` | The invoice as a PDF (`inline`, `INV0178.pdf`), printed with the invoice reminder template chosen in the `invoice_reminder_template_id` setting (see [Settings](../settings.md)) |

Rules:

- An unknown token returns `404`. So does a `pending` invoice, since it has not been approved or
  issued yet, and a `rejected` one, which never will be.
- The payment options are hidden when the balance is zero or less, or the invoice is `cancelled`.
- **Reprint** is hidden, and `/pdf` returns `404`, when the company has no active invoice template
  the invoice's property can use.
- Both routes are limited to 60 requests a minute per IP, and are marked `noindex`.

## Frontend Error Handling

Apply shared rules in `docs/frontend/app/README.md`.
