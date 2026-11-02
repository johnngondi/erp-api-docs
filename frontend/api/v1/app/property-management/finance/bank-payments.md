# Bank Payments (parked IPN payments)

Domain: `Property Management > Settings > Finance > Banks`, surfaced in the UI under
`Tenant Management > Billing & Receipting > Bank Payments`.

When the bank sends an Instant Payment Notification (see
[KCB IPN](../../../../../../integrations/kcb-ipn.md)), the notification is **parked** as a bank
payment on the bank account it was credited to. The money is visible in the ERP at once, and the
system then tries to tie it to a receipt:

1. **Park** — every pay-in and pay-out becomes a bank payment with status `unmatched`. Bank
   retries are absorbed (a payment and its bank charge that share a `requestId` are both kept).
   A notification whose account number is not one of ours is parked on the company with no bank
   account and listed under [Unresolved payments](#13-unresolved-payments).
2. **Auto-confirm a pending receipt** — when exactly one **pending** receipt on that account has
   the same amount and a transaction date inside the match window, the receipt is confirmed by the
   bank. It posts at once, its approval chain closes (open steps are skipped with the comment
   *"Confirmed by bank payment FT26300QQ5T3 received 27 Oct 2026"*), the cashbook line is
   reconciled against the payment, and the payment becomes `matched` (`matched_by: system`).
   **Groups:** two or more pending receipts on the account that share the same transaction number
   (compared without case, spaces or punctuation), the same tenant (`paying_user_id`) and the same
   property (`facility_id`), each smaller than the payment, whose amounts add up to the payment
   amount (at the cent) count as **one** candidate. The whole group must add up: no subset of a
   group is ever tried. When that group is the only candidate, every receipt in it is confirmed
   exactly as a single receipt is, and the payment is matched to all of them.
   Two or more candidates (singles or groups) are left as suggestions.
3. **Auto-receipt** — when the payer is certain, a **confirmed** receipt is created and allocated
   by the system and the payment becomes `receipted` (`matched_by: system`). See
   [When the system receipts by itself](#5-when-the-system-receipts-by-itself).
4. **Manual** — anything left stays `unmatched`. Staff can
   [match it to a pending receipt](#9-match-to-a-pending-receipt),
   [create a receipt from it](#14-create-a-receipt-from-a-payment), or
   [discard it](#11-discard-a-payment) (reversible with [restore](#12-restore-a-payment)).

Other things to know:

- **Pay-outs are listed only.** A payment with `direction.value = "out"` (negative amount at the
  bank) is parked and shown, and can be discarded/restored. It is never matched or receipted.
- **Verification runs the other way too.** A receipt keyed normally is confirmed at once when an
  unmatched pay-in on its receiving account has exactly one candidate and that candidate includes
  the new receipt, including the last receipt of a group (see [Verified by bank](#15-verified-by-bank)).
- **Backfill.** `php artisan bank-ipn:import-logs` parks the notifications already written to the
  IPN log files and computes suggestions **without** auto-confirming or auto-receipting, so they
  can be reconciled by hand from the page (see [Backfill command](#17-backfill-command)).
- Receipts matched to or created from a bank payment post immediately and **bypass the approval
  chain**: the bank's money is the strongest evidence an approver could ask for.
- The cashbook line of such a receipt is stored as already reconciled (`bank_entry` in
  statement-line shape), so a later [statement reconciliation](./bank-statement-reconciliation.md)
  recognises it and does not offer it again.
- Every IPN receipt uses the single global **Online Payments** payment method (code `ONLINE`; no
  reference pattern, no proof of payment, bank account required).

Base path for account-scoped endpoints (`{account}` below):
`/api/v1/app/{company}/property-management/settings/finance/banks/accounts/{account}`

Company-level endpoints live under
`/api/v1/app/{company}/property-management/settings/finance/banks`.

Endpoints:

- `GET {account}/payments`
- `GET {account}/payments/summary`
- `POST {account}/payments/rematch`
- `GET {account}/payments/{payment}`
- `POST {account}/payments/{payment}/match`
- `POST {account}/payments/{payment}/rematch`
- `POST {account}/payments/{payment}/discard`
- `POST {account}/payments/{payment}/restore`
- `GET banks/payments/unresolved`
- `POST banks/payments/{payment}/assign`
- `POST lease-management/receipts` with `bank_payment_id` (existing endpoint)

`{payment}` is scoped to `{account}`: a payment id that belongs to another account answers `404`.

---

## 1. Permissions and settings

### Permissions

| Permission | Grants |
|---|---|
| `view-bank-payment` | List, summary, show, unresolved list |
| `match-bank-payment` | Match to a pending receipt, create a receipt from a payment, rematch (one or whole account), assign an unresolved payment |
| `discard-bank-payment` | Discard and restore |

Every row of the resource carries a `permissions` object (see [section 3](#3-the-payment-resource))
so the UI can show or hide actions per row. A missing permission answers `403`.

### Per-company settings

Four rows under **Settings > Finance > General** (they render with the other general settings; no
extra screen):

| Key | Type | Default | Meaning |
|---|---|---|---|
| `bank_payments_auto_confirm` | yes/no | `yes` | Confirm a unique equal-amount pending receipt automatically (step 2). |
| `bank_payments_auto_receipt` | yes/no | `yes` | Create confirmed receipts automatically when the payer is certain (step 3). |
| `bank_payments_match_window_days` | whole number, 1 to 30 | `5` | A pending receipt is a candidate only when its `transaction_date` is within this many days of the bank's `transacted_at`. |
| `bank_payments_receiving_user_id` | staff user | company owner | The user recorded as `receiving_user` on system-created receipts. |

When both `yes/no` settings are `no`, payments are only parked and suggested; nothing is confirmed
or receipted without a person. The settings also gate [rematch](#10-rematch).

---

## 2. Statuses

| Field | Values (colour) |
|---|---|
| `status` | `unmatched` (warning), `matched` (success), `receipted` (success), `discarded` (secondary) |
| `direction` | `in` (success), `out` (danger) |
| `matched_by` | `system` (info), `user` (primary); `null` until matched |

- `unmatched` — waiting for a person (or for the matcher).
- `matched` — an existing pending receipt was confirmed by this payment.
- `receipted` — a receipt was created from this payment.
- `discarded` — set aside with a reason; reversible.

---

## 3. The payment resource

`BankAccountPaymentResource`. `id` is always present; every other field is returned when loaded,
relations only when included (`include=`).

```json
{
  "id": 412,
  "direction": { "value": "in", "color": "success" },
  "status": { "value": "receipted", "color": "success" },
  "matched_by": { "value": "system", "color": "info" },
  "bank": "kcb",
  "transaction_reference": "FT26300HLFKM",
  "request_id": "5d3f7c1e-8a2b-4e57-9a43-0c1f6b2d9e11",
  "channel_code": "100",
  "transacted_at": { "raw": "2026-10-27T11:42:08.000000Z", "formatted": "27 Oct, 2026 14:42", "diff": "5 days ago" },
  "received_at": { "raw": "2026-10-27T11:42:11.000000Z", "formatted": "27 Oct, 2026 14:42", "diff": "5 days ago" },
  "amount": 6000.0,
  "currency": { "id": 1, "name": "Kenya Shillings", "code": "KES" },
  "customer_reference": "RIVERSIDE",
  "customer_name": "Morris murungi",
  "payer_name": "Morris murungi",
  "customer_mobile": "254740217451",
  "payer_phone": "740217451",
  "narration": "Mobi",
  "credit_account_identifier": "1107685575",
  "organization_short_code": "522522",
  "till_number": "522522",
  "signature_verified": true,
  "bank_account_id": 5,
  "bank_account": { "id": 5, "account_name": "NW REALITE LTD", "account_number": "1107685575" },
  "receipt": {
    "id": 2501,
    "transaction_number": "FT26300HLFKM",
    "amount": 6000.0,
    "status": { "value": "confirmed", "color": "success" },
    "paying_user": { "id": 77, "name": "Morris Murungi" }
  },
  "receipts": [
    {
      "id": 2501,
      "transaction_number": "FT26300HLFKM",
      "amount": 6000.0,
      "status": { "value": "confirmed", "color": "success" },
      "paying_user": { "id": 77, "name": "Morris Murungi" }
    }
  ],
  "matched_by_user": null,
  "matched_at": "2026-10-27T11:42:14.000000Z",
  "discard_reason": null,
  "discarded_by": null,
  "discarded_at": null,
  "suggestions": null,
  "suggested_at": "2026-10-27T11:42:13.000000Z",
  "permissions": {
    "view": true,
    "match": true,
    "createReceipt": false,
    "rematch": false,
    "discard": false,
    "restore": false
  }
}
```

| Field | Type | Notes |
|---|---|---|
| `direction` / `status` / `matched_by` | `{ value, color }` | See [Statuses](#2-statuses). `matched_by` is `null` until matched. |
| `bank` | string | Notifying bank key (`kcb`). |
| `transaction_reference` | string | The bank's `transactionReference` (`FT...`). **Not unique**: a SWIFT payment and its charge share it. |
| `request_id` | string | The bank's `requestId`. Also not unique. |
| `channel_code` | string | Bank channel code (`100` pay-in, `206` pay-out). The sign of the amount decides `direction`; the code only corroborates. |
| `transacted_at` | date object | When the bank booked it (parsed from the bank timestamp, company timezone). `{ raw, formatted, diff }`. |
| `received_at` | date object | When the notification reached us. |
| `amount` | float | **Absolute** value in currency units (never cents). Use `direction` for the sign. |
| `currency` | `{ id, name, code }` | The bank account's currency. |
| `customer_reference` | string\|null | Reference typed by the payer (paybill account number, cheque depositor initials, ...). |
| `customer_name` | string\|null | The name exactly as the bank sent it. Only meaningful for M-Pesa, Pesalink and some RTGS; for cheque deposits the bank sends our own company name, and for pay-outs a `/account` stub. |
| `payer_name` | string\|null | The payer's name for display: `customer_name`, or `null` when that is our own company name or a pay-out stub. Prefer this over `customer_name` when showing who paid. |
| `customer_mobile` | string\|null | The raw mobile number as sent. |
| `payer_phone` | string\|null | Last 9 digits of the payer's phone. `null` when the number is our own company number or a placeholder (`0700000000`). Only M-Pesa notifications carry a real payer phone. Set only on mobile-money payments (a till number, or a Mobi/M-Pesa narration); cheque deposits, transfers and Pesalink carry the account holder's number, not the payer's. |
| `narration` | string\|null | Bank narration (`Mobi`, `Chq Dep`, ...). |
| `credit_account_identifier` | string | The account the bank credited; equals `bank_accounts.account_number`. |
| `organization_short_code`, `till_number` | string\|null | As sent. |
| `signature_verified` | boolean | Whether the IPN signature was verified. |
| `bank_account_id` | integer\|null | The account the payment was credited or debited to. `null` for [unresolved payments](#13-unresolved-payments). Always returned. |
| `bank_account` | `{ id, account_name, account_number }` | Returned when loaded (lists and the other endpoints load it by default). `null` for unresolved payments. |
| `receipt` | object | Include `receipt`. The matched or created receipt (for a group, the **first** one, lowest id): `{ id, transaction_number, amount, status{value,color}, paying_user{id,name} }`. |
| `receipts` | array | **Every** receipt the payment confirmed or created, ascending id, same shape as `receipt`. One element for a single match, several for a group, `[]` while unmatched. Loaded by default on the list, show and action responses (also accepted as `include=receipts`). |
| `matched_by_user` | `{ id, name }` | Include `matchedByUser`. Set when `matched_by` is `user`. |
| `matched_at` | datetime\|null | When it was matched or receipted. |
| `discard_reason`, `discarded_at` | string\|null | Set while `status` is `discarded`. |
| `discarded_by` | `{ id, name }` | Include `discardedBy`. |
| `suggestions` | object\|null | See below. Stored at ingestion and refreshed by rematch; `show` recomputes an unmatched payment's. |
| `suggested_at` | datetime\|null | When `suggestions` was last computed. |
| `payload` | object | **`show` only.** The raw IPN body as received. |
| `permissions` | object | `{ view, match, createReceipt, rematch, discard, restore }`, booleans for the current user **and** this row's state: `match`, `createReceipt` and `rematch` only for an unmatched pay-in, `discard` only for an unmatched payment, `restore` only for a discarded one. A payment with no bank account yet can only be viewed and assigned (`match` is the permission that assigns it). |

### `suggestions`

```json
{
  "receipts": [
    {
      "id": 2488,
      "receipt_ids": [2488],
      "receipts": [{ "id": 2488, "transaction_number": "CHQ-004512", "amount": 72900.0, "transaction_date": "2026-10-26" }],
      "transaction_number": "CHQ-004512",
      "amount": 72900.0,
      "transaction_date": "2026-10-26",
      "paying_user": { "id": 143, "name": "Hillary Kiprono" },
      "facility": { "id": 7, "name": "Riverside Apartments" },
      "score": 90,
      "reasons": ["Same amount", "Transaction date 1 day before the bank date", "Depositor name matches tenant"]
    },
    {
      "id": 2491,
      "receipt_ids": [2491, 2492],
      "receipts": [
        { "id": 2491, "transaction_number": "CHQ-004520", "amount": 40000.0, "transaction_date": "2026-10-25" },
        { "id": 2492, "transaction_number": "CHQ-004520", "amount": 32900.0, "transaction_date": "2026-10-26" }
      ],
      "transaction_number": "CHQ-004520",
      "amount": 72900.0,
      "transaction_date": "2026-10-26",
      "paying_user": { "id": 143, "name": "Hillary Kiprono" },
      "facility": { "id": 7, "name": "Riverside Apartments" },
      "score": 85,
      "reasons": ["2 receipts with the same transaction number total the payment amount", "Transaction date 1 day before the bank date", "Depositor name matches tenant"]
    }
  ],
  "tenants": [
    {
      "id": 77,
      "name": "Morris Murungi",
      "phone": "254740217451",
      "facilities": [{ "id": 7, "name": "Riverside Apartments" }],
      "leases": [{ "id": 310, "facility_id": 7 }],
      "open_invoices": [
        { "id": 9001, "number": "INV0481", "due_at": "2026-10-05", "total": 6000.0, "balance": 6000.0, "lease_id": 310 }
      ],
      "score": 95,
      "reasons": ["Payer phone matches tenant", "One active lease on a property this account collects for"]
    }
  ],
  "invoices": [
    {
      "id": 9001,
      "number": "INV0481",
      "lease_id": 310,
      "facility": { "id": 7, "name": "Riverside Apartments" },
      "tenant": { "id": 77, "name": "Morris Murungi" },
      "balance": 6000.0,
      "token": "481"
    }
  ],
  "auto_confirm_receipt_ids": null,
  "auto_receipt": {
    "user_id": 77,
    "facility_id": 7,
    "allocations": [{ "invoice_id": 9001, "amount": 6000.0 }]
  }
}
```

| Key | Meaning |
|---|---|
| `receipts[]` | Receipt candidates inside the match window, best first: each pending receipt on this account with the same amount, and each **group** of pending receipts sharing a transaction number, tenant and property whose amounts add up to the payment (see step 2 at the top). `score` is higher for stronger evidence (compare scores within one list only); `reasons` are display strings. |
| `receipts[].id` | The candidate's first receipt id (lowest). |
| `receipts[].receipt_ids` | int[], ascending: every receipt in the candidate. One id for a single receipt. Send this list as `receipt_ids` to [match](#9-match-to-a-pending-receipt). |
| `receipts[].receipts` | `[{ id, transaction_number, amount, transaction_date }]`, one per receipt in the candidate, ascending id (`transaction_date` is `Y-m-d`). |
| `receipts[].amount` | The receipt amount, or the group total. |
| `receipts[].transaction_date` | The receipt date, or the latest date in the group (the date score uses it). |
| `receipts[].transaction_number`, `paying_user`, `facility` | Of the first receipt (a group shares them). A group's first reason is *"N receipts with the same transaction number total the payment amount"*, then the same reference / phone / name evidence as a single receipt. |
| `tenants[]` | Candidate payers, found by phone, name, invoice numbers in the references, landlord identifier or the property's paybill account number. `open_invoices` are the tenant's posted invoices with a balance, oldest `due_at` first. |
| `invoices[]` | Invoices detected from numbers typed in `customer_reference` / `narration` (see [section 4](#4-invoice-numbers-typed-by-the-payer)). `token` is the fragment of text that resolved to the invoice. |
| `auto_confirm_receipt_ids` | int[] of the receipts the system would confirm (one id, or a whole group) when exactly one candidate exists, else `null`. Replaces the former `auto_confirm_receipt_id`. |
| `auto_receipt` | What the system would create: `{ user_id, facility_id, allocations: [{ invoice_id, amount }] }`, or `null`. |

Evidence that raises a receipt candidate's score: the bank reference equals the receipt's
`transaction_number`, the payer phone equals the tenant's phone, the payer/drawer name matches the
tenant's name (normalised equal, or about 90% similar), the references contain the cheque number.
Negative evidence excludes a receipt: for an M-Pesa payment, a payer phone that belongs to a
different tenant than the receipt's.

---

## 4. Invoice numbers typed by the payer

Tenants often type the invoice number in the paybill account field or the narration. Both
`customer_reference` and `narration` are scanned.

**Recognised forms:** `INV0481`, `INV 481`, `0481` and `481`; and lists of them separated by a
space, comma, `/`, `&`, `and` or `or` (`INV0481 & INV0482`, `481, 482 and 483`).

**A token counts only when** it resolves to an existing **posted, open** invoice (balance above
zero) of a lease on a property this bank account collects for. Numbers that equal the account
number, the till number or the short code, or that have 9+ digits (phone numbers), are ignored.

**What happens with the matches:**

- All referenced invoices belong to **one tenant on one property** and **either** every token was
  `INV`-prefixed **or** the amount equals the sum of their balances (within 0.01) -> the payer is
  certain and the receipt is auto-created, allocating to those invoices first (oldest-first for any
  remainder).
- Otherwise the invoices are only returned in `suggestions.invoices` and the tenant in
  `suggestions.tenants`.

---

## 5. When the system receipts by itself

Only when `bank_payments_auto_receipt` is `yes`, the payment is an unmatched pay-in, and no unique
pending receipt was confirmed first.

The payer must be identified reliably:

- **M-Pesa** (`narration` `Mobi`): `payer_phone` resolves to exactly one tenant with an Active or
  Suspended lease on a property this account collects for, and exactly one such property; **or**
- **Invoice numbers** typed in the references meet the rule in
  [section 4](#4-invoice-numbers-typed-by-the-payer).

Name-only identification (Pesalink, RTGS, a cheque drawer) is **never** enough: it becomes a
suggestion. Cheque deposits arrive in batches without a drawer, so for cheques the bank payment
validates a receipt the cashier already keyed; it does not create one.

The receipt is created **confirmed** with: method Online Payments, `transaction_number` = the bank
reference, `transaction_date` = the bank time, `receiving_account_id` = the account,
`paying_user_id` = the tenant, `facility_id` = the property, `receiving_user_id` = the
`bank_payments_receiving_user_id` setting, and `pop_message` = narration + customer reference.
Allocation goes to the detected invoices first, then the tenant's open invoices on that property
oldest `due_at` first; any remainder stays unallocated (it can be applied later by the usual
unallocated-money flow). The duplicate-payment check still runs: if it refuses, the payment stays
`unmatched` with the tenant as a suggestion. System receipts ignore the monthly receipting
deadline. The tenant gets the normal receipt email.

---

## 6. List payments

`GET {account}/payments`

Authorization: `view-bank-payment`.

Supported query params:

| Param | Notes |
|---|---|
| `filter[status]` | `unmatched`, `matched`, `receipted`, `discarded`. Comma list allowed (`unmatched,matched`). Discarded payments are left out unless `filter[status]` names them. |
| `filter[direction]` | `in` or `out`. |
| `filter[transacted_between]` | `YYYY-MM-DD,YYYY-MM-DD` (inclusive, on `transacted_at`). |
| `filter[has_suggestion]` | `1` = has at least one receipt, tenant or invoice suggestion; `0` = none. |
| `filter[search]` | Matches the bank reference, request id, payer name, payer phone, narration, customer reference, or an **exact** amount (`6000`, `72900.00`). |
| `sort` | `transacted_at`, `amount`, `received_at` (prefix `-` for descending). Default `-transacted_at,-id`. |
| `include` | `bankAccount`, `receipt`, `receipts`, `matchedByUser`, `discardedBy` (comma list). `receipts` is returned even without it. |
| `per_page`, `page` | Pagination. `per_page` defaults to the application default. |

Example:

`GET .../accounts/5/payments?filter[status]=unmatched&filter[direction]=in&filter[transacted_between]=2026-10-01,2026-10-31&include=receipt&sort=-amount`

Response (`200`), shown with one row:

```json
{
  "data": [
    {
      "id": 413,
      "direction": { "value": "in", "color": "success" },
      "status": { "value": "unmatched", "color": "warning" },
      "matched_by": null,
      "bank": "kcb",
      "transaction_reference": "FT26300QQ5T3",
      "request_id": "0b9e6a42-3c1d-4f0a-8d77-5e2a91c4b6f3",
      "channel_code": "100",
      "transacted_at": { "raw": "2026-10-27T08:15:00.000000Z", "formatted": "27 Oct, 2026 11:15", "diff": "5 days ago" },
      "received_at": { "raw": "2026-10-27T08:15:04.000000Z", "formatted": "27 Oct, 2026 11:15", "diff": "5 days ago" },
      "amount": 72900.0,
      "currency": { "id": 1, "name": "Kenya Shillings", "code": "KES" },
      "customer_reference": "HILLARY",
      "customer_name": null,
      "payer_name": null,
      "customer_mobile": "254722898360",
      "payer_phone": null,
      "narration": "Chq Dep",
      "credit_account_identifier": "1107685575",
      "organization_short_code": "522522",
      "till_number": "522522",
      "signature_verified": true,
      "bank_account_id": 5,
      "receipt": null,
      "receipts": [],
      "matched_at": null,
      "suggested_at": "2026-10-27T08:15:06.000000Z",
      "suggestions": {
        "receipts": [
          { "id": 2488, "receipt_ids": [2488], "receipts": [{ "id": 2488, "transaction_number": "CHQ-004512", "amount": 72900.0, "transaction_date": "2026-10-26" }],
            "transaction_number": "CHQ-004512", "amount": 72900.0, "transaction_date": "2026-10-26",
            "paying_user": { "id": 143, "name": "Hillary Kiprono" }, "facility": { "id": 7, "name": "Riverside Apartments" },
            "score": 90, "reasons": ["Same amount", "Transaction date 1 day before the bank date"] },
          { "id": 2490, "receipt_ids": [2490], "receipts": [{ "id": 2490, "transaction_number": "CHQ-004513", "amount": 72900.0, "transaction_date": "2026-10-27" }],
            "transaction_number": "CHQ-004513", "amount": 72900.0, "transaction_date": "2026-10-27",
            "paying_user": { "id": 151, "name": "Westlands Tenant Ltd" }, "facility": { "id": 9, "name": "Westlands Plaza" },
            "score": 60, "reasons": ["Same amount"] }
        ],
        "tenants": [],
        "invoices": [],
        "auto_confirm_receipt_ids": null,
        "auto_receipt": null
      },
      "permissions": { "view": true, "match": true, "createReceipt": true, "rematch": true, "discard": true, "restore": false }
    }
  ],
  "links": { "first": "...?page=1", "last": "...?page=3", "prev": null, "next": "...?page=2" },
  "meta": { "current_page": 1, "per_page": 15, "total": 41 }
}
```

This cheque deposit has `customer_name: null` and `payer_phone: null`: the bank sent our own
company name and number, which are never treated as the payer. Two pending receipts have the same
amount, so nothing was auto-confirmed and both are offered.

Errors: `403` without `view-bank-payment`; `404` when `{account}` is not a bank account of the
company; `422` for an invalid filter value (for example `filter[direction]=sideways`).

---

## 7. Summary cards

`GET {account}/payments/summary`

Authorization: `view-bank-payment`. Figures for one account, in the account's currency.

```json
{
  "data": {
    "unmatched_in": { "count": 9, "total": 548391.85 },
    "matched_today": { "count": 2, "total": 100000.0 },
    "receipted_this_month": { "count": 4, "total": 68000.0 },
    "unmatched_out": { "count": 4, "total": 7623664.0 },
    "currency": { "id": 1, "name": "Kenya Shillings", "code": "KES" }
  }
}
```

| Key | Meaning |
|---|---|
| `unmatched_in` | Pay-ins with status `unmatched` |
| `matched_today` | Payments with status `matched` whose `matched_at` is today |
| `receipted_this_month` | Payments with status `receipted` whose `matched_at` is in the current month |
| `unmatched_out` | Pay-outs with status `unmatched` |

`total` is a float in currency units (absolute value). Errors: `403`, `404`.

---

## 8. Show a payment

`GET {account}/payments/{payment}`

Authorization: `view-bank-payment`. Returns one payment with **freshly computed** `suggestions`
(also saved on the payment, with `suggested_at`) and the raw `payload`. Suggestions are recomputed
only while the payment is `unmatched`; a matched, receipted or discarded payment keeps the
suggestions it was settled with. Accepts the same `include` values as the list.

```json
{
  "data": {
    "id": 413,
    "status": { "value": "unmatched", "color": "warning" },
    "transaction_reference": "FT26300QQ5T3",
    "amount": 72900.0,
    "customer_reference": "HILLARY",
    "narration": "Chq Dep",
    "suggestions": {
      "receipts": [ { "id": 2488, "receipt_ids": [2488], "transaction_number": "CHQ-004512", "amount": 72900.0, "score": 90, "reasons": ["Same amount"] } ],
      "tenants": [],
      "invoices": [],
      "auto_confirm_receipt_ids": null,
      "auto_receipt": null
    },
    "payload": {
      "transactionReference": "FT26300QQ5T3",
      "requestId": "0b9e6a42-3c1d-4f0a-8d77-5e2a91c4b6f3",
      "channelCode": "100",
      "timestamp": "20261027111500",
      "transactionAmount": "72900.00",
      "currency": "KES",
      "customerReference": "HILLARY",
      "customerName": "NWREALITE LIMITED",
      "customerMobileNumber": "254722898360",
      "balance": "",
      "narration": "Chq Dep",
      "creditAccountIdentifier": "1107685575",
      "organizationShortCode": "522522",
      "tillNumber": "522522"
    },
    "permissions": { "view": true, "match": true, "createReceipt": true, "rematch": true, "discard": true, "restore": false }
  }
}
```

(Shortened; the real response carries every field of the [resource](#3-the-payment-resource).)
Errors: `403`, `404` (payment not on this account).

---

## 9. Match to a pending receipt

`POST {account}/payments/{payment}/match`

Confirms one pending receipt, or a group of them, **by** the bank payment. Each receipt posts at
once, any open approval steps are skipped with a recorded comment, its cashbook row is reconciled
against the payment, and the payment becomes `matched` (`matched_by: user`, `matched_by_user`,
`matched_at`). For a group the payment's `receipt` is the first receipt (lowest id) and
`receipts` lists them all.

Authorization: `match-bank-payment`.

Request body (`application/json`): send `receipt_ids` (preferred) or the legacy `receipt_id`.

| Field | Required | Type | Notes |
|---|---|---|---|
| `receipt_ids` | Yes, unless `receipt_id` is sent | int[] (min 1, distinct) | Usually a candidate's `receipt_ids` from `suggestions.receipts[]`. One id: a pending receipt on this account with the payment's amount. Several: pending receipts on this account sharing the same transaction number, tenant and property, whose amounts add up to the payment amount. |
| `receipt_id` | Yes, unless `receipt_ids` is sent | integer | Legacy single-receipt form; same as `receipt_ids: [id]`. |

```json
{ "receipt_ids": [2491, 2492] }
```

Response (`200`):

```json
{
  "data": {
    "message": "Receipts confirmed by bank payment successfully",
    "payment": { "id": 413, "status": { "value": "matched", "color": "success" }, "matched_by": { "value": "user", "color": "primary" }, "receipts": [ { "id": 2491 }, { "id": 2492 } ] },
    "receipts": [
      { "id": 2491, "status": { "value": "confirmed", "color": "success" } },
      { "id": 2492, "status": { "value": "confirmed", "color": "success" } }
    ],
    "receipt": { "id": 2491, "status": { "value": "confirmed", "color": "success" } }
  }
}
```

`payment` is the full [payment resource](#3-the-payment-resource) (shortened above), with
`receipts` loaded. `receipts` lists every confirmed receipt by ascending id; `receipt` is the first
of them, kept for compatibility. The message is *"Receipt confirmed by bank payment successfully"*
for a single receipt.

Errors:

| Status | Condition | Message |
|---|---|---|
| `403` | Missing `match-bank-payment` | This action is unauthorized. |
| `404` | `{payment}` is not on `{account}` | No query results for model. |
| `422` | Payment is not an unmatched pay-in | Only unmatched incoming payments can be matched. |
| `422` | Receipt is not pending (`receipt_id`) | The receipt is not pending, so it cannot be confirmed by a bank payment. |
| `422` | Receipt is on another account (`receipt_id`) | The receipt was not received into this bank account. |
| `422` | Amounts differ (`receipt_id`) | The receipt amount (72,000.00) does not equal the bank payment amount (72,900.00). |
| `422` | Group does not share one transaction number, tenant and property (`receipt_ids`) | Receipts matched together must share the same transaction number, tenant and property. |
| `422` | Group total differs (`receipt_ids`) | The receipts total 70,000.00 does not equal the bank payment amount 72,900.00. |
| `422` | An id is not a receipt of the company (`receipt_ids`, or `receipt_id` in the legacy form) | The selected receipt id is invalid. |
| `422` | Neither field sent, an id not an integer, or ids repeated | Standard validation message (`receipt_ids` / `receipt_id`). |

---

## 10. Rematch

Recompute suggestions. When the company settings allow, the automatic rules then run as they would
on arrival: confirm a unique pending receipt, else create a receipt when the payer is certain.
Pay-outs and payments that are not `unmatched` are skipped.

### 10.1 Whole account

`POST {account}/payments/rematch`

No body. Queues matching for every unmatched **pay-in** of the account (pay-outs are never
matched). Authorization: `match-bank-payment`.

```json
{ "data": { "message": "Matching has been queued for the unmatched payments", "result": { "queued": 14 } } }
```

`queued` is the number of unmatched pay-ins that were queued (`0` when there are none).

### 10.2 One payment

`POST {account}/payments/{payment}/rematch`

No body. Authorization: `match-bank-payment`. Runs synchronously and returns the refreshed
[payment resource](#3-the-payment-resource); it may now be `matched` or `receipted` if an
automatic rule fired.

```json
{
  "data": {
    "id": 413,
    "status": { "value": "unmatched", "color": "warning" },
    "suggestions": { "receipts": [], "tenants": [], "invoices": [], "auto_confirm_receipt_ids": null, "auto_receipt": null },
    "suggested_at": "2026-11-01T09:20:41.000000Z"
  }
}
```

(Shortened.) Errors: `403` without `match-bank-payment`; `404` payment not on the account; `422`
when the payment is not an unmatched pay-in ("Only unmatched incoming payments can be matched.").

---

## 11. Discard a payment

`POST {account}/payments/{payment}/discard`

For a payment that is not ours, a duplicate, or a bank error. Status becomes `discarded`; the
payment is kept and can be [restored](#12-restore-a-payment). Pay-ins and pay-outs can both be
discarded. Authorization: `discard-bank-payment`.

Request body (`application/json`):

| Field | Required | Type | Notes |
|---|---|---|---|
| `reason` | Yes | string | Max 500 characters |

```json
{ "reason": "Duplicate of FT26300QQ5T3, bank confirmed it was reversed" }
```

Response (`200`):

```json
{
  "data": {
    "message": "Bank payment discarded successfully",
    "payment": {
      "id": 413,
      "status": { "value": "discarded", "color": "secondary" },
      "discard_reason": "Duplicate of FT26300QQ5T3, bank confirmed it was reversed",
      "discarded_by": { "id": 4, "name": "Grace Wanjiku" },
      "discarded_at": "2026-11-01T09:31:02.000000Z"
    }
  }
}
```

(`payment` is the full resource, shortened above.)

Errors:

| Status | Condition | Message |
|---|---|---|
| `403` | Missing `discard-bank-payment` | This action is unauthorized. |
| `404` | Payment not on `{account}` | No query results for model. |
| `422` | `reason` missing | The reason field is required. |
| `422` | `reason` over 500 characters | The reason field must not be greater than 500 characters. |
| `422` | Payment is not `unmatched` | Only unmatched bank payments can be discarded. |

---

## 12. Restore a payment

`POST {account}/payments/{payment}/restore`

No body. A discarded payment goes back to `unmatched`, the discard fields are cleared, and its
suggestions are recomputed (the automatic rules do not run; use [rematch](#10-rematch) for that).
Authorization: `discard-bank-payment`.

```json
{
  "data": {
    "message": "Bank payment restored successfully",
    "payment": { "id": 413, "status": { "value": "unmatched", "color": "warning" }, "discard_reason": null, "discarded_by": null, "discarded_at": null }
  }
}
```

Errors: `403`, `404`, and `422` ("Only discarded bank payments can be restored.") for a payment
that is not `discarded`.

---

## 13. Unresolved payments

A notification whose `creditAccountIdentifier` matches none of the company's bank account numbers
is still parked, with `bank_account_id = null`. These cannot appear under any account, so they
have a company-level list.

### 13.1 List

`GET .../settings/finance/banks/payments/unresolved`

Authorization: `view-bank-payment`. Same filters, sorts and pagination as
[section 6](#6-list-payments). Returns the same resource with `bank_account: null`:

```json
{
  "data": [
    {
      "id": 420,
      "direction": { "value": "in", "color": "success" },
      "status": { "value": "unmatched", "color": "warning" },
      "transaction_reference": "FT26300ABCDE",
      "amount": 15000.0,
      "currency": { "id": 1, "name": "Kenya Shillings", "code": "KES" },
      "credit_account_identifier": "1107999999",
      "bank_account_id": null,
      "bank_account": null,
      "suggestions": null,
      "permissions": { "view": true, "match": true, "createReceipt": false, "rematch": false, "discard": false, "restore": false }
    }
  ],
  "links": { "first": "...?page=1", "last": "...?page=1", "prev": null, "next": null },
  "meta": { "current_page": 1, "per_page": 15, "total": 1 }
}
```

(Shortened. Until a payment is assigned to a bank account, only viewing and assigning are offered.)

### 13.2 Assign to a bank account

`POST .../settings/finance/banks/payments/{payment}/assign`

Attaches the payment to a bank account of the company and runs matching on it (suggestions, plus
the automatic rules when the settings allow). Authorization: `match-bank-payment`.

| Field | Required | Type | Notes |
|---|---|---|---|
| `bank_account_id` | Yes | integer | A bank account of this company |

```json
{ "bank_account_id": 5 }
```

Response (`200`):

```json
{
  "data": {
    "message": "Bank payment assigned to the bank account successfully",
    "payment": { "id": 420, "status": { "value": "unmatched", "color": "warning" }, "bank_account": { "id": 5, "account_name": "NW REALITE LTD", "account_number": "1107685575" } }
  }
}
```

(`payment` is the full resource, shortened above, with `suggestions` computed.)

Errors:

| Status | Condition | Message |
|---|---|---|
| `403` | Missing `match-bank-payment` | This action is unauthorized. |
| `404` | Payment not in this company | No query results for model. |
| `422` | `bank_account_id` missing, or not a bank account of this company | Standard validation message. |
| `422` | Payment already has a bank account | This payment is already assigned to a bank account. |

---

## 14. Create a receipt from a payment

Reuses the normal receipt endpoint:

`POST /api/v1/app/{company}/property-management/lease-management/receipts`

The body is the usual [create receipt](../lease-management/receipts.md#create-receipt) body plus
one optional field:

| Field | Required | Type | Notes |
|---|---|---|---|
| `bank_payment_id` | No | integer | The bank payment this receipt is created from |

When `bank_payment_id` is sent, the payment and the receipt must agree:

| Rule | Detail |
|---|---|
| Payment state | Status `unmatched`, direction `in`, same company |
| `amount` | Equals the payment's `amount` |
| `receiving_account_id` | Equals the payment's bank account |
| `transaction_number` | Equals the payment's `transaction_reference` |
| `transaction_date` | Within a day of the payment's `transacted_at` date |
| `payment_method_id` | Must be the **Online Payments** method |

The receipt is created **confirmed** immediately (no approval chain, no pending state), the
cashbook line is written and reconciled to the payment, and the payment becomes `receipted`
(`matched_by: user`). Everything else in the normal path still applies: the duplicate-payment
check, the property and balance checks, and the monthly receipting deadline (a user without
`receipt-after-deadline` is refused after the deadline, unlike system receipts). Sending
`bank_payment_id` also requires `match-bank-payment`.

Example request (cheque deposit, tenant chosen by the cashier):

```json
{
  "bank_payment_id": 413,
  "transaction_number": "FT26300QQ5T3",
  "receiving_account_id": 5,
  "transaction_date": "2026-10-27",
  "payment_method_id": 9,
  "paying_user_id": 143,
  "amount": 72900,
  "currency_id": 1,
  "pop_message": "Chq Dep HILLARY",
  "allocations": [{ "invoice_id": 9001, "amount": 72900 }]
}
```

Example response:

```json
{
  "message": "Receipt created successfully",
  "receipt": {
    "id": 2510,
    "transaction_number": "FT26300QQ5T3",
    "amount": 72900,
    "allocated": 72900,
    "balance": 0,
    "status": { "value": "confirmed", "color": "success" }
  }
}
```

Additional `422` cases (keyed on `bank_payment_id` unless noted). The agreement rules above are all
checked together, so a request that breaks several returns every message in `errors`:

| Condition | Message |
|---|---|
| Payment not found in this company | The selected bank payment is invalid. |
| Payment is not `unmatched` | This bank payment has already been matched, receipted or discarded. |
| Payment is a pay-out | Only incoming bank payments can be receipted. |
| `amount` differs | The receipt amount must equal the bank payment amount (72,900.00). |
| `receiving_account_id` differs | The receipt must be received into the bank account the payment was credited to. |
| `transaction_number` differs | The receipt reference must equal the bank reference (FT26300QQ5T3). |
| `transaction_date` more than a day off | The receipt date must be within a day of the bank transaction date (2026-10-27). |
| `payment_method_id` is not Online Payments (key `payment_method_id`) | The payment method must be Online Payments for a receipt created from a bank payment. |

---

## 15. Verified by bank

A receipt created **without** `bank_payment_id` stays pending and goes through approval as usual,
**unless** an unmatched pay-in on its `receiving_account_id` inside the match window
(`bank_payments_match_window_days`, measured from the receipt's `transaction_date`), of at least
the receipt's amount, has exactly one receipt candidate and that candidate includes the new
receipt. Then the receipt is confirmed at once, as in [section 9](#9-match-to-a-pending-receipt)
with `matched_by: system`, and the approval chain never prompts anyone. The candidate can be a
**group**: the new receipt may be the one that makes a set of receipts with the same transaction
number, tenant and property add up to the payment, and then every receipt of the group is
confirmed together. With no such payment the receipt proceeds through approval unchanged.

Receipts of one group must differ in amount: the duplicate-payment rule refuses a second live
receipt with the same reference, amount, tenant and property.

The response of `POST receipts` has the same shape as always; read `receipt.status` to see whether
it came back `confirmed` (verified by bank) or `pending`. The receipt then exposes `bank_payment`
(see below).

---

## 16. Receipt resource additions

`FacilityReceiptResource` gains `bank_payment`, returned when the relation is loaded (`null` for
receipts with no bank payment). Every receipt of a matched group carries the same `bank_payment`
(the link is `facility_receipts.bank_payment_id`):

```json
{
  "bank_payment": {
    "id": 413,
    "transaction_reference": "FT26300QQ5T3",
    "transacted_at": { "raw": "2026-10-27T08:15:00.000000Z", "formatted": "27 Oct, 2026 11:15", "diff": "5 days ago" },
    "received_at": { "raw": "2026-10-27T08:15:04.000000Z", "formatted": "27 Oct, 2026 11:15", "diff": "5 days ago" },
    "matched_by": { "value": "user", "color": "primary" },
    "matched_at": "2026-11-01T09:12:30.000000Z",
    "bank_account_id": 5
  }
}
```

The receipt list gains a filter:

| Param | Notes |
|---|---|
| `filter[bank_verified]` | `1` = receipts tied to a bank payment (matched, created from, or verified by); `0` = receipts with none |

Example: `GET .../lease-management/receipts?filter[bank_verified]=1`

---

## 17. Backfill command

```
php artisan bank-ipn:import-logs {--path=storage/logs/bank-ipn} {--company=} {--bank=kcb} {--dry-run}
```

Reads every `IPN received {...}` line from the IPN log files and parks each notification (the same
fingerprint de-duplication applies, so it is safe to run repeatedly), with `received_at` taken from
the log stamp and `signature_verified = false`. Suggestions are computed synchronously. It does
**not** auto-confirm or auto-receipt, whatever the settings say, so the imported payments are
reconciled by hand from the page. `--dry-run` reports what would be parked without writing. The
command prints the parked and duplicate counts.

---

## Notes

- All amounts are floats in currency units (never cents); a payment's `amount` is always positive
  and `direction` carries the sign.
- The payment fingerprint is bank + request id + transaction reference + amount + narration +
  timestamp. `transaction_reference` and `request_id` alone are **not** unique.
- The company's own phone number and company name are never treated as a payer.
- Matching is automatic only for pay-ins; pay-outs are listed and can be discarded or restored.
- Messages above are the English strings; all user-facing messages are translatable.

## Errors

- `403 Forbidden` — the user lacks the endpoint's permission (see
  [section 1](#1-permissions-and-settings)).
- `404 Not Found` — the account is not the company's, or the payment is not on that account.
- `422 Unprocessable Entity` — validation failed, or the payment/receipt is in a state that does
  not allow the action (messages in each section). Shape:
  `{ "message": "...", "errors": { "field": ["..."] } }`.
- `500 Internal Server Error` — unexpected failure; nothing was changed (every action runs in a
  transaction).
