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
| `pop_message` | Conditional | string | Proof of payment reference. Required when the payment method's `proof_of_payment_type` is `message` or `both` |
| `pop_upload_id` | Conditional | integer | Must exist in `uploads.id`. Required when the payment method's `proof_of_payment_type` is `document` or `both` |

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
    "transaction_number": "RCPT-1005",
    "amount": 1102,
    "allocated": 1102,
    "balance": 0,
    "status": { "value": "confirmed", "color": "success" }
  }
}
```

## Update Receipt

`PUT/PATCH /api/v1/app/{company}/property-management/lease-management/receipts/{receipt}`

Update request body:

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
one of `none`, `message`, `document` or `both`.

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

## Cancel Receipt

`PATCH /api/v1/app/{company}/property-management/lease-management/receipts/{receipt}/cancel`

No request body required.

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

## Dispute Receipt

`POST /api/v1/app/{company}/property-management/lease-management/receipts/{receipt}/dispute`

Request body:

| Field | Required | Type | Notes |
|---|---|---|---|
| `details` | Yes | string | Dispute description |
| `ticket_upload_id` | No | integer | Optional attachment upload id (`uploads.id`) |

## Withdraw Receipt Dispute

`DELETE /api/v1/app/{company}/property-management/lease-management/receipts/{receipt}/dispute`

## Frontend Error Handling

Apply shared rules in `docs/frontend/app/README.md`.
