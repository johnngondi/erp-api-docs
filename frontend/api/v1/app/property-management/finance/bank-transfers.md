# Bank Transfers API

Domain: `Property Management > Settings > Finance > Banks`

Base route:

`/api/v1/app/{company}/property-management/settings/finance/banks/transfers`

A bank transfer moves money from one of the company's own bank accounts to another. Recording one
writes **two cashbook lines**, both or neither:

- a **credit** in the **From** account's cashbook, because money leaves it;
- a **debit** in the **To** account's cashbook, because money arrives in it.

Both lines carry the same amount, date and `reference`, and both point back at the transfer
(`transaction: {"type": "BankTransfer", "id": …}` on the [statement](bank-account-statement.md)),
so either side leads to the other. If either line cannot be written, nothing is recorded.

A transfer entered by mistake is **reversed**, never deleted: two opposite lines are added and the
transfer is marked `reversed`.

## Endpoints

- `GET /transfers`: list
- `POST /transfers`: record a transfer
- `GET /transfers/{transfer}`: show, with its cashbook lines
- `POST /transfers/{transfer}/reverse`: reverse it

Permissions: `view-bank-transfer` (list, show), `create-bank-transfer` (record) and
`reverse-bank-transfer` (reverse).

Every transfer carries **`permissions.reverse`**: `true` when the caller holds
`reverse-bank-transfer` and the transfer is not already reversed. Use it to show or hide **Reverse**
per row. `GET /me/permissions` does not list `reverse-…` names: it returns only `view-`, `create-`,
`generate-`, `export-`, `manage-` and `sign-` permissions.

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
| `from_bank_account_id` / `to_bank_account_id` | Not one of this company's accounts; not the company's own account (a supplier's or landlord's account is paid with a voucher, not transferred to); not active |
| `to_bank_account_id` | The two accounts hold different currencies. An account with no currency set is taken to hold the other's |
| `amount` | 0 or less |
| `transfer_date` | In the future |
| `reference` | Already used by another transfer of the company |

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
    "permissions": { "reverse": true }
  }
}
```

The lines are ordinary cashbook lines: they show on each account's statement, move its balance,
and are matched in [reconciliation](bank-statement-reconciliation.md) like any other line.

## Reverse a transfer

`POST /api/v1/app/{company}/property-management/settings/finance/banks/transfers/{transfer}/reverse`

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
  - `filter[status]`: `completed` or `reversed`
  - `filter[reference]`: partial
  - `filter[transfer_date_between]`: `YYYY-MM-DD,YYYY-MM-DD`
- Sort: `transfer_date`, `amount`, `reference`, `created_at` (default newest transfer date first)
- Pagination: `per_page`, `page`

Rows have the same shape as above, without `entries`; `GET /transfers/{transfer}` includes them.
