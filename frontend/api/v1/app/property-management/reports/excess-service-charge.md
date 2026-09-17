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
- **Billed only.** Neither report compares billed with collected, and **no collections figure appears
  anywhere** in the payload. That comparison belongs to
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

| `bucket` | `header.label` | Rows |
|---|---|---|
| `overall` | `All Properties` | the portfolio's closing `Excess / (Deficit)` row, alone |
| `property-{id}` | the property's name | the full statement (below) |

`property-{id}` buckets come after `overall`, sorted by property name, and each `header` adds
`property` — `{ id, name, currency }`.

## Columns

Two columns only: `line` (string) and `amount` (money). The shape of the statement is in its **rows**,
not its columns.

## The statement

A property bucket's rows, in order:

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

Every bucket, and `data.summary`, carry the same three keys:

| Key | Meaning |
|---|---|
| `sc_billed` | Service charge billed in the period |
| `sc_expenses` | Service charge expenditure in the period |
| `excess` | `sc_billed − sc_expenses`; negative is a deficit |

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
        "items": [
          { "line": { "value": "Excess / (Deficit)" }, "amount": { "value": 412300.00, "color": "success" },
            "type": "grosstotal", "background_color": "secondary" }
        ],
        "summary": { "sc_billed": 3145600.00, "sc_expenses": 2733300.00, "excess": 412300.00 }
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
        "summary": { "sc_billed": 3145600.00, "sc_expenses": 2733300.00, "excess": 412300.00 }
      }
    ],
    "summary": { "sc_billed": 3145600.00, "sc_expenses": 2733300.00, "excess": 412300.00 }
  }
}
```

Reading the example: the two banner rows span both columns and have no `amount` key. `Total Billed`
less `Total Expenditure` is exactly the closing row, which is `success` because it is positive. Running
[Service Charge vs Expenses Summary](./sc-vs-expenses-summary.md) over the same period returns the same
`3145600.00` billed and `2733300.00` spent.
