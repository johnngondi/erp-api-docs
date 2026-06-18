# Vendor Statements API

Domain: `Property Management > Finance`

Base route:

`/api/v1/app/{company}/property-management/finance/vendor-statements`

A vendor statement is the running ledger of what a vendor is owed and what has been settled. It is
**read-only** and assembled on the fly from the lines recorded as bills move through their
lifecycle (see [How lines are recorded](#how-statement-lines-are-recorded)).

## Endpoints

- `GET /vendor-statements`

## Get a vendor statement

`GET /api/v1/app/{company}/property-management/finance/vendor-statements`

Returns the opening balance, the period's dated lines (each with a running balance), the period
totals, and the closing balance.

Supported query params:

- Filters:
  - `filter[vendor_id]` — **required** (`users.id`). A statement is always for one vendor.
  - `filter[facility_contract_id]` — optional (`facility_contracts.id`); restrict to lines raised
    against a single contract. The opening balance is scoped to the same contract.
  - `filter[transaction_at]` — optional date range over the line date — see
    [Date-range filtering](#date-range-filtering). **Defaults to the current month** when omitted.

There is no pagination, sort, or include — a statement is returned whole for the resolved period.

### Date-range filtering

`transaction_at` is a range filter. It accepts an inclusive `from`/`to` pair (dates, `Y-m-d`);
either bound may be omitted. Both shapes are supported:

- Bracket form: `filter[transaction_at][from]=2026-06-01&filter[transaction_at][to]=2026-06-30`
- CSV form: `filter[transaction_at]=2026-06-01,2026-06-30`

When neither bound is supplied the period defaults to the **current month**
(`startOfMonth … endOfMonth`).

### Balance convention

A vendor is a payable, so a **positive balance is what is owed to the vendor**. Each line moves the
balance by `credit − debit`:

- Posting a bill **credits** (raises) the balance by the full bill total.
- A credit note is a **negative credit** (lowers it).
- Settling a withholding and paying a bill via a voucher are **debits** (lower it).

A fully-settled bill therefore nets to zero.

- `brought_forward` (BBF) — net `credit − debit` for the vendor **before** the period start.
- Each line's `balance` — the running balance up to and including that line (starting from BBF).
- `carried_forward` (BCF) — the closing balance; equals the last line's `balance`
  (`brought_forward + period credit − period debit`).

Sample response:

```json
{
  "data": {
    "vendor_id": 7,
    "facility_contract_id": null,
    "period": { "from": "2026-06-01", "to": "2026-06-30" },
    "brought_forward": 1000.0,
    "transactions": [
      {
        "id": 5012,
        "transaction": { "type": "FacilityBill", "id": 1201 },
        "facility_contract": null,
        "notes": "Bill#1201 - INV#INV-1001 - Quarterly maintenance",
        "debit": "0.00000",
        "credit": "1160.00000",
        "balance": 2160.0,
        "transaction_at": {
          "raw": "2026-06-05T00:00:00.000000Z",
          "formatted": "05 Jun, 2026",
          "diff": "3 weeks ago"
        }
      },
      {
        "id": 5044,
        "transaction": { "type": "FacilityPaymentVoucher", "id": 801 },
        "facility_contract": null,
        "notes": "PV#801 - Payment for Bill#1201 - INV#INV-1001 - Quarterly maintenance",
        "debit": "1160.00000",
        "credit": "0.00000",
        "balance": 1000.0,
        "transaction_at": {
          "raw": "2026-06-20T00:00:00.000000Z",
          "formatted": "20 Jun, 2026",
          "diff": "5 days ago"
        }
      }
    ],
    "totals": { "debit": 1160.0, "credit": 1160.0 },
    "carried_forward": 1000.0
  }
}
```

Field notes (`FacilityVendorStatementResource`):

| Field | Type | Notes |
|---|---|---|
| `transaction.type` | string\|null | Source model short name: `FacilityBill`, `FacilityBillWithholding`, or `FacilityPaymentVoucher`. |
| `transaction.id` | integer\|null | Id of that source record. |
| `facility_contract` | object\|null | Present (`{ id, title }`) for lines raised against a contract. |
| `notes` | string | Human-readable line description. |
| `debit` / `credit` | string | Decimal (5 dp), currency units. |
| `balance` | number | Running balance up to and including this line. |
| `transaction_at` | object | `{ raw, formatted, diff }`. |

## How statement lines are recorded

Lines are written automatically as part of the bill/voucher lifecycle — there is no endpoint to
create them directly.

| Event | Line | Source (`transaction`) | Notes format |
|---|---|---|---|
| Bill posted to expenses | credit = bill total | the `FacilityBill` | `Bill#{id} - INV#{invoice} - {notes}` |
| Credit note raised | negative credit | the credit-note `FacilityBill` | `Bill#{srcId} - INV#{srcInvoice} - CreditNote#{id} {notes}` |
| Withholding settled (liability bill raised) | debit | the `FacilityBillWithholding` | `Bill#{srcId} - INV#{srcInvoice} - Withholding {rate%} - {notes}` |
| Bill paid via voucher | debit | the `FacilityPaymentVoucher` | `PV#{id} - Payment for Bill#{billId} - INV#{invoice} - {notes}` |

`INV#` is always the original bill's `invoice_number`. The contract (`facility_contract_id`) is
carried whenever the originating bill was generated from a fixed contract.

### Cancellation side-effects

| Cancelled | Statement effect |
|---|---|
| Payment voucher | The voucher's lines are deleted. |
| Bill — expense **was** remitted | A negative reversal line is added (original kept, nets to zero). |
| Bill — **not** remitted | The bill's lines are deleted. |
| Credit note | The credit note's line is deleted. |
| Liability bill | The withholding settlement debit lines it produced are deleted (the settlement is unwound and may be redone). |
