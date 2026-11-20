# Payment Vouchers API

Domain: `Property Management > Finance`

Base route:

`/api/v1/app/{company}/property-management/finance/payment-vouchers`

## Endpoints

- `GET /payment-vouchers`
- `GET /payment-vouchers/create` (options preview; `payment_voucher_id` adds a voucher's own payables for its edit form)
- `POST /payment-vouchers`
- `GET /payment-vouchers/{paymentVoucher}`
- `PUT /payment-vouchers/{paymentVoucher}` — edit while `pending` (see [Edit a payment voucher](#edit-a-payment-voucher))
- `DELETE /payment-vouchers/{paymentVoucher}`
- `PATCH /payment-vouchers/{paymentVoucher}/cancel`
- `PATCH /payment-vouchers/{paymentVoucher}/release`
- `PATCH /payment-vouchers/{paymentVoucher}/reverse` — back to `pending` after release (see [Reverse a released voucher](#reverse-a-released-voucher))
- `PUT /payment-vouchers/{paymentVoucher}/pay` — the bank's advice; on a landlord voucher it sends the [remittance pack](#remittance-pack-landlord-vouchers)
- `POST /payment-vouchers/{paymentVoucher}/remittance-packs/{remittancePack}/retry` — send a failed pack again

### Order of the payable items

`GET /payment-vouchers/create` returns its `options` **newest first**, by the date each option carries
in `created_at` (`raw`, `formatted`, `diff`): bills for a supplier voucher, remittances for a landlord
voucher. Items with the same `created_at` come newer `id` first, so the order is the same on every
call. The sort is part of the query, so it holds under every filter the endpoint takes, and on the
edit form the voucher's own items sit among the rest by their dates. The endpoint is not paginated
and has no search or `sort` parameter; it returns every eligible item in this one order.

```
GET /payment-vouchers/create?payable_as=vendor&payable_user_id=289
→ options: [ { "id": 44, "created_at": { "formatted": "06 Nov, 2026", … } },
             { "id": 43, "created_at": { "formatted": "05 Nov, 2026", … } }, … ]
```

## List Payment Vouchers

`GET /api/v1/app/{company}/property-management/finance/payment-vouchers`

Supported query params:

- Filters:
  - `filter[search]` (Scout-backed search; supports CSV IDs and payment advice numbers)
  - `filter[payable_user_id]`, `filter[credit_bank_account_id]`
  - `filter[payable_as]`, `filter[status]`
  - `filter[created_at]`, `filter[released_at]`, `filter[paid_at]`

**Ids, `status` and `type` match exactly.** They were partial until 2026-09-29, so
`filter[status]=paid` also returned `unpaid` and `partially-paid`, and `filter[facility_id]=6`
returned rows for facility 60. Send the whole value. Date filters are still partial, so
`filter[created_at]=2026-09` means that month.
Every date filter also takes a range, `filter[created_between]=2026-09-01,2026-09-30`; see
[List filters](../../../list-filters.md) for each one's name.
- Sort:
  - `sort=id,amount,paid_at,released_at,created_at,updated_at`
- Include:
  - `include=paymentVoucherItems`
- Pagination:
  - `per_page`, `page`

Enum filter options:

- `filter[payable_as]`: `vendor`, `landlord`, `tenant`, `other` (from `FacilityPaymentVoucherData`)
- `filter[status]`: `pending`, `released`, `paid`, `cancelled` (from `PaymentVoucherStatus` enum)

Sample list response (`FacilityPaymentVoucherResource`). `public_url` is the voucher's public payment advice page; see [Public Payment Voucher Page](#public-payment-voucher-page).

```json
{
  "data": [
    {
      "id": 801,
      "public_url": "https://api.example.com/payment-vouchers/Yx3...40 random chars",
      "payable_as": "vendor",
      "notes": "Batch payout",
      "amount": "2500.00",
      "payment_advice_number": "ADV-801",
      "paid_at": {
        "raw": "2026-05-08T00:00:00.000000Z",
        "formatted": "08 May, 2026",
        "diff": "1 day ago"
      },
      "released_at": {
        "raw": "2026-05-07T00:00:00.000000Z",
        "formatted": "07 May, 2026",
        "diff": "2 days ago"
      },
      "status": { "value": "released", "color": "info" },
      "payable_user": { "id": 7, "name": "Acme Vendor", "phone": null, "email": "accounts@acme.example.com" },
      "credit_bank_account": { "id": 13, "account_name": "Operations Account" },
      "currency": { "id": 1, "code": "KES" },
      "items": [
        { "id": 1, "amount": "1000.00" }
      ],
      "created": {
        "raw": "2026-05-09T10:15:00.000000Z",
        "formatted": "09 May, 2026",
        "diff": "moments ago"
      },
      "updated": {
        "raw": "2026-05-09T10:15:00.000000Z",
        "formatted": "09 May, 2026",
        "diff": "moments ago"
      }
    }
  ]
}
```

## Options for the create form

`GET /payment-vouchers/create` lists what the payee can be paid for. Not paginated or sorted.

| Param | Required | Meaning |
|---|---|---|
| `payable_as` | yes | `vendor` (supplier) or `landlord` |
| `payable_user_id` | yes | the payee |
| `facility_id` | no | one property's items only |
| `payment_voucher_id` | no | a voucher being edited: its own items are offered too |
| `expense_type_id` | no, suppliers only | the supplier's bills for that **service** (expense type) |
| `landlord_id` | no, suppliers only | the supplier's bills on that **landlord's** properties |

For a supplier the options are their open bills (`unpaid` or `partially paid`). `expense_type_id`
and `landlord_id` work alone or together; leave both out for the unfiltered list. Each bill option
carries what they match on:

```json
{
  "id": 42, "invoice_number": "INV-0042", "facility": "Kahawa Side Tower",
  "expense_type": { "id": 11, "name": "Cleaning" },
  "landlord": { "id": 97, "name": "Clementine Ndibo Holdings" },
  "amount": 1000, "tax": 0, "total": 1000, "balance": 1000, "payable_amount": 1000
}
```

`expense_type` is `null` on a bill with no expense type, which only the unfiltered list shows.
No match is an empty `options` list, not an error. The Service dropdown can be filled from the
company's expense types, and the Landlord dropdown from its landlords.

```
GET …/payment-vouchers/create?payable_as=vendor&payable_user_id=289&expense_type_id=11
GET …/payment-vouchers/create?payable_as=vendor&payable_user_id=289&landlord_id=97
GET …/payment-vouchers/create?payable_as=vendor&payable_user_id=289&expense_type_id=11&landlord_id=97
```

| `422` on | Message |
|---|---|
| `expense_type_id` | *The selected service does not exist.* (unknown, or another company's) / *The expense type id field must be an integer.* |
| `landlord_id` | *The selected landlord has no property in this company.* / *The landlord id field must be an integer.* |
| either, with `payable_as` other than `vendor` | *The service filter applies to supplier vouchers only.* / *The landlord filter applies to supplier vouchers only.* |

### Payee contact

`payable_user` is the payee: the user the voucher pays (supplier, landlord, tenant). On the list and
on `GET /payment-vouchers/{paymentVoucher}` it always carries **`phone`** and **`email`**, read from
that user. Each is `null` when the payee has none; show "-". Elsewhere a user's missing fields are
left out of the response, but on a voucher these two are always present.

```json
"payable_user": {
  "id": 289,
  "name": "Sparkle Cleaning Services Ltd",
  "email": "accounts@sparkle.example.com",
  "phone": null,
  "profile_photo_url": "https://ui-avatars.com/api/?name=S+C+S+L&color=7F9CF5&background=EBF4FF"
}
```

## Create payload (`PaymentVoucherData`)

| Field | Required | Type | Allowed Values / Notes |
|---|---|---|---|
| `payable_user_id` | Yes | integer | Must exist in `users.id` |
| `payable_as` | Yes | string | `vendor`, `landlord`, `tenant`, `other` |
| `credit_bank_account_id` | Yes | integer | Must exist in `bank_accounts.id` |
| `items` | Yes | array | At least 1 item |
| `currency_id` | No | integer | Must exist in `currencies.id` |

`items[]` for create:

| Field | Required | Type | Allowed Values / Notes |
|---|---|---|---|
| `payable_id` | Yes | integer | Expense/Remittance id depending on `payable_as` |
| `amount` | Yes | number | Must not exceed payable balance/remittable amount |

### Payment method and reference

Load the payment methods for the voucher form with `GET /api/v1/settings/accounting/payment-methods?for=payment`.
The voucher's `transaction_number` is checked against the method's `transaction_number_regex` on the
form only. For a user with the `override-payment-reference-format` permission, `?for=payment`
returns `transaction_number_regex: null` and `can_override_reference_format: true`, so the form
takes any reference (row 205). See the settings doc under **Payment methods on a receipt or voucher form**.

## Edit a payment voucher

`PUT /api/v1/app/{company}/property-management/finance/payment-vouchers/{paymentVoucher}`

Only while the voucher is `pending` — before it is released (row 138). Needs
`update-facility-payment-voucher`, or to be the voucher's creator.

Body: **the same payload as create**, sent whole. The items sent replace the voucher's items:

- Creating a voucher already marks what it pays: a bill's `paid` goes up by the item's amount, a
  remittance becomes `paid`. The edit takes that back for the old items (the bill's `paid` goes down
  again, a remittance goes back to `unpaid`) and applies the new items the same way create does.
- `amount` is checked against the bill's balance **plus what this voucher already pays on that bill**, so
  sending the voucher back unchanged, or raising an item's amount up to the bill's full balance, passes.
- The status stays `pending`. Nothing reaches the vendor statement until the voucher is released.

Response `200`: `{ "data": { "message": "Payment voucher updated successfully.", "payment_voucher": { ... } } }`

Errors:

- `403` — `"Only a pending payment voucher can be edited."`, or no edit right.
- `422` — validation, as on create.

Building the edit form: call the options endpoint with the voucher's id,
`GET /payment-vouchers/create?payable_as=…&payable_user_id=…&payment_voucher_id={id}`. Alongside what
is still open it returns the bills or remittances this voucher already pays - even ones it pays in
full - in the same shape, and every option carries `voucher_amount`:

- `voucher_amount`: what this voucher pays on it, in the voucher's currency; `null` when it is not on
  this voucher. Tick the options where it is set and prefill their amount with it.
- Bills: `payable_amount` includes what this voucher pays on the bill (remaining balance + this
  voucher's payment, in the bill's currency), so it is the most the edited voucher can pay on it.
  `paid` and `balance` are the bill's own figures, with this voucher's payment already in them.
- Remittances: `remittable_amount` as on create; the amount must still equal it.

The voucher's own items are offered only while the query names the voucher's own payee
(`payable_as` and `payable_user_id`). `payment_voucher_id` needs the edit right on that voucher
(`403` otherwise). Without it the endpoint behaves as before.

## Flags, printing and delete

- `permissions.update` is `true` only on a `pending` voucher the user may edit.
- `permissions.delete` is `true` only on a `pending` or `cancelled` voucher, matching what
  `DELETE` accepts.
- `permissions.reverse` is `true` only on a released (`paid`) voucher, for a user who may create
  vouchers. Show a "Reverse to pending" action when it is set.
- A voucher prints in every status, `pending` included, so show the print action on every row. A
  `cancelled` one prints under a CANCELLED watermark. The payee's public page still hides pending
  vouchers with a `404`.
- `permissions.printVoucher` and `permissions.printAdvice` (on list rows and the view) drive the print menu — see
  [Print the voucher or its advice](#print-the-voucher-or-its-advice).

## Print the voucher or its advice

`GET /payment-vouchers/{paymentVoucher}/print?template=voucher|advice`

Returns the PDF itself, ready to open, with no preview step: `200`, `Content-Type: application/pdf`,
`Content-Disposition: inline; filename="PV0005-voucher.pdf"` (or `PV0005-advice.pdf`). The voucher's
data is not changed. As a `GET` it can be fetched as a blob and opened the way other PDFs are.

| `template` | Prints | Template used | Statuses |
|---|---|---|---|
| `voucher` | The internal payment voucher | The company's default payment voucher template | Every status, `pending` included; `cancelled` under a CANCELLED watermark |
| `advice` | The payee's payment advice (a PDF generated from a payment voucher template, not the uploaded advice file) | The template in the `payment_voucher_advice_template_id` setting, else the default payment voucher template — the same PDF the payee downloads from the public page and is emailed on release | Not `pending`; `paid` and `cancelled` print |

Show the menu when `permissions.printVoucher` is `true`. Offer **Print Advice** only when
`permissions.printAdvice` is `true`: it is `false` on a pending voucher, because the advice tells
the payee they have been paid, and the public page and the release email both wait for that too.

Every refusal is JSON with a stable `code` beside the message, so read the body of a failed
blob request and translate by `code`:

| Status | `code` | Message | When |
|---|---|---|---|
| `403` | `print_not_allowed` | *You are not allowed to print this payment voucher.* | The user may not see the voucher (`view-facility-payment-voucher`) or print document templates (`view-document-template`) |
| `422` | `print_template_invalid` | *Choose what to print: voucher or advice.* / *The template must be voucher or advice.* | `template` missing, or any other value |
| `422` | `advice_not_available` | *A payment advice is available once the voucher is released.* | `advice` on a pending voucher |
| `422` | `print_template_missing` | *No active payment voucher template is set up. Add one under Settings > Documents > Templates.* | The company has no active payment voucher template |
| `500` | `print_failed` | *The payment voucher could not be printed. Please try again.* | The PDF could not be generated |

The `422`s also carry `errors.template`; the `403` and `500` carry no `errors`.

```json
{
  "message": "A payment advice is available once the voucher is released.",
  "code": "advice_not_available",
  "errors": { "template": ["A payment advice is available once the voucher is released."] }
}
```

## Reverse a released voucher

`PATCH /payment-vouchers/{paymentVoucher}/reverse` — no body.

Puts a released voucher back to `pending`, undoing what releasing it recorded:

| Released voucher | After reversing |
|---|---|
| `status: paid`, `releasing_user`, `released_at`, `paid_at` set | `status: pending`; the three are cleared |
| A cashbook credit on the paying (debit) bank account | Removed from the [bank account statement](./bank-account-statement.md) |
| Vendor statement debits for the bills it paid | Removed |
| A liability bill: each source vendor's withholding settlement | Removed |

What stays:

- **The bills and remittances it pays keep the amount.** That is applied when the voucher is raised,
  not when it is released, so a pending voucher holds it as before release. To give it back, cancel
  the voucher after reversing it (`PATCH …/cancel`, which only accepts a pending voucher).
- **The payment advice is not sent again**, neither now nor when the voucher is released again:
  the payee already has it.
- The payment details (`payment_advice_number`, the advice upload) stay on the voucher.

Once pending again, the voucher can be edited, released again, cancelled or deleted like any other
pending voucher. The audit trail records who reversed it.

**Who:** anyone with `create-facility-payment-voucher`, the same right that raises a voucher. No
reason is asked for.

Responses:

- `200` — `{ "message": "Payment voucher reversed to pending.", "payment_voucher": { … } }`, the voucher
  as on `GET`, now `pending`.
- `403` — the user may not create vouchers, or the voucher is not released: `"Only a released
  payment voucher can be reversed."`
- `422` on `status` — its cashbook entry is matched in bank reconciliation: `"This payment voucher is
  matched in bank reconciliation. Unmatch its cashbook entry before reversing it."` An entry that
  reconciliation only marked as ignored does not block it, and is removed.

## A bank transfer's voucher

A voucher with `bank_transfer_id` set was raised for a [bank transfer](bank-transfers.md): created
`paid`, payable to the To account's owner, its one item the transfer (`bank_transfer` has the
transfer's `id`, `reference`, `transfer_date` and `status`). It is not released, reversed, edited
or deleted. **Cancelling it removes the transfer's two cashbook lines**, the credit in the From
account and the debit in the To account, and marks the transfer `cancelled`; refused with a `422`
on `status` while either line is matched in bank reconciliation. See
[Bank Transfers](bank-transfers.md#the-transfers-payment-voucher).

## Cancel a pending voucher

`PATCH /payment-vouchers/{paymentVoucher}/cancel` (pending vouchers only, and a bank transfer's
voucher while it is paid: see above).

Gives back what the voucher held. Each bill it paid gets its paid amount reduced and its status
recomputed, and so does the bill's expense. Each **remittance** it paid goes back to `unpaid`.

A remittance's `remittable_amount` is **not** touched. Raising the voucher never reduced it, so
there is nothing to add back. Until 2026-11-04 cancelling added the voucher line's amount to it, so
a cancelled voucher doubled what its remittances said was owed (for example remittance #8123 went
from 162,435.00 to 324,870.00). `php artisan remittances:repair-voucher-inflation` reports the remittances
this inflated, and corrects them with `--apply`.

## Pay payload (`PaymentAdviceData`)

| Field | Required | Type |
|---|---|---|
| `payment_advice_number` | Yes | string |
| `payment_advice_upload_id` | Yes | integer (`uploads.id`) |

The response's `payment_voucher` includes `remittance_packs` (see below).

## Remittance pack (landlord vouchers)

Once the bank's remittance advice is saved on a **landlord** voucher (`PUT …/pay` above), every
remittance the voucher pays gets a **remittance pack** emailed to its landlord, in one email per
remittance, with four PDFs attached:

1. the system **remittance advice** (`REM0044 Remittance Advice.pdf`, the remittance advice template),
2. the **bank's remittance advice** just uploaded (`REM0044 Bank Remittance Advice.pdf`),
3. the **Income and Expenditure** report (`REM0044 Income and Expenditure.pdf`),
4. the **Property Collections** report (`REM0044 Property Collections.pdf`).

Both reports cover the remittance's property over its whole period (`period_from` to `period_to`),
with the I&E report in monthly columns. They are the same PDFs the reports' Export buttons give,
generated as the user who uploaded the advice, so that user must be assigned to the property.
Supplier vouchers send nothing.

**All or nothing.** The pack is built in the background after the upload. If any document is missing
or fails to generate, nothing is sent, and the pack is `failed` with the reason.

**Recipient.** The remittance's landlord, at their default billing address email, or else their
account email. Their billing addresses marked "always copy" are on cc, as on the other document emails.

**Duplicates.** A pack belongs to a remittance and an uploaded file:

- saving the same file again sends nothing;
- replacing the bank advice with a different file sends a new pack for each remittance, carrying the
  new advice; the earlier packs stay listed;
- a pack is sent at most once, however often the job or the retry runs.

### `remittance_packs` on the voucher

On `GET /payment-vouchers/{id}`, the `PUT …/pay` response and the retry response. Newest first; empty
for a supplier voucher or before the advice is uploaded.

```json
"remittance_packs": [
  {
    "id": 12,
    "remittance": { "id": 44, "number": "REM0044" },
    "status": { "value": "sent", "color": "success" },
    "recipient": { "id": 97, "name": "Clementine Ndibo Holdings", "email": "accounts@ndibo.co.ke" },
    "copied_to": ["finance@ndibo.co.ke"],
    "sent_at": { "raw": "2026-11-16T09:41:07.000000Z", "formatted": "16 Nov, 2026 09:41", "diff": "1 minute ago" },
    "failed_at": null,
    "error_code": null,
    "error": null,
    "attempts": 1,
    "can_retry": false
  },
  {
    "id": 13,
    "remittance": { "id": 45, "number": "REM0045" },
    "status": { "value": "failed", "color": "danger" },
    "recipient": null,
    "copied_to": [],
    "sent_at": null,
    "failed_at": { "raw": "2026-11-16T09:41:09.000000Z", "formatted": "16 Nov, 2026 09:41", "diff": "1 minute ago" },
    "error_code": "property_collections_failed",
    "error": "The Property Collections report could not be generated.",
    "attempts": 1,
    "can_retry": true
  }
]
```

| `status.value` | Meaning |
|---|---|
| `processing` | Being built and sent. Refresh the voucher until it changes (every few seconds; usually under a minute). |
| `sent` | The mail server accepted the email. `recipient` and `sent_at` are set. Delivery to the inbox is not tracked. |
| `failed` | Not sent. `error_code` says why (translate it), `error` is the English reason (the fallback). `can_retry` is `true` when the pack is failed **and** the user holds `release-facility-payment-voucher`, the permission the retry checks. |

Why a pack fails (`error_code`, with its `error`). Both are `null` unless the pack is `failed`.

| `error_code` | `error` |
|---|---|
| `template_missing` | `No remittance advice template is set up, so the remittance advice could not be generated.` |
| `advice_failed` | `The remittance advice could not be generated.` |
| `bank_advice_missing` | `The bank's remittance advice file is missing. Upload it again on the payment voucher.` |
| `income_and_expenditure_failed` | `The Income and Expenditure report could not be generated.` |
| `property_collections_failed` | `The Property Collections report could not be generated.` |
| `uploader_not_assigned` | `{user} is not assigned to {property}, so its reports cannot be generated.` The staff user who uploaded or retried, not the landlord. |
| `uploader_missing` | `The user who uploaded the bank advice no longer exists. Retry the pack.` |
| `remittance_missing` | `The remittance or its payment voucher no longer exists.` |
| `no_recipient_email` | `The landlord has no email address.` |
| `mail_failed` | `The email could not be sent: {mail server's reason}` |
| `timed_out` | `The remittance pack took too long to prepare. Try again.` |
| `unexpected_error` | `The remittance pack could not be sent. Try again.` (logged) |

### Retry a failed pack

`POST /payment-vouchers/{paymentVoucher}/remittance-packs/{remittancePack}/retry`, no body.

Puts a `failed` pack back to `processing` and sends it again, with the reports generated as the
user retrying. Same permission as uploading the advice (`release-facility-payment-voucher`).
Response: `{ "message": "The remittance pack is being sent again.", "payment_voucher": { …, "remittance_packs": [...] } }`.

- `403`: the user lacks the permission.
- `404`: the pack does not belong to this voucher.
- `422` on `remittance_pack`: `Only a failed remittance pack can be retried.` (already sent, or
  already being retried by another click).

Each pack's creation, its result, and the advice upload itself are in the audit trail.

## Public Payment Voucher Page

Every payment voucher has a public link that the payee can open without signing in, for example
from the email or SMS that tells them they have been paid. Voucher responses carry it as
`public_url`:

```json
{
  "id": 801,
  "public_url": "https://api.example.com/payment-vouchers/Yx3...40 random chars"
}
```

The link ends in a random 40-character token, not the voucher id, so voucher numbers cannot be
guessed. The token is issued when the voucher is created; vouchers that existed before this
feature were given one by the migration.

These are web pages served by the backend, not JSON endpoints:

| Route | Returns |
|---|---|
| `GET /payment-vouchers/{token}` | HTML page. The left column shows the payment advice: the voucher number, status, voucher date, paid date, payment method, reference, credited account and currency; the payee ("Paid to") and the company ("Paid by"); the items paid (date, property, description and memo, total, one column per withholding tax deducted on the voucher, withheld, advance, payable and paid) with a Total paid footer; who prepared it; and the voucher notes. The right column shows the amount paid and a **Download payment advice** button |
| `GET /payment-vouchers/{token}/pdf` | The voucher as a PDF (`inline`, `PV0801.pdf`), printed with the template chosen in the `payment_voucher_advice_template_id` setting (see [Settings](../settings.md)) |

Rules:

- An unknown token returns `404`. So does a `pending` voucher, since it has not been paid yet.
- A `cancelled` voucher is still shown, with its Cancelled badge.
- **Download** is hidden, and `/pdf` returns `404`, when the company has no usable payment
  voucher template.
- The PDF template is the one named by the `payment_voucher_advice_template_id` setting when it
  is still active, else the company's default payment voucher template, else its newest active
  one. A voucher has no property, so only company-wide templates (no `facility_ids`) apply.
- Both routes are limited to 60 requests a minute per IP, and are marked `noindex`.
