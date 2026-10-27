# Receipts API

Domain: `Property Management > Lease Management`

Base route:

`/api/v1/app/{company}/property-management/lease-management/receipts`

## Endpoints

- `GET /receipts`
- `POST /receipts`
- `GET /receipts/{receipt}`
- `PUT/PATCH /receipts/{receipt}`
- `PATCH /receipts/{receipt}/cancel`
- `GET /receipts/{receipt}/reallocation`
- `PUT /receipts/{receipt}/reallocation`
- `PATCH /receipts/{receipt}/backdate`
- `POST /receipts/{receipt}/dispute`
- `DELETE /receipts/{receipt}/dispute`
- `DELETE /receipts/{receipt}`

## List Receipts

`GET /api/v1/app/{company}/property-management/lease-management/receipts`

Supported query params:

- Filters:
  - `filter[id]`
  - `filter[receiving_account_id]`
  - `filter[transaction_number]`
  - `filter[transaction_date]`
  - `filter[payment_method_id]`
  - `filter[paying_user_id]`
  - `filter[amount]`
  - `filter[allocated]`
  - `filter[balance]`
  - `filter[receiving_user_id]`
- Sort:
  - `sort=id,receiving_account_id,transaction_number,transaction_date,payment_method_id,paying_user_id,amount,allocated,balance,receiving_user_id`
- Include: not supported
- Select fields: not supported
- Pagination: `per_page`, `page`

Example:

`GET /api/v1/app/12/property-management/lease-management/receipts?filter[transaction_number]=RCPT-1005&sort=-transaction_date`

### The receipt's dates

A receipt carries three dates:

- **`transaction_date`** is the day the tenant paid, as entered. It is the date on the bank
  statement and the cashbook line's `transaction_at`. It never changes on approval.
- **`created_at`** (`created` in the resource) is the day the receipt was **processed**. For a
  receipt that needs Receivables' approval, that's the day of the final approval, not the day it
  was captured. Captured on 30 Oct and approved on 2 Nov, it reads 2 Nov and falls in November.
  Its tenant statement lines, collections and cashbook entry are written on approval and carry
  the same day. A receipt that needs no approval is processed as it is captured, so its date
  doesn't move.

- **`requested_at`** is when the receipt was captured. It never changes. Filter with
  `filter[requested_between]=Y-m-d,Y-m-d`.

Allocating more money from an already approved receipt later does not move `created_at`. Only
[Change Receipt Date](#change-receipt-date-backdate) does. Receipts approved before 2026-10-27 keep
the day they were captured. See [Document dates](../document-dates.md).

## Create Receipt

`POST /api/v1/app/{company}/property-management/lease-management/receipts`

Request body:

| Field | Required | Type | Allowed Values / Notes |
|---|---|---|---|
| `transaction_number` | Yes | string | Unique business transaction number |
| `receiving_account_id` | Yes | integer | Must exist in `bank_accounts.id` |
| `transaction_date` | Yes | date (`YYYY-MM-DD`) | - |
| `payment_method_id` | Yes | integer | Must exist in `payment_methods.id` |
| `paying_user_id` | Yes | integer | Must exist in `users.id` |
| `amount` | Yes | integer | - |
| `currency_id` | No | integer | Must exist in `currencies.id` |
| `allocations` | Yes | array | Invoice allocations |
| `pop_message` | Conditional | string | Proof of payment reference. Required when the payment method's `proof_of_payment_type` is `message` or `both`; one of `pop_message` / `pop_upload_id` when it is `either` |
| `pop_upload_id` | Conditional | integer | Must exist in `uploads.id`. Required when the payment method's `proof_of_payment_type` is `document` or `both`; one of `pop_message` / `pop_upload_id` when it is `either` |

`allocations[]` fields:

| Field | Required | Type | Allowed Values / Notes |
|---|---|---|---|
| `invoice_id` | Yes | integer | Must exist in `invoices.id` |
| `amount` | Yes | integer | Allocation amount |

Example request:

```json
{
  "transaction_number": "RCPT-1005",
  "receiving_account_id": 5,
  "transaction_date": "2026-02-10",
  "payment_method_id": 2,
  "paying_user_id": 77,
  "amount": 1102,
  "currency_id": 1,
  "allocations": [
    { "invoice_id": 9001, "amount": 1102 }
  ]
}
```

Example response:

```json
{
  "message": "Receipt created successfully",
  "receipt": {
    "id": 2501,
    "public_url": "https://api.example.com/receipts/Yx3...40 random chars",
    "transaction_number": "RCPT-1005",
    "amount": 1102,
    "allocated": 1102,
    "balance": 0,
    "status": { "value": "confirmed", "color": "success" }
  }
}
```

### Validation (`422`)

| Key | Condition | Message |
|---|---|---|
| `allocations` | An invoice is not on a lease of `facility_id` | Some of the allocated invoices do not belong to leases in the selected property. |
| `allocations` | An invoice has no balance left (paid or already overpaid) | Invoice with invoice_id 9001 is already paid for fully |
| `transaction_number` | Another live receipt has the same reference, amount, property and tenant | Duplicate payment: receipt #412 already records reference QJK3X7L2MP for 25,000.00 from this tenant on this property. |

**Duplicate payments (row 178).** A receipt is the same payment as another when the **reference
(`transaction_number`), amount, property (`facility_id`) and tenant (`paying_user_id`)** all match.
The reference is compared without case or spaces (`qjk3 x7l2mp` is `QJK3X7L2MP`); the amount must be
exact. Only **live** receipts count, pending or confirmed: a cancelled, rejected or reversed receipt
no longer records the payment, so it can be entered again. Show the message on the reference field;
it names the receipt already holding the payment. The same check runs when an edit changes one of
the four (a pending receipt's full edit, or a confirmed receipt's reference); an edit that leaves
them alone is never refused by it.

### Overpaying an invoice

An allocation may be more than the invoice's balance, and more than its `available_balance`
while documents awaiting approval hold part of it (see
[Invoices → Amounts held](./invoices.md#amounts-held-by-documents-awaiting-approval)). It is
never refused for that and never cut down. The whole allocation goes onto the invoice when the
receipt is confirmed:

- the invoice `balance` goes below zero (it reads `paid`);
- the full amount becomes lease collections, spread over the invoice's components by the
  property's allocation method (by `order`, the excess goes to the highest-priority component),
  so all of it is remittable;
- the tenant statement is credited the full amount against the lease;
- the overpaid part carries forward when the lease's next invoice is processed.

Documents that were already awaiting approval on the invoice still post in full when approved,
so they may overpay it further. Nothing has to be redone.

**UI:** warn, do not block. When an allocation is above the invoice's `available_balance`,
show for example "250.00 is more than this invoice has available and will carry forward to the
tenant's next invoice." Compare in the invoice currency; the list carries
`available_balance` and `pending_holds`.

Money left on the receipt itself (an `amount` above the sum of `allocations`) is unallocated and
is not remittable until it is put on an invoice. See
[Applying unallocated money](#applying-unallocated-money).

## Update Receipt

`PUT/PATCH /api/v1/app/{company}/property-management/lease-management/receipts/{receipt}`

What the endpoint accepts depends on the receipt's status.

### Editing a pending receipt

A receipt that is still `pending` (awaiting approval) has posted nothing, so it is edited in
full with the **same body as Create**: `facility_id`, `transaction_number`,
`receiving_account_id`, `transaction_date`, `payment_method_id`, `paying_user_id`, `amount`,
`currency_id` and `allocations[]`. The allocations are replaced wholesale by the ones sent, as
`pending`. Omit `pop_message` / `pop_upload_id` to keep the current proof of payment (see
**Proof of payment** below for what sending them means).

The same `422` rules as Create apply, except that an invoice the receipt already has a pending
allocation on is not refused for having no balance left: another document may have paid it in
the meantime, and this receipt still overpays it when approved. The overpayment warning is
worked out the same way, with the receipt's own pending allocations added back to
`available_balance`.

Who may do it: anyone with `update-facility-receipt`, or, while the receipt is returned for
changes, its creator and the person who submitted it for approval.

#### Approval returned for changes

When the first approval step answers `review`, the receipt comes back to its creator and
submitter with a "Changes requested" task, and their `permissions.update` turns `true`. They fix
it through this endpoint (full body, `pop_*` omitted to keep the proof). That save
**resubmits** it: the step goes back to the reviewer alone. There is no separate resubmit call.
See [Approvals → Returned for changes](../../access-management/approvals.md#returned-for-changes).

### Editing a confirmed receipt

Once confirmed, only the fields below may change:

| Field | Required | Type | Allowed Values / Notes |
|---|---|---|---|
| `transaction_number` | Yes | string | - |
| `receiving_account_id` | Yes | integer | Must exist in `bank_accounts.id` |
| `transaction_date` | Yes | date (`YYYY-MM-DD`) | - |
| `payment_method_id` | Yes | integer | Must exist in `payment_methods.id` |
| `paying_user_id` | Yes | integer | Must exist in `users.id` |
| `receiving_user_id` | Yes | integer | Must exist in `users.id` |
| `pop_message` | No | string\|null | See **Proof of payment** below |
| `pop_upload_id` | No | integer\|null | See **Proof of payment** below |

### Proof of payment

The receipt's proof of payment — a reference (`pop_message`), a document (`pop_upload_id`) or both —
is set by the payment method's `proof_of_payment_type`, which is on the payment method resource as
one of `none`, `message`, `document`, `both` or `either`.

| `proof_of_payment_type` | On create |
|---|---|
| `none` | Neither field is required |
| `message` | `pop_message` is required |
| `document` | `pop_upload_id` is required |
| `both` | Both are required |
| `either` | At least one of the two. Sending neither returns `422` on **both** `pop_message` and `pop_upload_id`, with the same message on each, because either field would satisfy it |

On an update each of the two fields reads three ways, so **sending the key and leaving it out mean
different things**:

| You send | Result |
|---|---|
| the key is absent | The current value is kept |
| `"pop_upload_id": null` | The proof is removed |
| `"pop_upload_id": 41` | The proof is replaced |

So correcting a document that was uploaded by mistake is a normal update carrying the new
`pop_upload_id`. Nothing else about the receipt has to be resent for the proof to survive — an edit
that changes only the transaction date leaves the document attached.

**Removal is refused while the payment method requires that proof.** A `null` against a `document`
or `both` method returns `422` on `pop_upload_id`:

```json
{
  "message": "The proof of payment document cannot be removed while this payment method is selected. Upload the correct document to replace it.",
  "errors": {
    "pop_upload_id": ["The proof of payment document cannot be removed while this payment method is selected. Upload the correct document to replace it."]
  }
}
```

`pop_message` behaves the same way against a `message` or `both` method. Replacing always works;
only removal is gated. This matters for the withholding methods in particular, where the uploaded
document *is* the withholding tax certificate that the tenant withholding report links to.

`pop_message` behaves identically against a `message` or `both` method. On a `both` method each
half holds independently: either may be replaced, neither may be removed.

On an `either` method only the **last** proof is held. Removing the document while a message is on
the receipt succeeds, and vice versa. Removing the one that remains returns `422` on both
`pop_message` and `pop_upload_id`:

```json
{
  "message": "The last proof of payment cannot be removed while this payment method is selected. Keep a message or a document on the receipt.",
  "errors": {
    "pop_message": ["The last proof of payment cannot be removed while this payment method is selected. Keep a message or a document on the receipt."],
    "pop_upload_id": ["The last proof of payment cannot be removed while this payment method is selected. Keep a message or a document on the receipt."]
  }
}
```

Swapping one for the other in a single request (`"pop_upload_id": 41, "pop_message": null`) is
fine: the check runs on the request as a whole, not field by field. So the "remove" action on an
`either` method is offered only while the other half is present.

**Deciding whether to offer "remove".** The receipt's `payment_method` carries
`proof_of_payment_type`, so nothing else has to be fetched:

```json
"payment_method": { "id": 1, "name": "Cheque Deposit Slip",
                    "has_cancellation_fee": true, "proof_of_payment_type": "document" }
```

**PATCH is not partial here.** `transaction_number`, `transaction_date` and `payment_method_id`
are required on every update, so a request carrying only `pop_upload_id` returns `422` naming all
three. Send the receipt's current values alongside the proof.

**`permissions.update` is the gate**, and it has no status condition today — a cancelled receipt
still reports `update: true` and still accepts an edit. That is unlike `delete` (cancelled only),
`dispute` and `reallocate`, which all check status. Treat it as the gate, but do not expect it to
close on a cancelled or disputed receipt.

**Uploading is unchanged** — `POST /api/v1/app/{company}/settings/file-management/uploads` first,
then send the returned id as `pop_upload_id`. The previous upload is left in file management rather
than deleted, so an upload that several records point at is never pulled out from under them. To
discard it, call the file management delete endpoint separately.

### Reading the proof back

Receipt responses — list, show, create and update, on both the app and tenant sides — carry the
upload itself alongside its id:

```json
{
  "pop_upload_id": 41,
  "pop_upload": {
    "id": 41,
    "file_name": "deposit-slip.pdf",
    "source_url": "https://…",
    "thumbnail_url": "https://…",
    "type": "application/pdf",
    "extension": "pdf",
    "size": 20481
  },
  "pop_message": "STK ref QK12345678"
}
```

`pop_upload` is `null` when no document is attached — the key is always present on list and show
responses, so read it rather than testing for its existence.

## Proof of Payment

`PUT|PATCH /api/v1/app/{company}/property-management/lease-management/receipts/{receipt}/proof-of-payment`

Attach, replace or remove a receipt's proof of payment on its own.

```json
{ "pop_upload_id": 4821, "pop_message": "MPESA HI4949494P" }
```

**Use this, not Update Receipt, for the proof-of-payment panel.** Editing a *pending* receipt
goes through the full create payload — property, payer, amount, allocations — so a panel
holding only these two fields is refused with `The property field is required` and four
more. A receipt sent back for changes stays `pending`, which is exactly when that panel is
most needed.

Three states, as on the receipt's own edit:

| Payload | Effect |
|---|---|
| key absent | keeps what is attached |
| `"pop_upload_id": null` | removes it |
| `"pop_upload_id": 4821` | replaces it |

**Saving resubmits a returned receipt** to the reviewer who asked for the change, the same
way a full edit does. Attaching the corrected document *is* the answer to the reviewer, so
it needs no second step.

Nothing else on the receipt is touched — amount, allocations and transaction number are left
exactly as they were. Only `pop_upload_id` and `pop_message` are read: a `payment_method_id` or
`transaction_number` sent here is ignored. Changing the payment method means changing the
transaction number with it, so both stay on Update Receipt.

The payment method still decides what the proof must contain. A method requiring a document
or a reference refuses to have it removed, with *"cannot be removed while this payment
method is selected. Send a replacement instead."* The method itself is not editable here.

Needs `update-facility-receipt`, or to be the receipt's creator, or to hold a step that
opened the field, or to be the person it was returned to — the same rule as Update Receipt.

## Cancel Receipt

`PATCH /api/v1/app/{company}/property-management/lease-management/receipts/{receipt}/cancel`

The body is optional:

```json
{
  "charge_cancellation_fee": true,
  "reason": "Cheque bounced"
}
```

`reason` is optional and only used when a fee is charged. It goes on the fee invoice's notes.

### Cancellation fee

When `charge_cancellation_fee` is `true` and the receipt's payment method has
`has_cancellation_fee`, the tenant is invoiced a fee on the lease of one of the receipt's
invoices. The amount is taken from the first of these that is set:

1. `cancellation_fee` on the receipt's receiving bank account for that payment method
   (Settings > Finance > Bank Accounts)
2. `default_cancellation_fee` on the payment method itself

If neither is set, the call returns `422` on `charge_cancellation_fee` and nothing is cancelled:

```json
{
  "message": "Cheque Deposit Slip cancellation fee must be set on the account in \"Settings > Finance > Bank Accounts\".",
  "errors": {
    "charge_cancellation_fee": ["Cheque Deposit Slip cancellation fee must be set on the account in \"Settings > Finance > Bank Accounts\"."]
  }
}
```

For a payment method without `has_cancellation_fee`, the flag is ignored and no fee is charged.

Possible status values returned by API resource:

- `pending` (`secondary`)
- `confirmed` (`success`)
- `cancelled` (`danger`)
- `rejected` (`danger`) — the approval chain was rejected. The receipt was never posted, so it
  settles no invoice and counts in no report. It can only be viewed or deleted (delete removes it
  outright); `update`, `cancel`, `dispute`, `reallocate` and every other `permissions` flag
  except `view`/`delete` are `false`, and the endpoints refuse it — including approving it
  through reconciliation

`cancelled` is only ever set by this endpoint now; a rejection never produces it.

## Reallocate Receipt

Moves a confirmed receipt's money between the tenant's invoices, leaving the receipt and its
cashbook entry untouched. Two endpoints (`GET` to load the screen, `PUT` to commit), the
`Reallocate` item in the receipt row/header split button, and the reallocation modal are
specified in full in:

`docs/frontend/app/property-management/lease-management/receipt-reallocation.md`

Show the action only when `receipt.permissions.reallocate === true`.

## Applying unallocated money

Money that is on the tenant's account but not on an invoice line is not remittable: remittance
is built from lease collections, and those only exist once money is allocated to an invoice. A
console command finds that money and puts it where it belongs. There is no endpoint; it is run
by hand on the server.

```bash
php artisan receipts:apply-unallocated            # report only, nothing changes
php artisan receipts:apply-unallocated --apply    # post it
php artisan receipts:apply-unallocated --company=3 --receipt=2501 --receipt=2502
```

It works through three kinds of money, in this order, lease by lease:

1. **Overpaid invoices.** An invoice whose `balance` is below zero while a newer or older invoice
   on the same lease is still open (the overpayment arrived after the next invoice was
   processed, so nothing carried it). The overpaid part moves onto the open invoices, oldest
   first, exactly as it would have when the next invoice was processed.
2. **Unallocated receipt money.** A confirmed receipt with a `balance` above zero. It goes onto
   the tenant's open invoices in the receipt's property, oldest first, each up to its
   `available_balance`. What is left overpays the lease's latest invoice and carries forward
   from there. When the tenant has more than one lease in the property, that leftover is not
   placed (there is no telling which lease it belongs to) and the report says so.
   - The full amount becomes lease collections, so it is remittable, also on a receipt that
     was remitted before (the unallocated part was never in that remittance).
   - The receipt's "Unallocated Receipt #…" statement line is replaced by a credit against the
     lease, on the same date, so the tenant is not credited twice. Anything left unplaced keeps
     its unallocated line, for the smaller amount.
3. **Credit notes not fully applied** (`unapplied`, `partially applied`). They are drawn down
   against the lease's open invoices, oldest first, the same way a newly processed invoice
   draws them down.

The report lists one row per item. A report run changes nothing.

| Column | Meaning |
|---|---|
| Amount | what is found: the overpaid part, the unallocated receipt balance, or the unspent credit |
| Would apply / Applied | how much of it goes onto open invoices |
| To invoices | each receiving invoice with its amount, e.g. `#217916: 1446.84, #217920: 300` (receipt rows in the receipt currency, the rest in the lease currency) |
| Left | what no open invoice can take |
| Left goes to | where `Left` stays and what happens to it: an overpayment stays on its invoice and an unspent credit on its credit note, each carrying to the lease's next invoice when it is processed (with the lease's next due date); unallocated receipt money stays on the receipt |
| Note | why something was not placed, or, for receipt money that overpays the lease's latest invoice, that it carries on from there |

## Change Receipt Date (Backdate)

`PATCH /api/v1/app/{company}/property-management/lease-management/receipts/{receipt}/backdate`

Moves a confirmed receipt to an earlier date. Show a **Change Date** item in the receipt row
actions and in the view page's split-button dropdown, only when
`receipt.permissions.backdate === true`. The flag is already `false` for receipts that are not
confirmed, are reversed, or are included in a remittance.

Request body:

| Field | Required | Type | Notes |
|---|---|---|---|
| `new_receipt_date` | Yes | string (`YYYY-MM-DD`) | Today or earlier. Cap the date picker at today. |

```json
{ "new_receipt_date": "2026-08-15" }
```

Behaviour:

- The receipt's `created_at`, its tenant statement lines, its cashbook entry date and its lease
  collections move to the new date. `transaction_date` is not changed.
- If a non-advance landlord remittance for the property covers the chosen month, the request is
  refused. The user has to pick a date in a different month.

Success (`200`) returns `{ data: { message, receipt } }`. Upsert the receipt and refresh the
tenant statement views.

Errors:

- `422` `errors.new_receipt_date`: the date is missing, invalid, in the future, or in a month that
  has already been remitted. Show the message under the date picker.
- `422` `errors.receipt`: the receipt was remitted after the screen was loaded. Show it as a toast
  and close the modal.
- `403`: the user lacks `backdate-facility-receipt`, or the receipt is no longer eligible.

Backend reference: `docs/backend/api/v1/app/receipt-backdate.md`.

## Delete Receipt

`DELETE /api/v1/app/{company}/property-management/lease-management/receipts/{receipt}`

A confirmed receipt that has not been remitted is cancelled first, then deleted. When the receipt
ends up `cancelled` (it was already cancelled, or it was cancelled by this call), the delete also
removes everything it left behind:

- its tenant statement lines, both the confirmed credit lines and the cancelled reversal lines;
- any lease collections still tied to its allocations, and the allocations themselves.

Invoice and invoice item balances are not touched again, because the cancellation already
reversed them. A remitted receipt that was reversed stays `confirmed` with `is_reversed: true`,
and its records, including the mirror receipt, are kept because the remittance depends on them.

**Which receipts may be deleted.** `permissions.delete` is `true` only on a `pending`, `cancelled`
or `rejected` receipt, for holders of `delete-facility-receipt` or the receipt's creator. A
confirmed receipt has to be cancelled first, through the cancel endpoint.

A **pending** receipt (awaiting approval) has posted nothing, so deleting it is like deleting a
pending invoice: the receipt goes together with its pending allocations, its approval chain and
the approvers' pending tasks. No invoice balance moves. The amounts it held on invoices are
released at once (see [Invoices → Amounts held](./invoices.md#amounts-held-by-documents-awaiting-approval)).

## Dispute Receipt

`POST /api/v1/app/{company}/property-management/lease-management/receipts/{receipt}/dispute`

**`permissions.dispute` says whether this receipt can be disputed at all**, and it follows the
receipt's status. Read it per record.

| Status | `permissions.dispute` |
| --- | --- |
| `confirmed` | `true` |
| `pending`, `cancelled`, `rejected` | `false` |

The endpoint enforces the same rule, so a `false` flag and a `422` keyed `receipt` are the same
decision reached twice.

Request body:

| Field | Required | Type | Notes |
|---|---|---|---|
| `details` | Yes | string | Dispute description |
| `ticket_upload_id` | No | integer | Optional attachment upload id (`uploads.id`) |

## Withdraw Receipt Dispute

`DELETE /api/v1/app/{company}/property-management/lease-management/receipts/{receipt}/dispute`

## Public Receipt Page

Every receipt has a public link that anyone holding it can open without signing in, for example
from a payment confirmation SMS or email. Receipt responses carry it as `public_url`:

```json
{
  "id": 2501,
  "public_url": "https://api.example.com/receipts/Yx3...40 random chars"
}
```

The link ends in a random 40-character token, not the receipt id, so receipt numbers cannot be
guessed. The token is issued when the receipt is created; receipts that existed before this
feature were given one by the migration.

These are web pages served by the backend, not JSON endpoints:

| Route | Returns |
|---|---|
| `GET /receipts/{token}` | HTML page. The left column shows the receipt number, status, receipt date, payment method, reference, currency and who served the payment; the payer (KRA PIN, unit, phone, email), the landlord and property, and the bank account the money was paid into (left out when the receipt has none); the payment itself; the invoices it was allocated to (invoice total, amount allocated, invoice balance); and, when the allocation was split by charge, the distribution by component with its total, plus a footer with Amount received and Allocated. Notes on the receipt show in their own card. The right column shows the amount received and a **Download** button |
| `GET /receipts/{token}/pdf` | The receipt as a PDF (`inline`, `RCP2501.pdf`), printed with the company's default receipt template that the receipt's property can use, else its newest active receipt template |

Rules:

- An unknown token returns `404`. So does a `pending` receipt, since it has not been confirmed
  yet, and a `rejected` one, which never will be.
- A `cancelled` receipt is still shown, with its **Cancelled** badge.
- **Download** is hidden, and `/pdf` returns `404`, when the company has no active receipt
  template the receipt's property can use.
- Both routes are limited to 60 requests a minute per IP, and are marked `noindex`.

## Frontend Error Handling

Apply shared rules in `docs/frontend/app/README.md`.
