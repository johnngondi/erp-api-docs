# Property Status Report API

Domain: `Property Management > Reports > Landlords`

Base route:

`/api/v1/app/{company}/property-management/reports`

The Property Status report is a **snapshot of each managed property**: its units and occupancy, what
it billed and collected, whether it is on budget, and the state of its contract. It is read at three
levels — the whole portfolio, every property, and each landlord's properties. It is **read-only** and
computed on the fly.

> **Read [the shared reports contract](./README.md) first.** This page only documents what is specific
> to this report — its three bucket levels and, above all, its **two colour rules**. The response
> envelope, the bucket keys, the field keys, the per-cell `{ value, color }` shape, and the colour/format
> vocabulary all live in the README and apply here unchanged.

## Endpoints

- `GET /reports/landlords/property-status` — requires `view-property-status-report`
- `GET /reports/landlords/property-status/export` — requires `export-property-status-report`. Takes a
  required `format` (`excel` | `pdf`) plus the same filters as the report and downloads the generated
  file — see [the shared contract](./README.md#permissions--export). Any other `format`, or none,
  returns `422` with a `format` error.
- Listed by [`GET /reports`](./README.md#discovering-reports--get-reports).

The two permissions are **independent**: neither grants the other.

## Filters

Only the six [global filters](./README.md#global-filters) — this report adds none. All are optional
and flat; `header.filters` echoes all six, `null` included, and an omitted period resolves to the
current month in `header.period`.

```
GET …/reports/landlords/property-status?landlord_id=4&facility_type_id=2&properties_status=active&period_from=2026-07-01&period_to=2026-07-31
```

Money is shown as stored, in each property's reporting currency — nothing is converted.
`header.currency` names the currency the properties share, or is `null` when they differ; for a single
property it sits on `header.property.currency` instead.

## Buckets

Always in this order:

| `bucket` | `header.label` | Rows |
|---|---|---|
| `overall` | `Portfolio Health` | one `grosstotal` row, `Portfolio`, for the whole portfolio |
| `all-properties` | `All Properties` | one row per property, sorted by name, then a `subtotal` row |
| `landlord-{id}` | the landlord's name | one per landlord, sorted by landlord name: that landlord's properties only, then a `subtotal` row |

- A `landlord-{id}` bucket `header` adds `landlord` — `{ id, name }`. A property with no landlord falls
  under `landlord-0`, `Unassigned`.
- Every bucket declares the same columns.
- The `grosstotal` and `subtotal` rows open with a `property` cell spanning `property`, `type_` and
  `landlord` (`col_span: 3`), so those keys are absent on them.

## Columns

| Column | `key` | `format` | Value |
|---|---|---|---|
| Property | `property` | string | The property's name |
| Type | `type_` | string | The property type |
| Landlord | `landlord` | string | The property's landlord |
| Total Units | `total_units` | integer | Every space on the property |
| Occupied | `occupied` | integer | Spaces a lease covered at any point in the period |
| Vacant | `vacant` | integer | `total_units − occupied` |
| Occupancy Rate % | `occupancy_rate` | string | `occupied ÷ total_units`, e.g. `93.55 %` — [fixed bands](#rule-1--fixed-bands-occupancy-and-collection) |
| Billing | `billing` | money | Billed to tenants in the period |
| Collections | `collections` | money | Confirmed collections in the period |
| % Collection | `collection_percent` | string | `collections ÷ billing` — [fixed bands](#rule-1--fixed-bands-occupancy-and-collection) |
| Income Realisation | `income_realisation` | string | `ok`, `over` or `under` — [budget rule](#rule-2--budget-judgement-realisation-and-utilisation) |
| Expense Utilisation | `expense_utilisation` | string | `ok`, `over` or `under` — [budget rule](#rule-2--budget-judgement-realisation-and-utilisation) |
| Contract Status | `contract_status` | string | The management contract's status, or `—` with no contract |
| Property Status | `property_status` | string | The property's own status |

> **The key is `type_`, not `type`.** Every row carries its own `type` (`normal` / `subtotal` /
> `grosstotal`), so the property-type column is keyed `type_`. Read columns from `fields` as always.

`contract_status` treats an `active` contract past its end date as `expired`, the same as the
[Management Contracts report](./management-contracts.md#status).

## The two colour rules

This report colours cells by **two different rules**. They are not interchangeable, and the frontend
must **render the `color` the payload returns** — never compute a colour from the percentage or the
status word.

### Rule 1 — fixed bands: occupancy and collection

`occupancy_rate` and `collection_percent` use fixed bands on the rate:

| Rate | `color` |
|---|---|
| Above 80% | `success` |
| 50% to 80%, both included | `warning` |
| Below 50% | `danger` |

- Exactly `80.00 %` is `warning`, not `success`. Exactly `50.00 %` is `warning`, not `danger`.
- A rate with **nothing to measure** — no units, or nothing billed — reads `0.00 %` and carries **no
  `color`**. An empty property is not reported as failing.
- The same bands apply to every report that colours occupancy or collection.

### Rule 2 — budget judgement: realisation and utilisation

`income_realisation` and `expense_utilisation` are **not** banded. They are judged server-side by the
same rule as the [Facility Budget report](./facility-budget.md), so the two reports always agree on
whether a property is at risk:

1. **Time-prorated.** A property is judged against the part of its budget that should have been
   realised by now, not the full budget.
2. **Per-property tolerance.** Each property has its own at-risk threshold (default 10%). Actual more
   than that threshold above the expectation is `over`; more than that below is `under`; anything
   within is `ok`. Two properties with identical figures can therefore get different verdicts.
3. **Coloured by favourability**, so the same word means different colours on each side:

| Verdict | `income_realisation` | `expense_utilisation` |
|---|---|---|
| `over` | `success` — earned more than planned | `danger` — spent more than planned |
| `under` | `danger` — an income shortfall | `success` — an underspend |
| `ok` | **no `color` key** | **no `color` key** |

An `ok` cell omits `color` entirely rather than sending `"none"` — per the shared contract an absent
colour means none, so it is not a missing value.

## Totals

- **`subtotal` rows** (in `all-properties` and each `landlord-{id}`) total units, billing and
  collections, and **recompute** `occupancy_rate` and `collection_percent` from those totals — never
  the average of the rows — then colour them by Rule 1. They carry **no** budget verdict: that belongs
  to each property's own threshold.
- **The `overall` row** totals the whole portfolio the same way and **does** judge
  `income_realisation` and `expense_utilisation`, on the summed budgets and actuals against the
  company-wide threshold, since no single property's threshold describes the portfolio.

For example, a property with 9 of 9 units let and one with 0 of 1 total `90.00 %`, not the `50.00 %`
average of their rates.

## Summary

| Key | Where | Meaning |
|---|---|---|
| `property_count` | every bucket, and `data.summary` | Properties in the bucket |
| `occupancy_rate` | `overall`, and `data.summary` | Portfolio occupancy as a number, e.g. `81.76` |
| `collection_percent` | `overall`, and `data.summary` | Portfolio collection as a number, e.g. `47.21` |

## Example response

```jsonc
{
  "data": {
    "header": {
      "property": null,
      "period": { "from": "2026-07-01", "to": "2026-07-31" },
      "filters": { "landlord_id": null, "facility_id": null, "facility_type_id": null,
                   "properties_status": null, "period_from": "2026-07-01", "period_to": "2026-07-31" },
      "generated_at": "2026-08-17T09:12:44+00:00",
      "currency": { "code": "KES", "name": "Kenyan Shilling" }
    },
    "report": [
      {
        "bucket": "overall",
        "header": { "label": "Portfolio Health" },
        "fields": [ /* the fourteen columns above */ ],
        "items": [
          { "property": { "value": "Portfolio", "col_span": 3 },
            "total_units": { "value": 148 }, "occupied": { "value": 121 }, "vacant": { "value": 27 },
            "occupancy_rate": { "value": "81.76 %", "color": "success" },
            "billing": { "value": 22476165.27 }, "collections": { "value": 10611417.55 },
            "collection_percent": { "value": "47.21 %", "color": "danger" },
            "income_realisation": { "value": "under", "color": "danger" },
            "expense_utilisation": { "value": "ok" },
            "type": "grosstotal", "background_color": "secondary" }
        ],
        "summary": { "property_count": 5, "occupancy_rate": 81.76, "collection_percent": 47.21 }
      },
      {
        "bucket": "all-properties",
        "header": { "label": "All Properties" },
        "fields": [ /* same columns */ ],
        "items": [
          { "property": { "value": "KAHAWA HOUSE" }, "type_": { "value": "Commercial" },
            "landlord": { "value": "Clementine Ndibo Holdings" },
            "total_units": { "value": 62 }, "occupied": { "value": 58 }, "vacant": { "value": 4 },
            "occupancy_rate": { "value": "93.55 %", "color": "success" },
            "billing": { "value": 11100472.85 }, "collections": { "value": 5426489.44 },
            "collection_percent": { "value": "48.89 %", "color": "danger" },
            "income_realisation": { "value": "under", "color": "danger" },
            "expense_utilisation": { "value": "under", "color": "success" },
            "contract_status": { "value": "active" }, "property_status": { "value": "active" },
            "type": "normal" },
          { "property": { "value": "Total", "col_span": 3 },
            "total_units": { "value": 148 }, "occupied": { "value": 121 }, "vacant": { "value": 27 },
            "occupancy_rate": { "value": "81.76 %", "color": "success" },
            "billing": { "value": 22476165.27 }, "collections": { "value": 10611417.55 },
            "collection_percent": { "value": "47.21 %", "color": "danger" },
            "type": "subtotal", "background_color": "secondary" }
        ],
        "summary": { "property_count": 5 }
      },
      {
        "bucket": "landlord-4",
        "header": { "label": "Clementine Ndibo Holdings", "landlord": { "id": 4, "name": "Clementine Ndibo Holdings" } },
        "fields": [ /* same columns */ ],
        "items": [ /* that landlord's properties, then a subtotal row */ ],
        "summary": { "property_count": 2 }
      }
    ],
    "summary": { "property_count": 5, "occupancy_rate": 81.76, "collection_percent": 47.21 }
  }
}
```

Reading the example: KAHAWA HOUSE's `93.55 %` occupancy is `success` under Rule 1, while its
`48.89 %` collection is `danger`. Its income shortfall is `danger` and its expense underspend is
`success` under Rule 2 — the same verdict word, opposite colours. The portfolio's `expense_utilisation`
is `ok` and so has no `color`.
