# Bank Account Statement (Cashbook) API

Domain: `Property Management > Settings > Finance > Banks`

Base route:

`/api/v1/app/{company}/property-management/settings/finance/banks/accounts/{account}/transactions`

A bank account statement is the running cashbook of a single bank account: the money received into
it and the money paid out of it. It is **read-only** and assembled on the fly from the transaction
lines recorded as receipts and payment vouchers are processed (see
[How lines are recorded](#how-statement-lines-are-recorded)).

## Endpoints

- `GET /accounts/{account}/transactions`

## Get a bank account statement

`GET /api/v1/app/{company}/property-management/settings/finance/banks/accounts/{account}/transactions`

Returns the opening balance, the period's dated lines (each with a running balance), the period
totals, and the closing balance. `{account}` is the bank account id.

Authorization: the `view` ability on the bank account.

Supported query params:

- Filters:
  - `filter[transaction_at]` — optional date range over the line date — see
    [Date-range filtering](#date-range-filtering). **Defaults to the current month** when omitted.

There is no pagination, sort, or include — a statement is returned whole for the resolved period,
ordered by `transaction_at` then `id`.

### Date-range filtering

`transaction_at` is the only supported filter and is a range filter. It accepts an inclusive
`from`/`to` pair (dates, `Y-m-d`); either bound may be omitted. Both shapes are supported:

- Bracket form: `filter[transaction_at][from]=2026-06-01&filter[transaction_at][to]=2026-06-30`
- CSV form: `filter[transaction_at]=2026-06-01,2026-06-30`

When neither bound is supplied the period defaults to the **current month**
(`startOfMonth … endOfMonth`).

### Balance convention

From the account's point of view a **debit is money in** and a **credit is money out**, so a
positive balance is the available cash. Each line moves the balance by `debit − credit`:

- Processing a receipt **debits** (raises) the balance by the amount received.
- Releasing a payment voucher **credits** (lowers) the balance by the amount paid out.

- `brought_forward` (BBF) — net `debit − credit` for the account **before** the period start.
- Each line's `balance` — the running balance up to and including that line (starting from BBF).
- `carried_forward` (BCF) — the closing balance; equals the last line's `balance`
  (`brought_forward + period debit − period credit`).

Sample response:

```json
{
  "data": {
    "bank_account_id": 12,
    "period": { "from": "2026-06-01", "to": "2026-06-30" },
    "brought_forward": 5000.0,
    "transactions": [
      {
        "id": 9001,
        "transaction": { "type": "FacilityReceipt", "id": 4210 },
        "refference_number": "RCP-000123",
        "payment_method": { "id": 3, "name": "Bank Transfer" },
        "notes": "Receipt#4210 - 05 Jun, 2026 - Jane Tenant - Riverside Apartments",
        "debit": "12000.00000",
        "credit": "0.00000",
        "balance": 17000.0,
        "transaction_at": {
          "raw": "2026-06-05T10:14:00.000000Z",
          "formatted": "05 Jun, 2026",
          "diff": "3 weeks ago"
        }
      },
      {
        "id": 9044,
        "transaction": { "type": "FacilityPaymentVoucher", "id": 801 },
        "refference_number": "PV-000801",
        "payment_method": { "id": 3, "name": "Bank Transfer" },
        "notes": "PV#801 - vendor - Acme Maintenance Ltd",
        "debit": "0.00000",
        "credit": "4000.00000",
        "balance": 13000.0,
        "transaction_at": {
          "raw": "2026-06-20T09:00:00.000000Z",
          "formatted": "20 Jun, 2026",
          "diff": "5 days ago"
        }
      }
    ],
    "totals": { "debit": 12000.0, "credit": 4000.0 },
    "carried_forward": 13000.0
  }
}
```

Field notes (`BankAccountTransactionResource`):

| Field | Type | Notes |
|---|---|---|
| `transaction.type` | string\|null | Source model short name: `FacilityReceipt` or `FacilityPaymentVoucher`. |
| `transaction.id` | integer\|null | Id of that source record. |
| `refference_number` | string\|null | The source receipt/voucher `transaction_number`. |
| `payment_method` | object\|null | `{ id, name }` when the source carried a payment method. |
| `notes` | string | Human-readable line description. |
| `debit` / `credit` | string | Decimal (5 dp), currency units. |
| `balance` | number | Running cash balance up to and including this line. |
| `transaction_at` | object | `{ raw, formatted, diff }`. |

## How statement lines are recorded

Lines are written automatically as part of the receipt/voucher lifecycle — there is no endpoint to
create them directly. Recording is **idempotent**: a receipt or voucher that has already produced a
line is skipped, so re-processing never double-posts.

| Event | Line | Account | Source (`transaction`) | Notes format |
|---|---|---|---|---|
| Receipt processed | debit = receipt amount | the receipt's `receiving_account_id` | the `FacilityReceipt` | `Receipt#{id} - {transaction_date} - {paying user} - {property}` |
| Payment voucher released | credit = voucher amount | the voucher's `debit_bank_account_id` | the `FacilityPaymentVoucher` | `PV#{id} - {payable_as} - {receiving user}` |

`refference_number` carries the source `transaction_number`, and `transaction_at` is set when the
receipt is processed / the voucher is released (paid). A receipt with no receiving account, or a
voucher with no debit bank account, produces no line.
