# Unbilled Leases Report API

Domain: `Property Management > Reports > Tenants`

Base route:

`/api/v1/app/{company}/property-management/reports`

Unbilled Leases lists the leases **whose billing has not run**: the invoice that should have been
raised has not been. It is **read-only** and computed on the fly.

> **Read [the shared reports contract](./README.md) first.** This page documents what is specific to
> this report. The response envelope, the bucket keys, the field keys, the per-cell `{ value, color }`
> shape and the colour vocabulary live in the README and apply here unchanged.

## Endpoints

- `GET /reports/tenants/unbilled-leases` — requires `view-unbilled-leases-report`
- `GET /reports/tenants/unbilled-leases/export` — requires `export-unbilled-leases-report`. Takes a
  required `format` (`excel` | `pdf`); any other value, or none, returns `422` with a `format` error.
- Listed by [`GET /reports`](./README.md#discovering-reports--get-reports).

The two permissions are **independent**, and neither is shared with
[Ageing Report](./ageing-report.md) or [Debtors Listing](./debtors-listing.md).

## Every row is an exception

**This is an exception report, so the whole row is tinted `danger` — every row, always.**

The tint means *"this lease should have been billed"*. It does **not** rank one row against another,
and there is nothing to filter within the report: if a lease is here, it is a problem. Do not add a
"show only the serious ones" control — they are all the serious ones. An empty report is the good
outcome.

There is deliberately **no lease-status filter** either. A lease whose billing has not run is a
problem whatever its status.

## What makes a lease unbilled

Both conditions, as at `period_to`:

1. `leases.next_due_at` has passed, and
2. **no invoice has been raised for it** — no invoice on that lease created on or after
   `next_due_at`, ignoring cancelled ones.

The second condition is the one that matters, because there are two ways a lease stops being
unbilled:

- **The automatic biller runs.** It raises the invoice *and* advances `next_due_at` past today, so
  both conditions clear.
- **Someone raises an invoice by hand.** `next_due_at` does **not** move — but the invoice exists, so
  the lease still leaves the report.

Testing for the invoice covers both. **A cancelled invoice does not count**: cancelling it puts the
lease back on the report, which is correct — nothing was billed.

## Filters

The six [global filters](./README.md#global-filters) and nothing else.

| Param | Type | Meaning when omitted |
|---|---|---|
| `period_to` | `Y-m-d` | End of the current month. **The date a lease is judged overdue against.** |
| `period_from` | `Y-m-d` | Echoed only |
| other global filters | see the shared contract | all |

`header.as_at` carries the judging date on its own, so you never have to work out which end of
`period` was used.

```
GET …/reports/tenants/unbilled-leases?facility_id=1&period_to=2026-07-31
```

## Buckets

| `bucket` | `header.label` | Rows |
|---|---|---|
| `overview` | `Unbilled Leases` | one row per overdue lease, ordered by property then tenant, then a `subtotal` row |

The bucket is always present. When nothing is overdue it holds only its subtotal, reading
`0 unbilled leases`.

## Columns

| `key` | `label` | `format` | Notes |
|---|---|---|---|
| `tenant` | `Tenant` | `string` | `—` where the lease has no tenant |
| `lease_id` | `Lease` | `id` | |
| `property` | `Property` | `string` | |
| `unit` | `Unit` | `string` | **Every** space on the lease, comma-separated |
| `lease_start` | `Lease Start` | `string` | `01 Jan, 2026` |
| `lease_end` | `Lease End` | `string` | or `—` |
| `expected_rent` | `Expected Rent` | `money` | Per month, from components flagged `is_rent` |
| `expected_sc` | `Expected S/C` | `money` | Per month, from components flagged `is_service_charge` |
| `next_due_at` | `Next Due` | `string` | The date billing was due |
| `last_invoice_date` | `Last Invoice` | `string` | `—` where the lease has never been invoiced |
| `months_since_last_invoice` | `Months Since Last Invoice` | `integer` | `0` where never invoiced |

> **`months_since_last_invoice` is `0` for a lease that has never been invoiced**, because there is
> no elapsed time to measure — a large number there would read as a fact. `last_invoice_date` is `—`
> in that case, and that is the field to look at.

The subtotal's label spans the six text columns (`col_span: 6`) and carries only the expected
charges — a portfolio has no next due date.

## Summary

| Key | Meaning |
|---|---|
| `lease_count` | Leases awaiting an invoice |
| `total_expected_rent` | Rent per month across them |
| `total_expected_sc` | Service charge per month across them |
| `total_expected` | The two added together |

## Example — the bucket's rows

```jsonc
"items": [
  { "type": "normal", "background_color": "danger",
    "tenant": { "value": "Acme Traders Ltd" }, "lease_id": { "value": 412 },
    "property": { "value": "KAHAWA HOUSE" }, "unit": { "value": "3rd Flr — 3B" },
    "lease_start": { "value": "01 Jan, 2026" }, "lease_end": { "value": "31 Dec, 2028" },
    "expected_rent": { "value": 300000.00 }, "expected_sc": { "value": 80000.00 },
    "next_due_at": { "value": "01 Jun, 2026" }, "last_invoice_date": { "value": "01 May, 2026" },
    "months_since_last_invoice": { "value": 2 } },
  { "type": "subtotal", "background_color": "secondary",
    "tenant": { "value": "1 unbilled lease", "col_span": 6 },
    "expected_rent": { "value": 300000.00 }, "expected_sc": { "value": 80000.00 } }
]
```

## Export

`…/unbilled-leases/export?format=excel|pdf`, taking every filter the view endpoint takes. The
frontend exports exactly what is on screen by replaying the query string with `format` appended.
