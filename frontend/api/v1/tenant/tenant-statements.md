# Tenant Statements API

Domain: `Tenant Portal`

Base route:

`/api/v1/tenant/tenant-statements`

The logged-in tenant's own statement: a running ledger of what they have been billed and what they
have paid. It is **read-only** and assembled from the same lines as the app-side statement — see the
[app Tenant Statements doc](../app/property-management/lease-management/tenant-statements.md).

> **A statement is per lease.** `filter[lease_id]` is required and is the statement's scope key. A
> tenant holding two leases has two statements, each with its own opening balance. List the tenant's
> leases first and let them pick one.

## Endpoints

- `GET /tenant-statements` — the statement for one of my leases
- `GET /tenant-statements/export` — the same statement as a PDF or Excel download
- `GET /tenant-statements/{tenantStatement}` — a single line

---

## Get my statement

`GET /api/v1/tenant/tenant-statements`

The tenant is **inferred from the authenticated user** — there is no `tenant_id` param. Returns the
opening balance, the period's dated lines (each with a running balance), the period totals, and the
closing balance.

Supported query params:

- `filter[lease_id]` — **required** (`leases.id`). Must be a lease belonging to the authenticated
  tenant, otherwise `404`.
- `filter[transaction_at]` — optional date range over the line date. Accepts an inclusive `from`/`to`
  pair (dates, `Y-m-d`), either bound optional, in bracket form
  (`filter[transaction_at][from]=…&filter[transaction_at][to]=…`) or CSV form
  (`filter[transaction_at]=2026-06-01,2026-06-30`). **Defaults to the current month** when omitted.

There is no pagination, sort, or include — the statement is returned whole for the resolved period.

### Balance convention

A tenant is a receivable, so a **positive balance is what you owe**. Each line moves the balance by
`debit − credit`: invoices debit it up; receipts and credit notes draw it down.

- `brought_forward` (BBF) — net balance **before** the period start.
- Each line's `balance` — running balance up to and including that line.
- `carried_forward` (BCF) — closing balance (`brought_forward + period debit − period credit`).

Sample response:

```json
{
  "data": {
    "lease_id": 101,
    "period": { "from": "2026-06-01", "to": "2026-06-30" },
    "brought_forward": 12000.0,
    "transactions": [
      {
        "id": 3201,
        "lease_id": 101,
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
      }
    ],
    "totals": { "debit": 15000.0, "credit": 15000.0 },
    "carried_forward": 12000.0
  }
}
```

Field notes match the app-side resource — see the
[app doc](../app/property-management/lease-management/tenant-statements.md#get-a-tenant-statement).

### ⚠️ Changes from the previous version of this endpoint

- **Pagination is gone** — `transactions` is a plain array, not a `{ data, links, meta }` wrapper.
- **Root-level `balance` is gone**, replaced by `brought_forward` / `carried_forward` plus a
  per-line `balance`.
- **`filter[transaction_at]` now actually applies.** It was accepted and silently ignored before, so
  every request returned the whole history; it now defaults to the current month.
- **`filter[lease_id]` is required** (it was previously optional in practice).
- Lines are ordered **oldest first**, not newest-first.

---

## Export my statement

`GET /api/v1/tenant/tenant-statements/export`

Download the statement above as a file. **It accepts every filter the statement itself accepts** —
same param names, same defaults — plus a required `format`. The statement is regenerated
server-side, so export by **replaying the current query string with `format` appended**.

- `format` — **required**, `excel` or `pdf`. Anything else ⇒ `422` on `format`.
- `filter[lease_id]` — **required**, as on the statement itself.
- `filter[transaction_at]` — optional, exactly as above.

Example:

```
GET /api/v1/tenant/tenant-statements/export?format=pdf&filter[lease_id]=101&filter[transaction_at][from]=2026-06-01&filter[transaction_at][to]=2026-06-30
```

### Response

**Not JSON** — a binary file with `Content-Disposition: attachment`:

| `format` | `Content-Type` | Extension |
|---|---|---|
| `excel` | `application/vnd.openxmlformats-officedocument.spreadsheetml.sheet` | `.xlsx` |
| `pdf` | `application/pdf` | `.pdf` |

The filename is built server-side and sent in the `Content-Disposition` header, e.g.
`tenant-statement_acme-properties_riverside-court_jane-tenant_2026-06-01_2026-06-30.pdf`. Read it
from the header rather than composing your own; fall back to `tenant-statement.pdf` / `.xlsx` if it
is unreadable.

> **Note:** `Content-Disposition` is only readable cross-origin when the API exposes it
> (`Access-Control-Expose-Headers`). If you cannot read it, use the fallback name.

Fetch it as a blob, not JSON — see the
[app doc](../app/property-management/lease-management/tenant-statements.md#export-the-statement) for
the snippet; only the URL differs.

### What the files contain

The same single statement table as the app-side export — Date / Particulars / Status / Debit /
Credit / Balance, opened by **Balance B/F** and closed by **Total** and **Balance C/F** — laid out
portrait in the PDF and on one worksheet in Excel. See
[What the files contain](../app/property-management/lease-management/tenant-statements.md#what-the-files-contain).

The PDF carries the managing company's letterhead and names the **Landlord**, **Property** and
**Tenant** the lease resolves to. Portal routes carry no `{company}` segment, so the company is
derived from the statement's own lines; in the rare case it cannot be resolved the PDF is produced
without a letterhead and the filename uses a generic `company` segment. The table itself is
unaffected.

### Errors

- `422` — `format` missing or not `excel`/`pdf`, or `filter[lease_id]` missing.
- `404` — the lease does not belong to the authenticated tenant.
- No separate permission: if you can read your statement you can export it.

---

## Show a statement line

`GET /api/v1/tenant/tenant-statements/{tenantStatement}`

```json
{
  "data": {
    "id": 3201,
    "lease_id": 101,
    "transaction": { "type": "FacilityInvoice", "id": 66 },
    "notes": "INV#INV-2001 - June rent",
    "debit": "15000.00000",
    "credit": "0.00000",
    "status": { "value": "confirmed", "color": "success" },
    "transaction_at": {
      "raw": "2026-06-01T00:00:00.000000Z",
      "formatted": "01 Jun, 2026",
      "diff": "3 weeks ago"
    },
    "created_at": {
      "raw": "2026-06-01T08:00:00.000000Z",
      "formatted": "01 Jun, 2026",
      "diff": "3 weeks ago"
    }
  }
}
```

If the line does not belong to the authenticated tenant, the API returns `404`. A line fetched on its
own carries no `balance` — a running balance only means anything inside an ordered period, so it is
present on the list endpoint only.
