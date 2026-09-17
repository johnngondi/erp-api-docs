# Excess Service Charge Report API

Domain: `Property Management > Reports > Landlords`

Base route:

`/api/v1/app/{company}/property-management/reports`

The Excess Service Charge report answers one question per property: **did the service charge billed
cover what was spent on service?** Each property reads as a small statement — what was billed, what was
spent, and the surplus or deficit between them. It is **read-only** and computed on the fly.

> **Read [the shared reports contract](./README.md) first.** This page documents what is specific to
> this report. The response envelope, the bucket keys, the field keys, the per-cell `{ value, color }`
> shape and the colour vocabulary live in the README and apply here unchanged.

## Endpoints

- `GET /reports/landlords/excess-service-charge` — requires `view-excess-service-charge-report`
- `GET /reports/landlords/excess-service-charge/export` — requires
  `export-excess-service-charge-report`. Takes a required `format` (`excel` | `pdf`); any other value,
  or none, returns `422` with a `format` error.
- Listed by [`GET /reports`](./README.md#discovering-reports--get-reports).

The two permissions are **independent**, and neither is shared with the
[Service Charge vs Expenses Summary](./sc-vs-expenses-summary.md) report.

## What counts as service charge

This report and [Service Charge vs Expenses Summary](./sc-vs-expenses-summary.md) read **one shared
definition**, so their totals always agree:

- **Income is billed service charge**: lease billings against lease components flagged
  `is_service_charge`. Rent and other components never appear.
- **Expenditure is classified by the expense's own category**: expenses whose own
  `expense_category_id` is the company's configured **Service Charge** category (see the
  [shared conventions](./README.md#expense-categories-by-role--rent-service-charge-service)). It is
  **never** taken from the expense type's category — one expense type carries expenses of several
  categories, so the type is not a proxy. A category counts across every expense type.
- Cancelled and pending expenses are excluded.
- **Billed, not collected.** The surplus or deficit is always billed less spent. Service charge
  collected appears once, as context on `summary_by_properties`, and never enters any arithmetic.
  Comparing billed with collected in earnest belongs to
  [billings-and-collections.md](./billings-and-collections.md).

If a company has not configured its service-charge category, nothing classifies as service charge and
the expenditure side is empty — the report does not guess.

## Filters

Only the six [global filters](./README.md#global-filters); this report adds none. `header.filters`
echoes all six, `null` included, and an omitted period resolves to the current month.

```
GET …/reports/landlords/excess-service-charge?facility_id=1&period_from=2026-07-01&period_to=2026-07-31
```

Money is shown as stored, in each property's reporting currency — nothing is converted.

## Buckets

Always in this order:

| `bucket` | `header.label` | Rows |
|---|---|---|
| `overall` | `All Properties` | the portfolio as one statement, every property's lines merged |
| `summary_by_properties` | `Summary by Property` | one row per property, then a `Total` row |
| `property-{id}` | the property's name | that property's own statement |

`property-{id}` buckets come last, sorted by property name, and each `header` adds `property` —
`{ id, name, currency }`.

## Columns

The statement buckets (`overall` and each `property-{id}`) have two columns only: `line` (string) and
`amount` (money). The shape of a statement is in its **rows**, not its columns.

`summary_by_properties` has its own five:

| Column | `key` | `format` |
|---|---|---|
| Property | `property` | string |
| SC Billed | `sc_billed` | money |
| SC Collected | `sc_collected` | money |
| SC Expenditure | `sc_expenses` | money |
| Excess / (Deficit) | `excess` | money, coloured like the closing row |

**`sc_collected` is context only.** It says how much of the service charge billed has actually come in.
**The surplus is always billed less spent** — a service charge is owed whether or not the tenant has
paid it yet — so a property can collect nothing and still show a surplus. Collected appears nowhere
else: the statements never mention it, and the
[Service Charge vs Expenses Summary](./sc-vs-expenses-summary.md) has no collections figure at all.

## The statement

Both `overall` and each `property-{id}` bucket are statements, with the same rows in the same order.
`overall` is the whole portfolio: every property's billed lines merged by lease component and every
property's expenditure lines merged by expense type.

Rows, in order:

| Row | `type` | Notes |
|---|---|---|
| `SERVICE CHARGE BILLED` | `normal` | Banner, `background_color: info` |
| one row per lease component billed | `normal` | Largest first |
| `Total Billed` | `subtotal` | |
| `SERVICE CHARGE EXPENDITURE` | `normal` | Banner, `background_color: info` |
| one row per expense type | `normal` | Largest first |
| `Total Expenditure` | `subtotal` | |
| `Excess / (Deficit)` | `grosstotal` | `Total Billed − Total Expenditure` |

**Banner rows carry `col_span: 2` and omit the `amount` key entirely** — render an empty cell, not a
zero. That is the drop-when-default convention from the shared contract, not a missing value.

### The closing row

| Excess | `color` |
|---|---|
| Positive — a surplus | `success` |
| Negative — a deficit | `danger` |
| Exactly zero | no `color` |

## Summary

Every bucket, and `data.summary`, carry the same four keys:

| Key | Meaning |
|---|---|
| `sc_billed` | Service charge billed in the period |
| `sc_collected` | Service charge collected in the period — context only |
| `sc_expenses` | Service charge expenditure in the period |
| `excess` | `sc_billed − sc_expenses`; negative is a deficit. Never computed from collections |

`summary_by_properties` adds `property_count`.

## Example response

```jsonc
{
  "data": {
    "header": {
      "property": { "id": 1, "name": "KAHAWA HOUSE", "currency": { "code": "KES", "name": "Kenyan Shilling" } },
      "period": { "from": "2026-07-01", "to": "2026-07-31" },
      "filters": { "landlord_id": null, "facility_id": 1, "facility_type_id": null,
                   "properties_status": null, "period_from": "2026-07-01", "period_to": "2026-07-31" },
      "generated_at": "2026-08-17T09:12:44+00:00"
    },
    "report": [
      {
        "bucket": "overall",
        "header": { "label": "All Properties" },
        "fields": [
          { "label": "Line", "key": "line", "format": "string", "type": "normal", "weight": "font-normal", "background_color": "none", "alignment": "left", "visible": true, "togglable": false },
          { "label": "Amount", "key": "amount", "format": "money", "type": "normal", "weight": "font-normal", "background_color": "none", "alignment": "right", "visible": true, "togglable": false }
        ],
        "items": [ /* the same statement rows as below, with every property's lines merged */ ],
        "summary": { "sc_billed": 3145600.00, "sc_collected": 1980000.00, "sc_expenses": 2733300.00, "excess": 412300.00 }
      },
      {
        "bucket": "summary_by_properties",
        "header": { "label": "Summary by Property" },
        "fields": [
          { "label": "Property", "key": "property", "format": "string", "type": "normal", "weight": "font-normal", "background_color": "none", "alignment": "left", "visible": true, "togglable": false },
          { "label": "SC Billed", "key": "sc_billed", "format": "money", "type": "normal", "weight": "font-normal", "background_color": "none", "alignment": "right", "visible": true, "togglable": false },
          { "label": "SC Collected", "key": "sc_collected", "format": "money", "type": "normal", "weight": "font-normal", "background_color": "none", "alignment": "right", "visible": true, "togglable": false },
          { "label": "SC Expenditure", "key": "sc_expenses", "format": "money", "type": "normal", "weight": "font-normal", "background_color": "none", "alignment": "right", "visible": true, "togglable": false },
          { "label": "Excess / (Deficit)", "key": "excess", "format": "money", "type": "subtotal", "weight": "font-medium", "background_color": "none", "alignment": "right", "visible": true, "togglable": false }
        ],
        "items": [
          { "property": { "value": "KAHAWA HOUSE" }, "sc_billed": { "value": 3145600.00 },
            "sc_collected": { "value": 1980000.00 }, "sc_expenses": { "value": 2733300.00 },
            "excess": { "value": 412300.00, "color": "success" }, "type": "normal" },
          { "property": { "value": "Total" }, "sc_billed": { "value": 3145600.00 },
            "sc_collected": { "value": 1980000.00 }, "sc_expenses": { "value": 2733300.00 },
            "excess": { "value": 412300.00, "color": "success" },
            "type": "subtotal", "background_color": "secondary" }
        ],
        "summary": { "sc_billed": 3145600.00, "sc_collected": 1980000.00, "sc_expenses": 2733300.00, "excess": 412300.00, "property_count": 1 }
      },
      {
        "bucket": "property-1",
        "header": { "label": "KAHAWA HOUSE", "property": { "id": 1, "name": "KAHAWA HOUSE", "currency": { "code": "KES", "name": "Kenyan Shilling" } } },
        "fields": [ /* same two columns */ ],
        "items": [
          { "line": { "value": "SERVICE CHARGE BILLED", "col_span": 2 }, "type": "normal", "background_color": "info" },
          { "line": { "value": "Service Charge" }, "amount": { "value": 2891600.00 }, "type": "normal" },
          { "line": { "value": "Water Service Charge" }, "amount": { "value": 254000.00 }, "type": "normal" },
          { "line": { "value": "Total Billed" }, "amount": { "value": 3145600.00 }, "type": "subtotal", "background_color": "secondary" },
          { "line": { "value": "SERVICE CHARGE EXPENDITURE", "col_span": 2 }, "type": "normal", "background_color": "info" },
          { "line": { "value": "Security" }, "amount": { "value": 1420000.00 }, "type": "normal" },
          { "line": { "value": "Cleaning" }, "amount": { "value": 813300.00 }, "type": "normal" },
          { "line": { "value": "Common Area Electricity" }, "amount": { "value": 500000.00 }, "type": "normal" },
          { "line": { "value": "Total Expenditure" }, "amount": { "value": 2733300.00 }, "type": "subtotal", "background_color": "secondary" },
          { "line": { "value": "Excess / (Deficit)" }, "amount": { "value": 412300.00, "color": "success" }, "type": "grosstotal", "background_color": "secondary" }
        ],
        "summary": { "sc_billed": 3145600.00, "sc_collected": 1980000.00, "sc_expenses": 2733300.00, "excess": 412300.00 }
      }
    ],
    "summary": { "sc_billed": 3145600.00, "sc_collected": 1980000.00, "sc_expenses": 2733300.00, "excess": 412300.00 }
  }
}
```

Reading the example: the property collected `1980000.00` of the `3145600.00` it billed, yet its surplus
is still `3145600.00 − 2733300.00`. The two banner rows span both columns and have no `amount` key. `Total Billed`
less `Total Expenditure` is exactly the closing row, which is `success` because it is positive. Running
[Service Charge vs Expenses Summary](./sc-vs-expenses-summary.md) over the same period returns the same
`3145600.00` billed and `2733300.00` spent.
