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
- `GET /vendor-statements/export` — the same statement as a PDF or Excel download

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

## Export my statement

`GET /api/v1/vendor/finance/vendor-statements/export`

Download the statement above as a file. **It accepts every filter the statement itself accepts** —
same param names, same defaults — plus a required `format`, and the vendor is inferred from the
authenticated user exactly as it is on the statement. The statement is regenerated server-side, so
export by **replaying the current query string with `format` appended**.

- `format` — **required**, `excel` or `pdf`. Anything else ⇒ `422` on `format`.
- `filter[facility_contract_id]`, `filter[transaction_at]` — optional, exactly as above.

Example:

```
GET /api/v1/vendor/finance/vendor-statements/export?format=pdf&filter[transaction_at][from]=2026-06-01&filter[transaction_at][to]=2026-06-30
```

### Response

**Not JSON** — a binary file with `Content-Disposition: attachment`:

| `format` | `Content-Type` | Extension |
|---|---|---|
| `excel` | `application/vnd.openxmlformats-officedocument.spreadsheetml.sheet` | `.xlsx` |
| `pdf` | `application/pdf` | `.pdf` |

The filename is built server-side and sent in the `Content-Disposition` header, e.g.
`vendor-statement_acme-properties_acme-plumbing_2026-06-01_2026-06-30.pdf`. Read it from the header
rather than composing your own; fall back to `vendor-statement.pdf` / `.xlsx` if it is unreadable.

> **Note:** `Content-Disposition` is only readable cross-origin when the API exposes it
> (`Access-Control-Expose-Headers`). If you cannot read it, use the fallback name.

Fetch it as a blob, not JSON — see the
[app doc](../../app/property-management/finance/vendor-statement.md#export-the-statement) for the
snippet; only the URL differs.

### What the files contain

The same single statement table as the app-side export — Date / Particulars / Debit / Credit /
Balance, opened by **Balance B/F** and closed by **Total** and **Balance C/F** — laid out portrait
in the PDF and on one worksheet in Excel. See
[What the files contain](../../app/property-management/finance/vendor-statement.md#what-the-files-contain).

> **Letterhead is best-effort on this portal.** Portal routes carry no `{company}` segment, so the
> managing company is derived from the statement's own lines (via their contract, else the bill or
> voucher behind them). When a period's lines resolve to exactly one company its letterhead is
> printed; when they span several, or none can be resolved, the PDF is produced **without a
> letterhead** and the filename uses a generic `company` segment. The statement table itself is
> unaffected.

### Errors

- `422` — `format` missing or not `excel`/`pdf`. Same JSON error shape as the statement endpoint.
- No separate permission: if you can read your statement you can export it.
