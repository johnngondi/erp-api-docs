# Vendor Statement API

Domain: `Vendor Portal > Finance`

Base route:

`/api/v1/vendor/finance/vendor-statements`

The logged-in vendor's own statement: a running ledger of what they are owed and what has been
settled. It is **read-only** and assembled from the lines recorded as bills move through their
lifecycle (the same data as the app-side statement — see the
[app Vendor Statements doc](../../app/property-management/finance/vendor-statement.md#how-statement-lines-are-recorded)).

## Endpoints

- `GET /vendor-statements`

## Get my statement

`GET /api/v1/vendor/finance/vendor-statements`

The vendor is **inferred from the authenticated user** — there is no `vendor_id` param. Returns the
opening balance, the period's dated lines (each with a running balance), the period totals, and the
closing balance.

Supported query params (all optional):

- `filter[facility_contract_id]` — restrict to lines raised against a single contract
  (`facility_contracts.id`). The opening balance is scoped to the same contract.
- `filter[transaction_at]` — date range over the line date — see
  [Date-range filtering](#date-range-filtering). **Defaults to the current month** when omitted.

There is no pagination, sort, or include — the statement is returned whole for the resolved period.

### Date-range filtering

`transaction_at` is a range filter accepting an inclusive `from`/`to` pair (dates, `Y-m-d`); either
bound may be omitted. Both shapes are supported:

- Bracket form: `filter[transaction_at][from]=2026-06-01&filter[transaction_at][to]=2026-06-30`
- CSV form: `filter[transaction_at]=2026-06-01,2026-06-30`

### Balance convention

A vendor is a payable, so a **positive balance is what is owed to the vendor**. Each line moves the
balance by `credit − debit` (bills credit it up; credit notes, withholding settlements and voucher
payments draw it down).

- `brought_forward` (BBF) — net balance **before** the period start.
- Each line's `balance` — running balance up to and including that line.
- `carried_forward` (BCF) — closing balance (`brought_forward + period credit − period debit`).

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
      }
    ],
    "totals": { "debit": 1160.0, "credit": 1160.0 },
    "carried_forward": 1000.0
  }
}
```

Field notes match the app-side `FacilityVendorStatementResource` — see the
[app doc](../../app/property-management/finance/vendor-statement.md#get-a-vendor-statement).
