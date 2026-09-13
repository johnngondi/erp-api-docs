# Tenant Statements API

Domain: `Property Management > Lease Management`

Base route:

`/api/v1/app/{company}/property-management/lease-management/tenant-statements`

A tenant statement is the running ledger of what a tenant has been billed and what they have paid.
It is **read-only** and assembled on the fly from the lines written as invoices, credit notes and
receipts are processed.

> **A statement is per lease, not per tenant.** `filter[lease_id]` is required and is the statement's
> scope key. A lease pins one property, one landlord and one currency, which is what makes the
> statement — and its export — meaningful. A tenant holding two leases has two statements, each with
> its own opening balance.

## Endpoints

- `GET /tenant-statements` — the statement for one lease
- `GET /tenant-statements/export` — the same statement as a PDF or Excel download
- `GET /tenant-statements/{tenantStatement}` — a single line

---

## Get a tenant statement

`GET /api/v1/app/{company}/property-management/lease-management/tenant-statements`

Returns the opening balance, the period's dated lines (each with a running balance), the period
totals, and the closing balance.

Supported query params:

- `filter[lease_id]` — **required** (`leases.id`). The statement's scope.
- `filter[tenant_id]` — optional (`users.id`). Derived from the lease when omitted; when sent it
  must match the lease's tenant, otherwise `422`.
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

A tenant is a receivable, so a **positive balance is what the tenant owes**. Each line moves the
balance by `debit − credit`:

- Raising an invoice **debits** (raises) the balance.
- A receipt is a **credit** (lowers it).
- A credit note is a **credit** (lowers it).

> ⚠️ **This is the opposite of the [Vendor Statement](../finance/vendor-statement.md), which is a
> payable and moves by `credit − debit`.** The two pages look alike; the sign does not. Do not share
> a balance-formatting helper between them without passing the direction in.

- `brought_forward` (BBF) — net `debit − credit` for the lease **before** the period start.
- Each line's `balance` — the running balance up to and including that line (starting from BBF).
- `carried_forward` (BCF) — the closing balance; equals the last line's `balance`
  (`brought_forward + period debit − period credit`).

Example requests:

- `GET …/tenant-statements?filter[lease_id]=101`
- `GET …/tenant-statements?filter[lease_id]=101&filter[transaction_at][from]=2026-06-01&filter[transaction_at][to]=2026-06-30`

Sample response:

```json
{
  "data": {
    "lease_id": 101,
    "tenant_id": 77,
    "period": { "from": "2026-06-01", "to": "2026-06-30" },
    "brought_forward": 12000.0,
    "transactions": [
      {
        "id": 3201,
        "transaction": { "type": "FacilityInvoice", "id": 66 },
        "notes": "INV#INV-2001 - June rent",
        "debit": "15000.00000",
        "credit": "0.00000",
        "balance": 27000.0,
        "status": { "value": "confirmed", "color": "success" },
        "transaction_at": {
          "raw": "2026-06-01T00:00:00.000000Z",
          "formatted": "01 Jun, 2026",
          "diff": "3 weeks ago"
        }
      },
      {
        "id": 3244,
        "transaction": { "type": "FacilityReceipt", "id": 412 },
        "notes": "RCT#412 - Payment for INV#INV-2001",
        "debit": "0.00000",
        "credit": "15000.00000",
        "balance": 12000.0,
        "status": { "value": "confirmed", "color": "success" },
        "transaction_at": {
          "raw": "2026-06-18T00:00:00.000000Z",
          "formatted": "18 Jun, 2026",
          "diff": "5 days ago"
        }
      }
    ],
    "totals": { "debit": 15000.0, "credit": 15000.0 },
    "carried_forward": 12000.0
  }
}
```

Field notes (`TenantStatementResource`):

| Field | Type | Notes |
|---|---|---|
| `transaction.type` | string\|null | Source model short name: `FacilityInvoice`, `FacilityReceipt`, or `FacilityCreditNote`. |
| `transaction.id` | integer\|null | Id of that source record. |
| `notes` | string | Human-readable line description. |
| `debit` / `credit` | string | Decimal (5 dp), currency units. |
| `balance` | number | Running balance up to and including this line. |
| `status` | object | `{ value, color }` — `confirmed` (success) or `cancelled` (danger). |
| `transaction_at` | object | `{ raw, formatted, diff }`. |

> **Cancelled lines are included.** `status` is not filtered and cancelled lines are still summed
> into the balances. Render the `status` chip so a cancelled line is visible on screen; it is shown
> as a column in the export for the same reason.

### ⚠️ Changes from the previous version of this endpoint

This endpoint previously returned a paginated transaction list with a single overall balance. If you
are updating an existing screen:

- **Pagination is gone.** `per_page` and `page` are no longer accepted and the response is no longer
  wrapped in `{ data: [...], links, meta }` — `transactions` is a plain array. A running balance
  needs the whole ordered period, so the statement is returned whole (the same reasoning as the
  vendor statement).
- **Root-level `balance` is gone**, replaced by `brought_forward` / `carried_forward` plus a
  per-line `balance`.
- **`filter[transaction_at]` now actually applies.** It was previously accepted and silently
  ignored, so every request returned the lease's entire history. Screens that relied on that will
  now see only the current month unless they send a range.
- **`filter[lease_id]` is required** and `filter[tenant_id]` is now optional.
- Lines are ordered **oldest first**; each gains `balance` and `transaction`.

---

## Export the statement

`GET /api/v1/app/{company}/property-management/lease-management/tenant-statements/export`

Download the statement above as a file. **It accepts every filter the statement itself accepts** —
same param names, same defaults — plus a required `format`. The statement is regenerated
server-side from those filters, so what you download is always what the table is showing: export by
**replaying the current query string with `format` appended**.

- `format` — **required**, `excel` or `pdf`. Anything else ⇒ `422` on `format`.
- `filter[lease_id]` — **required**, as on the statement itself.
- `filter[tenant_id]`, `filter[transaction_at]` — optional, exactly as above.

Example:

```
GET …/tenant-statements/export?format=pdf&filter[lease_id]=101&filter[transaction_at][from]=2026-06-01&filter[transaction_at][to]=2026-06-30
```

### Response

**Not JSON** — a binary file with `Content-Disposition: attachment`:

| `format` | `Content-Type` | Extension |
|---|---|---|
| `excel` | `application/vnd.openxmlformats-officedocument.spreadsheetml.sheet` | `.xlsx` |
| `pdf` | `application/pdf` | `.pdf` |

The filename is built server-side and sent in the `Content-Disposition` header, e.g.
`tenant-statement_acme-properties_jane-holdings_riverside-court_jane-tenant_2026-06-01_2026-06-30.pdf`
(company, then the landlord and property resolved from the lease, then the tenant). Read it from the
header rather than composing your own; fall back to `tenant-statement.pdf` / `.xlsx` if it is
unreadable.

> **Note:** `Content-Disposition` is only readable cross-origin when the API exposes it
> (`Access-Control-Expose-Headers`). If you cannot read it, use the fallback name.

Fetch as a blob, not JSON:

```js
// The same query string the statement table is already using.
const qs = new URLSearchParams(currentStatementQuery)
qs.set('format', 'pdf')

const res = await fetch(`${base}/lease-management/tenant-statements/export?${qs}`, {
  headers: { Accept: '*/*' },
})

if (!res.ok) {
  // Validation errors still come back as JSON.
  throw new Error((await res.json())?.message ?? 'Export failed')
}

const blob = await res.blob()
const name = res.headers.get('Content-Disposition')?.match(/filename="(.+?)"/)?.[1]
  ?? 'tenant-statement.pdf'
```

### What the files contain

Both formats render the same single statement table:

| Column | Source |
|---|---|
| Date | the line's `transaction_at`, formatted `01 Jun, 2026` |
| Particulars | the line's `notes` |
| Status | `confirmed` / `cancelled` |
| Debit | the line's `debit` |
| Credit | the line's `credit` |
| Balance | the line's running `balance` |

It is opened by a **Balance B/F** row (`brought_forward`) and closed by a **Total** row (the period
`totals`) and a **Balance C/F** row (`carried_forward`). B/F and Total are bold; C/F is bold and
tinted. Amounts carry the property's reporting currency code.

- **PDF** — portrait A4, flowing onto further pages for a long period. It opens with the company
  letterhead (logo, name, tagline, contact details) and a panel naming the statement: **Landlord**,
  **Property** and **Tenant** on the left — all resolved from the lease, with the lease appended to
  the tenant (`Jane Tenant - Lease #101`) — and **Report** / **Period** on the right.
- **Excel** — a single worksheet, same rows and columns, no letterhead. The filename is what
  identifies it.

### Errors

- `422` — `format` missing or not `excel`/`pdf`, `filter[lease_id]` missing, or a `filter[tenant_id]`
  that does not match the lease's tenant. Same JSON error shape as the statement endpoint.
- No separate export permission: **anyone who can read the statement can export it.**

---

## Show a statement line

`GET /api/v1/app/{company}/property-management/lease-management/tenant-statements/{tenantStatement}`

A single line, with its tenant and lease loaded. Unchanged.

```json
{
  "data": {
    "id": 3201,
    "tenant": { "id": 77, "name": "Jane Tenant" },
    "lease": { "id": 101 },
    "transaction": { "type": "FacilityInvoice", "id": 66 },
    "notes": "INV#INV-2001 - June rent",
    "debit": "15000.00000",
    "credit": "0.00000",
    "status": { "value": "confirmed", "color": "success" },
    "transaction_at": {
      "raw": "2026-06-01T00:00:00.000000Z",
      "formatted": "01 Jun, 2026",
      "diff": "3 weeks ago"
    }
  }
}
```

A line fetched on its own carries no `balance` — a running balance only means anything inside an
ordered period, so it is present on the list endpoint only.
