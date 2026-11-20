# Bank Transfers API

Domain: `Property Management > Settings > Finance > Banks`

Base route:

`/api/v1/app/{company}/property-management/settings/finance/banks/transfers`

A bank transfer moves money from one bank account on the company to another. Either account may be
the company's own or a landlord's or supplier's, for example between two accounts the same landlord
holds. Recording one writes **two cashbook lines** and **a payment voucher**, all or nothing:

- a **credit** in the **From** account's cashbook, because money leaves it;
- a **debit** in the **To** account's cashbook, because money arrives in it;
- a **payment voucher**, already **paid**, whose one item is the transfer (see
  [The transfer's payment voucher](#the-transfers-payment-voucher)).

Both lines carry the same amount, date and `reference`, and both point back at the transfer
(`transaction: {"type": "BankTransfer", "id": …}` on the [statement](bank-account-statement.md)),
so either side leads to the other. If either line or the voucher cannot be written, nothing is
recorded.

A transfer entered by mistake is taken back by **cancelling its payment voucher**: both cashbook
lines are removed and the transfer is marked `cancelled`. **Reversal is turned off for now.**

## Endpoints

- `GET /transfers`: list
- `POST /transfers`: record a transfer
- `GET /transfers/{transfer}`: show, with its cashbook lines
- `POST /transfers/{transfer}/reverse`: turned off, answers **403**
- `PATCH /api/v1/app/{company}/property-management/finance/payment-vouchers/{payment_voucher.id}/cancel`:
  cancel the transfer, through its voucher

Permissions: `view-bank-transfer` (list, show), `create-bank-transfer` (record) and
`cancel-facility-payment-voucher` (cancel).

Every transfer carries **`permissions`**:

- `cancel`: `true` when the transfer has a voucher, the voucher is still paid, and the caller holds
  `cancel-facility-payment-voucher`. Use it to show or hide **Cancel** per row.
- `reverse`: always `false` while reversal is turned off. Hide **Reverse**.

## Record a transfer

`POST /api/v1/app/{company}/property-management/settings/finance/banks/transfers`

| Field | Required | Type | Notes |
|---|---|---|---|
| `from_bank_account_id` | Yes | integer | The account money leaves |
| `to_bank_account_id` | Yes | integer | The account money enters. Not the same as From |
| `amount` | Yes | number, > 0 | |
| `transfer_date` | Yes | date (`YYYY-MM-DD`) | Not in the future. Both cashbook lines are dated on it |
| `reference` | No | string, up to 100 | E.g. the bank's reference. Unique within the company. Generated as `TRF-000123` when not given |
| `notes` | No | string, up to 1000 | |

Refused with a **422**:

| Key | When |
|---|---|
| `to_bank_account_id` | Same account as From ("Money has to move to a different account.") |
| `from_bank_account_id` / `to_bank_account_id` | Not one of this company's accounts; not active |
| `to_bank_account_id` | The two accounts hold different currencies. An account with no currency set is taken to hold the other's |
| `amount` | 0 or less |
| `transfer_date` | In the future |
| `reference` | Already used by another transfer of the company |
| `to_bank_account_id` | A landlord's or supplier's account with no owner to raise the voucher to |

Example request:

```json
{
  "from_bank_account_id": 69,
  "to_bank_account_id": 66,
  "amount": 150000,
  "transfer_date": "2026-11-11",
  "notes": "Float for supplier payments"
}
```

Example response:

```json
{
  "message": "Transfer recorded",
  "transfer": {
    "id": 1,
    "reference": "TRF-000001",
    "amount": "150000.00000",
    "transfer_date": { "raw": "2026-11-11", "formatted": "11 Nov, 2026" },
    "notes": "Float for supplier payments",
    "status": "completed",
    "cancelled_at": null,
    "cancelled_by": null,
    "payment_voucher": { "id": 412, "payable_as": "other", "status": { "value": "paid", "color": "success" } },
    "reversed_at": null,
    "reversal_date": null,
    "reversal_reason": null,
    "reversed_by": null,
    "from_account": { "id": 69, "account_name": "NW Realite - Collections", "account_number": "0990011223", "alias": null, "currency_id": 1 },
    "to_account": { "id": 66, "account_name": "NW Realite - Supplier Payments", "account_number": "2050000001", "alias": null, "currency_id": 1 },
    "currency": { "id": 1, "code": "KES", "name": "Kenyan Shilling" },
    "created_by": { "id": 1, "name": "Super Admin" },
    "entries": [
      { "id": 501, "bank_account_id": 69, "debit": "0.00000", "credit": "150000.00000", "refference_number": "TRF-000001", "notes": "Transfer TRF-000001 to NW Realite - Supplier Payments (2050000001)", "reconciliation_status": "pending" },
      { "id": 502, "bank_account_id": 66, "debit": "150000.00000", "credit": "0.00000", "refference_number": "TRF-000001", "notes": "Transfer TRF-000001 from NW Realite - Collections (0990011223)", "reconciliation_status": "pending" }
    ],
    "created_at": { "raw": "2026-11-11T09:12:40.000000Z", "formatted": "11 Nov, 2026" },
    "permissions": { "reverse": false, "cancel": true }
  }
}
```

The lines are ordinary cashbook lines: they show on each account's statement, move its balance,
and are matched in [reconciliation](bank-statement-reconciliation.md) like any other line.

## The transfer's payment voucher

Each transfer is raised with a payment voucher (`payment_voucher` on the transfer, with its `id`,
`payable_as` and `status`; on the voucher, `bank_transfer_id` and `bank_transfer`). Open it on the
page `payable_as` names: `landlord`, `vendor`, or `other` for one of the company's own accounts. It
is a record of the transfer:

- **Payee**: the owner of the **To** account. For one of the company's own accounts, that is the
  company, through the user who owns it (`payable_as: "other"`). For a landlord's or supplier's
  account, that landlord or supplier (`payable_as: "landlord"` or `"vendor"`).
- **Accounts**: `debit_bank_account` is the From account (money paid out of it),
  `credit_bank_account` the To account.
- **Amount, currency, `transaction_number`**: the transfer's amount, currency and `reference`.
- **Item**: one line, the transfer, described as "Transfer TRF-000001 from … to …".
- **Status**: `paid` from the start, with `paid_at` the transfer date. It is not released, so it adds
  no cashbook line of its own and sends no payment advice email.

It shows in the payment vouchers list and prints like any other voucher, with the transfer as its
one line.

What it allows (its `permissions` say so per voucher):

| Action | On a transfer's voucher |
|---|---|
| Cancel | Yes, while it is `paid`. See below |
| Release | No: it is already paid (**422**). `permissions.release` is still `true`, as on every paid voucher, because the same check allows the payment advice upload; hide Release by status |
| Reverse | No (**403**). Cancel it instead |
| Edit | No (**403**) |
| Delete | No (**403**): it stays as the record of the transfer, cancelled or not |
| Payment advice (`PUT …/pay`) | Yes, like any paid voucher |

## Cancel a transfer

`PATCH /api/v1/app/{company}/property-management/finance/payment-vouchers/{payment_voucher.id}/cancel`

No body. Cancelling the transfer's voucher **removes both cashbook lines**: the credit in the From
account and the debit in the To account. Both balances are as if the transfer never happened. The
voucher becomes `cancelled`, and the transfer `status: "cancelled"` with `cancelled_at` and
`cancelled_by`. The transfer itself is kept, with no `entries`.

Refused:

| Status | When |
|---|---|
| **403** | The caller lacks `cancel-facility-payment-voucher`, or the voucher is already cancelled |
| **422** on `status` | Either cashbook line is matched in [reconciliation](bank-statement-reconciliation.md). Unmatch it first |

Transfers recorded before vouchers were raised for them have `payment_voucher: null` and
`permissions.cancel: false`: they cannot be cancelled from the app.

## Reverse a transfer (turned off)

`POST /api/v1/app/{company}/property-management/settings/finance/banks/transfers/{transfer}/reverse`

**Turned off for now:** the endpoint answers **403** ("Reversing a bank transfer is turned off.
Cancel its payment voucher instead.") and writes nothing. What follows is how it works when it is
turned back on.

| Field | Required | Type | Notes |
|---|---|---|---|
| `reason` | Yes | string, up to 1000 | Kept on the transfer |
| `reversal_date` | No | date (`YYYY-MM-DD`) | Today when not given. Not before the transfer, not in the future |

Adds a **debit** to the From account's cashbook and a **credit** to the To account's, each the
transfer's amount, dated `reversal_date`, with `refference_number` `{reference}-REV`. Both balances
are back where they were, and both statements show the transfer and its reversal. The response is
the transfer with `status: "reversed"`, `reversed_at`, `reversal_date`, `reversal_reason`,
`reversed_by`, and all four `entries`.

A transfer is reversed once: a second reversal answers **422** on `transfer` ("This transfer has
already been reversed.").

## List transfers

`GET /api/v1/app/{company}/property-management/settings/finance/banks/transfers`

- Filters:
  - `filter[bank_account_id]`: transfers in **or** out of an account (comma-separated ids match any)
  - `filter[from_bank_account_id]`, `filter[to_bank_account_id]`: exact
  - `filter[status]`: `completed`, `cancelled` or `reversed`
  - `filter[reference]`: partial
  - `filter[transfer_date_between]`: `YYYY-MM-DD,YYYY-MM-DD`
- Sort: `transfer_date`, `amount`, `reference`, `created_at` (default newest transfer date first)
- Pagination: `per_page`, `page`

Rows have the same shape as above, without `entries`; `GET /transfers/{transfer}` includes them.
