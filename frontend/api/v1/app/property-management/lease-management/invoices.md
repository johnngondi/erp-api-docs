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

### Invoices already being paid by a pending receipt

An invoice that sits on a receipt which is still **awaiting approval** (receipt status `pending`)
is locked: it cannot be put on another receipt, credited by a new credit note, or have more
money reallocated onto it until that receipt is confirmed, rejected or cancelled. It still
reads `unpaid` / `partially paid` and **still appears** in the list, so a picker built from
`filter[status]=unpaid,partially paid` keeps showing it. Both list rows and the show payload say
which receipt holds it:

```json
{
  "id": 9001,
  "status": { "value": "unpaid", "color": "warning" },
  "balance": "1102.00000",
  "pending_receipt": { "id": 2500, "transaction_number": "RCPT-1004" },
  "permissions": { "pay": false, "issueCreditNote": false }
}
```

- `pending_receipt` is `null` when nothing locks the invoice. A rejected, cancelled or confirmed
  receipt never locks.
- `permissions.pay` and `permissions.issueCreditNote` are `false` while locked; the receipt and
  credit-note endpoints refuse the invoice with a `422` naming the receipt (see those docs).
- Show the invoice in pickers with a "being paid in receipt #2500" hint and do not let it be
  selected; the receipt that holds it may still be edited to use it (its own allocations do not
  lock it against itself).

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

### Components a manual invoice can bill

`GET /api/v1/app/{company}/property-management/lease-management/leases/{lease}/invoice-components`

Feeds the component dropdown on the create screen. It returns every active catalogue component,
sorted by name and not paginated, and works out the table above for this lease, so the screen can
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
- `amount` is the line's monthly cost, so use it to prefill the item. A line added at `0` has
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

Use the same payload shape as create.

## Cancel Invoice

`PATCH /api/v1/app/{company}/property-management/lease-management/invoices/{invoice}/cancel`

No request body required.

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

One endpoint, three outcomes, decided by the invoice.

| Invoice | Outcome |
|---|---|
| **Payments allocated** | Refused. `permissions.delete` is `false`, so hide the action; a direct call returns **403**. The money is reallocated to another invoice first |
| **Issued and carrying a CU number** | **Reversed**, not deleted. A credit note for the full value is raised and signed to ETR. The invoice survives |
| **Pending, or no CU number** | Cancelled and removed, along with its tenant statement entries and lease billings |

A **pending** invoice is always removed outright, even if it carries a CU number: it is a draft the
tenant never received, so there is nothing filed to reverse.

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
