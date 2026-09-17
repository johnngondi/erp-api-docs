# Service Charge vs Expenses Summary Report API

Domain: `Property Management > Reports > Landlords`

Base route:

`/api/v1/app/{company}/property-management/reports`

The Service Charge vs Expenses Summary tracks **service charge billed against service charge spent,
per property, over time**. Each period column shows what was billed, what was spent, and the variance
between them, so a property drifting into deficit is visible period by period. It is **read-only** and
computed on the fly.

> **Read [the shared reports contract](./README.md) first.** This page documents what is specific to
> this report. The response envelope, the bucket keys, the field keys, the per-cell `{ value, color }`
> shape and the colour vocabulary live in the README and apply here unchanged.

## Endpoints

- `GET /reports/landlords/sc-vs-expenses-summary` — requires `view-sc-vs-expenses-summary-report`
- `GET /reports/landlords/sc-vs-expenses-summary/export` — requires
  `export-sc-vs-expenses-summary-report`. Takes a required `format` (`excel` | `pdf`); any other value,
  or none, returns `422` with a `format` error.
- Listed by [`GET /reports`](./README.md#discovering-reports--get-reports).

The two permissions are **independent**, and neither is shared with the
[Excess Service Charge](./excess-service-charge.md) report.

## What counts as service charge

This report and [Excess Service Charge](./excess-service-charge.md) read **one shared definition**, so
their totals always agree:

- **Income is billed service charge**: lease billings against lease components flagged
  `is_service_charge`. Rent and other components never appear.
- **Expenditure is classified by the expense's own category**: expenses whose own
  `expense_category_id` is the company's configured **Service Charge** category (see the
  [shared conventions](./README.md#expense-categories-by-role--rent-service-charge-service)). It is
  **never** taken from the expense type's category — one expense type carries expenses of several
  categories, so the type is not a proxy. A category counts across every expense type.
- Cancelled and pending expenses are excluded.
- **Billed only.** Neither report compares billed with collected, and **no collections figure appears
  anywhere** in the payload.

## Filters

The six [global filters](./README.md#global-filters) plus one:

| Param | Type | Meaning when omitted |
|---|---|---|
| six global filters | see the shared contract | all / current month |
| `report_cycle` | `weekly` \| `monthly` \| `quarterly` \| `annually` | **`monthly`** |

`header.filters` echoes all seven, `null` included — including `report_cycle`, which is `null` when the
default was used.

```
GET …/reports/landlords/sc-vs-expenses-summary?period_from=2026-01-01&period_to=2026-06-30&report_cycle=quarterly
```

Money is shown as stored, in each property's reporting currency — nothing is converted.

## Buckets

| `bucket` | `header.label` | Rows |
|---|---|---|
| `overall` | `All Properties` | one row per property, then a `Total` `subtotal` row |
| `property-{id}` | the property's name | that property's row, alone |

`property-{id}` buckets follow `overall`, sorted by property name, and each `header` adds `property` —
`{ id, name, currency }`. Every bucket declares the same columns.

## Columns — generated from the cycle

The period is sliced into the cycle's calendar periods, and **each period contributes three columns**:

| Column | `key` | `format` |
|---|---|---|
| Property | `property` | string |
| *(per period)* Billed | `period_{i}_billed` | money |
| *(per period)* Expenses | `period_{i}_expenses` | money |
| *(per period)* Variance | `period_{i}_variance` | money |
| Total Variance | `total_variance` | money, `grosstotal` |

- **Period columns are `togglable: true`**, so the user can hide periods.
- Each period column carries `period_from` and `period_to`, so a column's exact window needs no
  parsing. Its `label` reads e.g. `Q1 2026 Billed` or `Jan 2026 Expenses`.
- Read the columns from `fields`; never assume how many there are.

How many groups you get depends only on `report_cycle`:

| Period requested | `report_cycle` | Period groups | Columns |
|---|---|---|---|
| 1 Jan – 30 Jun 2026 | `quarterly` | 2 | 1 + 6 + 1 |
| 1 Jan – 30 Jun 2026 | `monthly` (default) | 6 | 1 + 18 + 1 |
| 1 Jan – 30 Jun 2026 | `annually` | 1 | 1 + 3 + 1 |

Calendar periods are clamped to the requested window, so an edge period only counts activity inside it.

## Variance and colour

**Variance is `billed − expenses`** for the period, and `total_variance` is the same across the whole
window:

| Variance | `color` on a property row |
|---|---|
| Positive — billed more than spent | `success` |
| Negative — spent more than billed | `danger` |
| Exactly zero | no `color` |

The closing `Total` row is not coloured: its `subtotal` row type already carries the emphasis.

## Summary

Every bucket, and `data.summary`, carry the same three keys:

| Key | Meaning |
|---|---|
| `total_billed` | Service charge billed across the bucket |
| `total_expenses` | Service charge expenditure across the bucket |
| `total_variance` | `total_billed − total_expenses`; negative is a shortfall |

These match the [Excess Service Charge](./excess-service-charge.md) report's `sc_billed`,
`sc_expenses` and `excess` for the same scope and period.

## Example response

```jsonc
{
  "data": {
    "header": {
      "property": null,
      "period": { "from": "2026-01-01", "to": "2026-06-30" },
      "filters": { "landlord_id": null, "facility_id": null, "facility_type_id": null,
                   "properties_status": null, "period_from": "2026-01-01", "period_to": "2026-06-30",
                   "report_cycle": "quarterly" },
      "generated_at": "2026-08-17T09:12:44+00:00",
      "currency": { "code": "KES", "name": "Kenyan Shilling" }
    },
    "report": [
      {
        "bucket": "overall",
        "header": { "label": "All Properties" },
        "fields": [
          { "label": "Property", "key": "property", "format": "string", "type": "normal", "weight": "font-normal", "background_color": "none", "alignment": "left", "visible": true, "togglable": false },
          { "label": "Q1 2026 Billed", "key": "period_0_billed", "format": "money", "type": "normal", "weight": "font-normal", "background_color": "none", "alignment": "right", "visible": true, "togglable": true, "period_from": "2026-01-01", "period_to": "2026-03-31" },
          { "label": "Q1 2026 Expenses", "key": "period_0_expenses", "format": "money", "type": "normal", "weight": "font-normal", "background_color": "none", "alignment": "right", "visible": true, "togglable": true, "period_from": "2026-01-01", "period_to": "2026-03-31" },
          { "label": "Q1 2026 Variance", "key": "period_0_variance", "format": "money", "type": "normal", "weight": "font-normal", "background_color": "none", "alignment": "right", "visible": true, "togglable": true, "period_from": "2026-01-01", "period_to": "2026-03-31" },
          /* …the same three for Q2 2026 as period_1_*… */
          { "label": "Total Variance", "key": "total_variance", "format": "money", "type": "grosstotal", "weight": "font-bold", "background_color": "none", "alignment": "right", "visible": true, "togglable": false }
        ],
        "items": [
          { "property": { "value": "KAHAWA HOUSE" },
            "period_0_billed": { "value": 9134500.00 }, "period_0_expenses": { "value": 8210300.00 },
            "period_0_variance": { "value": 924200.00, "color": "success" },
            "period_1_billed": { "value": 9436800.00 }, "period_1_expenses": { "value": 9899900.00 },
            "period_1_variance": { "value": -463100.00, "color": "danger" },
            "total_variance": { "value": 461100.00, "color": "success" },
            "type": "normal" },
          { "property": { "value": "Total" },
            "period_0_billed": { "value": 9134500.00 }, "period_0_expenses": { "value": 8210300.00 }, "period_0_variance": { "value": 924200.00 },
            "period_1_billed": { "value": 9436800.00 }, "period_1_expenses": { "value": 9899900.00 }, "period_1_variance": { "value": -463100.00 },
            "total_variance": { "value": 461100.00 },
            "type": "subtotal", "background_color": "secondary" }
        ],
        "summary": { "total_billed": 18571300.00, "total_expenses": 18110200.00, "total_variance": 461100.00 }
      },
      {
        "bucket": "property-1",
        "header": { "label": "KAHAWA HOUSE", "property": { "id": 1, "name": "KAHAWA HOUSE", "currency": { "code": "KES", "name": "Kenyan Shilling" } } },
        "fields": [ /* the same period columns */ ],
        "items": [ /* that property's row */ ],
        "summary": { "total_billed": 18571300.00, "total_expenses": 18110200.00, "total_variance": 461100.00 }
      }
    ],
    "summary": { "total_billed": 18571300.00, "total_expenses": 18110200.00, "total_variance": 461100.00 }
  }
}
```

Reading the example: six months read quarterly give two period groups of three columns each. Q1 is a
surplus and `success`; Q2 is a shortfall and `danger`; the total is still positive. Sending
`report_cycle=monthly` over the same window returns six groups instead, and the totals do not change.
