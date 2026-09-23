# Ageing Report API

Domain: `Property Management > Reports > Tenants`

Base route:

`/api/v1/app/{company}/property-management/reports`

The Ageing Report answers **how long the money owed has been owed**, as at a single date. It is
**read-only** and computed on the fly.

> **Read [the shared reports contract](./README.md) first.** This page documents what is specific to
> this report. The response envelope, the bucket keys, the field keys, the per-cell `{ value, color }`
> shape and the colour vocabulary live in the README and apply here unchanged.

## Endpoints

- `GET /reports/tenants/ageing-report` — requires `view-ageing-report-report`
- `GET /reports/tenants/ageing-report/export` — requires `export-ageing-report-report`. Takes a
  required `format` (`excel` | `pdf`); any other value, or none, returns `422` with a `format` error.
- Listed by [`GET /reports`](./README.md#discovering-reports--get-reports).

The two permissions are **independent**, and neither is shared with
[Debtors Listing](./debtors-listing.md) or [Unbilled Leases](./unbilled-leases.md).

## `period_from` has no effect on this report

**This is a point-in-time snapshot, not a window.** Every invoice raised on or before `period_to` is
aged into a band; nothing else exists yet. `period_from` is echoed in `header.period` and
`header.filters` because the contract echoes every filter, but **changing it alone returns exactly
the same rows**.

Two things follow for the UI:

- **Do not present a date range.** Show the as-at date. `header.as_at` carries it on its own, so you
  never have to work out which end of `period` was used.
- If the figures ever move when only `period_from` changes, something has regressed — that is a
  tested guarantee, not an incidental one.

## What counts as owed, and how it ages

- **Outstanding is `total - paid`** per invoice, computed rather than read from the stored `balance`,
  so a stale balance cannot make this report disagree with [Debtors Listing](./debtors-listing.md).
- **Pending and cancelled invoices are excluded** — not issued, not owed. A fully settled invoice
  drops out too; it has no age worth reporting.
- **Age runs from `facility_invoices.created_at`**, not from `due_at`. The question is how long the
  debt has existed, not how long it has been late.

Both reports read one shared source, so **the Debtors Listing's `total_arrears` equals this report's
`total_outstanding`** for the same filters.

### The bands

| Band | `key` | Age at the as-at date | Colour |
|---|---|---|---|
| Current | `current` | raised today (0 days) | none |
| 1–30 days | `d_1_30` | 1–30 days | none |
| 31–60 days | `d_31_60` | 31–60 days | `warning` |
| 61–90 days | `d_61_90` | 61–90 days | `warning` |
| 91+ days | `d_91_plus` | 91 days or more | `danger` |

Severity rises with age. A band whose amount is nil carries **no `color` key** — absent means `none`,
per the shared contract — and subtotal rows are never tinted: their emphasis is the row `type`.

## Filters

The six [global filters](./README.md#global-filters) plus one:

| Param | Type | Meaning when omitted |
|---|---|---|
| `period_to` | `Y-m-d` | End of the current month. **This is the as-at anchor.** |
| `period_from` | `Y-m-d` | **Ignored by this report** |
| other global filters | see the shared contract | all |
| `leases_status` | `active` \| `suspended` \| `terminated` | Any lease status |

```
GET …/reports/tenants/ageing-report?facility_id=1&period_to=2026-07-31&leases_status=active
```

## Buckets

| `bucket` | `header.label` | Rows |
|---|---|---|
| `overall` | `Portfolio Ageing Profile` | one row **per band**, then a `grosstotal` Total row |
| `properties-summary` | `Properties Summary` | one row **per indebted property**, bands as columns, then a `grosstotal` Total row |
| `property-{id}` | the property's name | one row per indebted tenant, bands as columns, then a `subtotal` Total row |

`overall` inverts the usual shape on purpose: the **bands are the rows**, so the reader sees the
shape of the debt before seeing who owes it. Its per-band amounts are the **column sums of the
property buckets** — they are folded from the same invoices, so they always agree.

`properties-summary` is that same fold cut the other way: one row per property, in the same band
columns a property bucket uses, with the property's name where a property bucket puts the tenant's.
Each row equals that property bucket's Total row, and its own Total row equals `overall`'s
grosstotal.

**`properties-summary` is omitted when `facility_id` is set.** It would hold a single row, and
`overall` already is that row.

Property buckets carry `header.property` `{ id, name }`. A property with no debt gets no bucket, and
so gets no row in `properties-summary` either.

## Columns

**`overall`**

| `key` | `label` | `format` |
|---|---|---|
| `period` | `Ageing Period` | `string` |
| `amount` | `Amount Owed` | `money` |

**`properties-summary`**

| `key` | `label` | `format` |
|---|---|---|
| `property` | `Property` | `string` |
| `status` | `Status` | `string`, tinted by the property's status |
| `current` / `d_1_30` / `d_31_60` / `d_61_90` / `d_91_plus` | `Current` / `1–30` / `31–60` / `61–90` / `91+` | `money` |
| `total` | `Total` | `money`, `type: grosstotal` |

The Total row's `property` cell carries `col_span: 2`, so the row omits `status`.

**`property-{id}`**

| `key` | `label` | `format` |
|---|---|---|
| `tenant` | `Tenant` | `string` |
| `lease_id` | `Lease` | `id` — `null` on the Total row |
| `current` / `d_1_30` / `d_31_60` / `d_61_90` / `d_91_plus` | `Current` / `1–30` / `31–60` / `61–90` / `91+` | `money` |
| `total` | `Total` | `money`, `type: grosstotal` |

## Summary

| Key | Meaning |
|---|---|
| `total_outstanding` | Everything owed as at the as-at date |
| `tenant_count` | Leases owing something |

Property buckets carry the same two, scoped to that property. `properties-summary` carries
`total_outstanding` and `property_count` (the rows above its Total).

## Example response

```jsonc
{
  "data": {
    "header": {
      "property": { "id": 1, "name": "KAHAWA HOUSE", "currency": { "code": "KES", "name": "Kenyan Shilling" } },
      "landlord": null,
      "period": { "from": "2026-07-01", "to": "2026-07-31" },
      "filters": { "landlord_id": null, "facility_id": 1, "facility_type_id": null, "properties_status": null,
                   "period_from": null, "period_to": "2026-07-31", "leases_status": "active" },
      "generated_at": "2026-08-17T09:12:44+00:00",
      "as_at": "2026-07-31"
    },
    "report": [
      {
        "bucket": "overall",
        "header": { "label": "Portfolio Ageing Profile" },
        "fields": [ /* period, amount */ ],
        "items": [
          { "type": "normal", "period": { "value": "Current" }, "amount": { "value": 400000.00 } },
          { "type": "normal", "period": { "value": "1–30 days" }, "amount": { "value": 600000.00 } },
          { "type": "normal", "period": { "value": "31–60 days" }, "amount": { "value": 540000.00, "color": "warning" } },
          { "type": "normal", "period": { "value": "61–90 days" }, "amount": { "value": 0 } },
          { "type": "normal", "period": { "value": "91+ days" }, "amount": { "value": 120000.00, "color": "danger" } },
          { "type": "grosstotal", "background_color": "secondary", "period": { "value": "Total" }, "amount": { "value": 1660000.00 } }
        ],
        "summary": { "total_outstanding": 1660000.00 }
      },
      {
        "bucket": "property-1",
        "header": { "label": "KAHAWA HOUSE", "property": { "id": 1, "name": "KAHAWA HOUSE" } },
        "fields": [ /* tenant, the five bands, total */ ],
        "items": [
          { "type": "normal", "tenant": { "value": "Acme Traders Ltd" },
            "current": { "value": 400000.00 }, "d_1_30": { "value": 0 },
            "d_31_60": { "value": 540000.00, "color": "warning" }, "d_61_90": { "value": 0 },
            "d_91_plus": { "value": 120000.00, "color": "danger" }, "total": { "value": 1060000.00 } },
          { "type": "subtotal", "background_color": "secondary", "tenant": { "value": "Total" },
            "current": { "value": 400000.00 }, "d_1_30": { "value": 600000.00 }, "d_31_60": { "value": 540000.00 },
            "d_61_90": { "value": 0 }, "d_91_plus": { "value": 120000.00 }, "total": { "value": 1660000.00 } }
        ],
        "summary": { "total_outstanding": 1660000.00, "tenant_count": 3 }
      }
    ],
    "summary": { "total_outstanding": 1660000.00, "tenant_count": 3 }
  }
}
```

## Export

`…/ageing-report/export?format=excel|pdf`, taking every filter the view endpoint takes. The frontend
exports exactly what is on screen by replaying the query string with `format` appended.
