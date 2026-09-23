# Debtors Listing Report API

Domain: `Property Management > Reports > Tenants`

Base route:

`/api/v1/app/{company}/property-management/reports`

The Debtors Listing answers **who owes what, and whether it has gone on too long**. It is
**read-only** and computed on the fly.

> **Read [the shared reports contract](./README.md) first.** This page documents what is specific to
> this report. The response envelope, the bucket keys, the field keys, the per-cell `{ value, color }`
> shape and the colour vocabulary live in the README and apply here unchanged.

## Endpoints

- `GET /reports/tenants/debtors-listing` — requires `view-debtors-listing-report`
- `GET /reports/tenants/debtors-listing/export` — requires `export-debtors-listing-report`. Takes a
  required `format` (`excel` | `pdf`); any other value, or none, returns `422` with a `format` error.
- Listed by [`GET /reports`](./README.md#discovering-reports--get-reports).

The two permissions are **independent**, and neither is shared with
[Ageing Report](./ageing-report.md) or [Unbilled Leases](./unbilled-leases.md).

## The danger threshold is per lease, not a fixed number

A tenant's row is tinted by how far behind they are — but **"too far behind" is measured against
their own billing cycle**, read from `leases.billing_cycle`:

| `billing_cycle` | Cycle length |
|---|---|
| `biennial` | 24 months |
| `annually` | 12 months |
| `biannual` | 6 months |
| `quarterly` | 3 months |
| `monthly` | 1 month |
| `weekly` | 0.25 months |
| `daily` / `hourly` | a fraction of a month |

| Condition | Row `background_color` |
|---|---|
| `months_in_arrears` is 0 | none |
| `months_in_arrears` is not 0 | `warning` |
| `months_in_arrears` **exceeds the lease's own cycle length** | `danger` |

Two months in arrears is a **month late** for a monthly tenant and **not yet a cycle late** for a
quarterly one — so the monthly tenant is `danger` and the quarterly one is `warning`, on identical
figures. **Do not re-derive this on the frontend as "three months".** A hardcoded number mislabels
every lease that is not on that cycle. Render the `background_color` the backend returns.

`months_in_arrears` is `arrears ÷ month_rent`, **rounded up**: a tenant 3.5 months behind is into
their fourth month. It is `0` where the lease has no monthly rent to measure against.

## What counts as owed

Identical to the [Ageing Report](./ageing-report.md), because both read one shared source:

- **Outstanding is `total - paid`** per invoice, floored at nil per invoice so one overpaid invoice
  cannot cancel another's debt.
- Invoices raised **on or before `period_to`**; `pending` and `cancelled` excluded.

**`summary.total_arrears` here equals the Ageing Report's `summary.total_outstanding`** for the same
filters. That equality is tested, and it is the quickest way to tell the two are in sync.

`billed` and `collected` are the raw sums behind `arrears` — they differ from it only where an
invoice has been overpaid.

`deposit_held` is the lease's **unrefunded** deposits; a refunded deposit is no longer security.

`debt_beyond_deposit` is `max(0, arrears − deposit_held)` — what the deposit would not cover. **It
never goes negative**: a tenant whose deposit exceeds their debt sits at nil, not in credit.

## Filters

The six [global filters](./README.md#global-filters) plus one:

| Param | Type | Meaning when omitted |
|---|---|---|
| six global filters | see the shared contract | all / current month |
| `leases_status` | `active` \| `suspended` \| `terminated` | Any lease status |

```
GET …/reports/tenants/debtors-listing?facility_id=1&period_to=2026-07-31&leases_status=active
```

## Buckets

| `bucket` | `header.label` | Rows |
|---|---|---|
| `overall` | `All Properties` | one row per property with debt, then a `subtotal` Total row |
| `property-{id}` | the property's name | one row per indebted tenant, then a `subtotal` Total row |

Property buckets carry `header.property` `{ id, name }`. A property with no debt gets no bucket, and
a tenant who owes nothing gets no row.

**The two buckets have different columns** — `overall` has no `tenant`, `lease_id` or
`months_in_arrears`, because those are properties of a lease, not of a property. Read each bucket's
own `fields`, as the shared contract requires.

## Columns

**`overall`**: `property`, `month_rent`, `billed`, `collected`, `arrears`, `deposit_held`,
`debt_beyond_deposit`.

**`property-{id}`**

| `key` | `label` | `format` | Notes |
|---|---|---|---|
| `tenant` | `Tenant` | `string` | |
| `lease_id` | `Lease` | `id` | |
| `month_rent` | `Monthly Rent` | `money` | Lease item components on `is_rent` components |
| `billed` | `Billed` | `money` | |
| `collected` | `Collected` | `money` | |
| `arrears` | `Arrears` | `money` | Reconciles with the Ageing Report |
| `deposit_held` | `Deposit Held` | `money` | Unrefunded only |
| `months_in_arrears` | `Months in Arrears` | `integer` | Rounded up |
| `debt_beyond_deposit` | `Debt Beyond Deposit` | `money` | `danger` when above nil |

The subtotal's label spans `tenant` and `lease_id` (`col_span: 2`) and carries no
`months_in_arrears`.

## Summary

| Key | Meaning |
|---|---|
| `lease_count` | Leases in arrears |
| `total_billed` | Invoiced up to the as-at date |
| `total_collected` | Paid against those invoices |
| `total_arrears` | Still owed — **equals the Ageing Report's `total_outstanding`** |
| `total_deposit_held` | Unrefunded deposits held |
| `total_debt_beyond_deposit` | Arrears the deposits would not cover |

## Example — a property bucket's rows

```jsonc
"items": [
  { "type": "normal", "background_color": "danger",
    "tenant": { "value": "Acme Traders Ltd" }, "lease_id": { "value": 412 },
    "month_rent": { "value": 300000.00 }, "billed": { "value": 1060000.00 }, "collected": { "value": 0 },
    "arrears": { "value": 1060000.00 }, "deposit_held": { "value": 900000.00 },
    "months_in_arrears": { "value": 4 },
    "debt_beyond_deposit": { "value": 160000.00, "color": "danger" } },

  // Identical figures to the row below; only the billing cycle differs.
  { "type": "normal", "background_color": "warning",
    "tenant": { "value": "Njiru Hardware (quarterly)" }, "lease_id": { "value": 388 },
    "month_rent": { "value": 150000.00 }, "billed": { "value": 300000.00 }, "collected": { "value": 0 },
    "arrears": { "value": 300000.00 }, "deposit_held": { "value": 0 },
    "months_in_arrears": { "value": 2 }, "debt_beyond_deposit": { "value": 300000.00, "color": "danger" } },

  { "type": "normal", "background_color": "danger",
    "tenant": { "value": "Solo Traders (monthly)" }, "lease_id": { "value": 389 },
    "month_rent": { "value": 150000.00 }, "billed": { "value": 300000.00 }, "collected": { "value": 0 },
    "arrears": { "value": 300000.00 }, "deposit_held": { "value": 0 },
    "months_in_arrears": { "value": 2 }, "debt_beyond_deposit": { "value": 300000.00, "color": "danger" } },

  { "type": "subtotal", "background_color": "secondary",
    "tenant": { "value": "Total", "col_span": 2 },
    "month_rent": { "value": 600000.00 }, "billed": { "value": 1660000.00 }, "collected": { "value": 0 },
    "arrears": { "value": 1660000.00 }, "deposit_held": { "value": 900000.00 },
    "debt_beyond_deposit": { "value": 760000.00 } }
]
```

Njiru and Solo are two months in arrears on the same money. Njiru is `warning` because a quarterly
cycle is three months; Solo is `danger` because a monthly one is one.

## Export

`…/debtors-listing/export?format=excel|pdf`, taking every filter the view endpoint takes. The
frontend exports exactly what is on screen by replaying the query string with `format` appended.
