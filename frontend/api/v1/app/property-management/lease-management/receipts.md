# Receipts API

Domain: `Property Management > Lease Management`

Base route:

`/api/v1/app/{company}/property-management/lease-management/receipts`

## Endpoints

- `GET /receipts`
- `GET /receipts/deadline` — where the monthly receipting deadline stands (row 166)
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
  statement and the cashbook line's `transaction_at`. It never changes on approval. Label it
  **Payment date**: its validation messages name it that way on create and on edit ("The payment
  date field is required.", "The payment date field must be a date before or equal to now."), and
  the printed receipt shows it as "Date".
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

### The remittance holding a receipt

The list, the receipt itself (`GET /receipts/{receipt}`) and the update response carry `remittance`:
the landlord remittance this receipt was paid out in.

```json
"remittance": {
  "id": 8,
  "reference": "RE-008",
  "status": { "value": "paid", "color": "success" },
  "permissions": { "view": true }
}
```

- `null` until the receipt is remitted. A receipt whose remittance was cancelled or rejected is back
  to unremitted, so it is `null` again. An advance remittance holds no receipts, so it never shows here.
- `reference` is how the remittance screens name it (`RE-008`, `RE-123`). Show it in the
  **Remittance** column of the list and in the receipt's Payment Information card, and `-` when
  `remittance` is `null`.
- Link it to the remittance page (`/remittances/{id}`) only when `permissions.view` is `true`. That is
  the remittance's own view rule: `view-facility-remittance` and an allocation to the property.
  Otherwise show the reference as plain text.
- Read in one query for the whole page, not one per row.

## Create Receipt

`POST /api/v1/app/{company}/property-management/lease-management/receipts`

Request body:

| Field | Required | Type | Allowed Values / Notes |
|---|---|---|---|
| `transaction_number` | Yes | string | The payment's reference number. Labelled "Reference number" on the receipt screens, and its validation messages say so on create and update: `The reference number field is required.`, `The reference number does not match the required format for the selected payment method.` (row 210). The field name is unchanged |
| `receiving_account_id` | Yes | integer | Must exist in `bank_accounts.id` |
| `transaction_date` | Yes | date (`YYYY-MM-DD`) | The payment date. Today or earlier |
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
| `transaction_number` | The reference doesn't match the payment method's `transaction_number_regex`, and the caller lacks `override-payment-reference-format` (see [Overriding the reference format](#overriding-the-reference-format-and-proof-of-payment)) | The reference number does not match the required format for the selected payment method. |
| `receipting` | Receipting has closed for the month and the caller lacks `receipt-after-deadline` (see [Monthly receipting deadline](#monthly-receipting-deadline)) | Receipting for October 2026 closed on Thu 29 Oct at 17:00. It reopens on Sun 1 Nov at 07:00. |

### Duplicate payments: warning and confirmation

A suspected duplicate is a **warning the user confirms**, not a refusal (PROP-284). It is checked on
the backend on every create and on every edit that changes one of the four fields below, including
a receipt created from a bank payment on the reconciliation screen.

**The matching rule.** A receipt is the same payment as another when the **reference
(`transaction_number`), amount, property (`facility_id`) and tenant (`paying_user_id`)** all match.
The reference is compared without case or spaces (`qjk3 x7l2mp` is `QJK3X7L2MP`); the amount must be
exact. Only **live** receipts count, pending or confirmed: a cancelled, rejected or reversed receipt
no longer records the payment, so it can be entered again. An edit that leaves the four alone is
never checked, so a receipt that already duplicated another before this rule stays editable.

**1. The first request: `409`, nothing saved.** Every other check (fields, permissions, the monthly
deadline, proof of payment, allocations) runs first; a request failing one of those gets its usual
`422`. A duplicate is answered:

```json
409
{
  "message": "Possible duplicate payment: receipt RCP0412 already records reference QJK3X7L2MP for 25,000.00 from this tenant on this property.",
  "code": "duplicate_receipt",
  "reason": null,
  "errors": { "transaction_number": ["Possible duplicate payment: receipt RCP0412 already records ..."] },
  "duplicate_receipt": {
    "id": 412,
    "number": "RCP0412",
    "transaction_number": "QJK3X7L2MP",
    "amount": 25000,
    "payment_date": "2026-11-02",
    "status": "confirmed",
    "tenant": { "id": 77, "name": "Achieng Atieno" },
    "property": { "id": 61, "name": "Kahawa Side Tower" }
  },
  "duplicate_token": "eyJpdiI6...",
  "duplicate_token_expires_at": "2026-11-06T10:25:00+03:00"
}
```

Open the warning on `code: "duplicate_receipt"` (or `"duplicate_confirmation_invalid"`, below), not
on the status alone. Show the existing receipt's number, reference, amount, payment date, tenant and
property, with **Cancel** and **Yes, Receipt duplicate**. Cancel sends nothing; keep the form as it
is. `duplicate_receipt` names the **newest** matching receipt.

**2. The confirmed retry.** Send the same body again with two more fields:

| Field | Type | Notes |
|---|---|---|
| `allow_duplicate` | boolean | `true` |
| `duplicate_token` | string | `duplicate_token` from the `409`, as given |

Both are needed. Either one alone is no confirmation, and the request is checked as a first one (a
fresh `409`). The receipt is then saved through the normal flow, so it can still fail any other
check with its usual `422`, and then nothing is saved and the token stays usable.

The token:

- lasts **10 minutes** from the `409` that issued it;
- covers only the user it was issued to, this property, tenant, amount and reference, and the
  receipt it matched. On an edit it also covers only the receipt being edited, so a token from the
  create form does not confirm an edit, and the reverse;
- **saves one receipt only.** The saved receipt keeps the token's hash under a unique index, and
  identical requests from the same tenant run one after another, so a double click or a retried
  request saves nothing more.

Send the retry once, and disable the button while either request is in flight.

**3. A confirmation that does not cover it: `409` again.** Same body as step 1, with
`code: "duplicate_confirmation_invalid"`, a `reason`, the reason first in `message`, and a fresh
`duplicate_token`:

| `reason` | When | Message starts |
|---|---|---|
| `used` | The token already saved a receipt. `duplicate_receipt` is now the receipt it saved | That duplicate confirmation was already used. |
| `expired` | More than 10 minutes since the `409` | That duplicate confirmation has expired. |
| `invalid` | Altered, another user's, or for a different receipt or edit | That duplicate confirmation does not match this receipt. |

Show the warning again; confirming it sends the fresh token.

**Audit.** A receipt saved this way records which receipt it duplicates, who confirmed it and when:
`duplicate_of_receipt_id`, `duplicate_confirmed_by_id`, `duplicate_confirmed_at` (and the token's
hash). They land in the receipt's audit trail with the confirming user as the actor. Receipt
responses show it:

```json
"duplicate_of": { "id": 412, "number": "RCP0412", "confirmed_at": "2026-11-06T07:16:02.000000Z", "confirmed_by": { "id": 5, "name": "Jane Wanjiku" } }
```

`duplicate_of` is `null` on any other receipt; `confirmed_by` comes on `GET /receipts/{receipt}`.

**From reconciliation.** `POST /receipts` with `bank_payment_id` runs the same check and takes the
same confirmation; on success the bank payment is receipted as usual and the screen returns to
reconciliation. The automatic matcher never confirms a duplicate: it leaves the payment as a
suggestion with the duplicate message as its reason.

### Overriding the reference format and proof of payment

Each payment method can set a reference pattern (`transaction_number_regex`, e.g. ten letters and
digits for M-Pesa) and a proof of payment (`proof_of_payment_type`). A user with the
**`override-payment-reference-format`** permission is held to neither (row 205):

- **Any reference** is accepted on create and on edit, pending or confirmed.
- **Proof of payment is optional.** They may attach one, leave it out, or remove it later, including
  through [Proof of Payment](#proof-of-payment-1).
- **The duplicate check still applies**, as above.
- There is no extra field to send and nothing to confirm: holding the permission is the override.
  The receipt records who created it, as for any receipt.

**Payment methods for the form.** Load them with `?for=payment`:

`GET /api/v1/settings/accounting/payment-methods?for=payment`

For a holder, every method comes back with `transaction_number_regex: null`, so the form doesn't
check the reference. Each method also carries `can_override_reference_format` (`true` for a holder,
`false` otherwise). `proof_of_payment_type` stays as configured: when the flag is `true`, keep
offering the proof but show it as optional, since a holder may still attach one. Without
`?for=payment` the methods come back as configured and without the flag. The settings screen must not send `?for=payment`: it edits these methods, and
saving the blanks back would remove the pattern for everyone.

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

An edit that changes the reference, amount, property or tenant into another receipt's is a
suspected duplicate, with the same `409`, `allow_duplicate` and `duplicate_token` as on create (see
[Duplicate payments](#duplicate-payments-warning-and-confirmation)). The token from an edit's `409`
confirms only that edit.

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
| `transaction_date` | Yes | date (`YYYY-MM-DD`) | The payment date. Today or earlier |
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

**Disputing again.** Only an open dispute holds the next one back: `disputed_at` is set, or the
linked ticket (`dispute_ticket_id`) is not `closed`. Until then the endpoint answers `422` keyed
`receipt`, "This receipt has already been disputed. See ticket#{id}", naming that ticket, with
`open_ticket: { "id": … }` beside `errors` for a link to it. Once the ticket is closed, the receipt
can be disputed again: a new ticket is raised and `dispute_ticket_id` points at it. The closed
ticket stays in the tickets list with its resolution, as the history of the first dispute.
`permissions.dispute` does not look at an open dispute, so keep the button and show the `422`
message.

The list and the receipt itself carry `dispute_ticket` (`id`, `title`, `status`) whenever one is
linked, open or closed. The receipt is disputed while `disputed_at.raw` is set or
`dispute_ticket.status.value` is not `closed`; otherwise a linked ticket is the settled history of
the last dispute. `disputed_at` is always sent, as `raw: null` when nothing is open.

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

## Monthly receipting deadline

Row 166. When a company turns it on, **new receipts close on the second-last working day of each
month** at the closing time, and **reopen on the 1st of the next month** at the reopening time. It
all happens on its own; nobody closes or opens receipting by hand.

- **Working days** are Monday to Friday, as for every deadline in the app (no holiday calendar).
  October 2026 closes on Thu 29 Oct; November 2026 on Fri 27 Nov.
- **Only creating receipts** is closed. Editing, backdating, reallocating, cancelling and proof of
  payment are unaffected.
- **Who still receipts while it is closed:** anyone with the `receipt-after-deadline` permission (the
  receivables accountants). Everyone else gets `422` on `receipting` from `POST /receipts`.
- **A day before it closes**, everyone in the company who can create receipts gets an in-app
  notification ("Receipting closes soon"), and the receipt screens start the countdown.

**Settings** (company settings, Finance group, all dropdowns):

| Key | Values | Default |
| --- | --- | --- |
| `receipting_deadline_enabled` | `yes` / `no` | `no` |
| `receipting_deadline_close_time` | `12:00` … `23:00` | `17:00` |
| `receipting_deadline_reopen_time` | `06:00` … `09:00` | `07:00` |

### Where it stands

`GET /receipts/deadline` answers for the caller. The same object comes back as `receipting` on
`GET /receipts` (next to `data`), so the list page needs no extra call.

```json
{
  "data": {
    "receipting": {
      "enabled": true,
      "state": "closing_soon",
      "period": "October 2026",
      "closes_at": "2026-10-29T17:00:00+03:00",
      "countdown_from": "2026-10-28T17:00:00+03:00",
      "reopens_at": "2026-11-01T07:00:00+03:00",
      "seconds_until_close": 28800,
      "seconds_until_reopen": null,
      "message": "The receipting deadline is fast approaching: receipting for October 2026 closes on Thu 29 Oct at 17:00.",
      "exempt": false,
      "can_receipt": true
    }
  }
}
```

| `state` | When | Show |
| --- | --- | --- |
| `open` | Not within a day of the deadline (or the deadline is off: `enabled: false`) | Nothing special |
| `closing_soon` | From `countdown_from` (a day before) until `closes_at` | The **New Receipt** button in a warning colour with a countdown to `closes_at`, and a banner at the top of the form with `message` ("the deadline is fast approaching") and the countdown |
| `closed` | From `closes_at` until `reopens_at` (the 1st, at the reopening time) | When `can_receipt` is false: no New Receipt button on the list; show `message` ("Receipting for October 2026 closed… It reopens on Sun 1 Nov at 07:00.") instead, and don't let the form submit |

- `can_receipt` is what decides the button: `true` while open, and while closed for an `exempt` user
  (`receipt-after-deadline`). An exempt user still sees `state: closed`, so the screen can say they
  are receipting after the deadline.
- Count down from `closes_at` (or to `reopens_at` when closed) with the device clock;
  `seconds_until_close` / `seconds_until_reopen` are the server's figure at the time of the request,
  should the device clock be off. Re-fetch when a countdown reaches zero: the state changes on its
  own.
- The notification carries `period`, `closes_at` and `reopens_at` in its data.

### Testing before the deadline (not production)

Outside production, `GET /receipts/deadline`, `GET /receipts` and `POST /receipts` take
`?as_of=2026-10-29T09:00` (Nairobi time) and answer as if it were that moment, so every state can be
seen now. Production ignores `as_of`, as does a value that is not a date. Useful moments for
October 2026:

| `as_of` | State |
| --- | --- |
| `2026-10-28T16:59` | `open` |
| `2026-10-28T17:00` | `closing_soon` (countdown starts) |
| `2026-10-30T09:00` | `closed` |
| `2026-11-01T06:59` | `closed` (October's closure still holds) |
| `2026-11-01T07:00` | `open` (November) |

The notice can be fired for a test moment with
`php artisan receipts:notify-receipting-deadline --as-of=2026-10-28T18:00`.

## Frontend Error Handling

Apply shared rules in `docs/frontend/app/README.md`.
