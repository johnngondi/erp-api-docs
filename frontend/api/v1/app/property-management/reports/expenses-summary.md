# Expenses Summary Report API

Domain: `Property Management > Reports > Suppliers > Expenditure Reports`

Base route:

`/api/v1/app/{company}/property-management/reports`

The Expenses Summary shows **what was spent across the properties in scope, period by period** — once
grouped by **expense category** and once by **expense type**. It is **read-only** and computed on the fly.

> **Read [the shared reports contract](./README.md) first.** This page documents what is specific to
> this report. The response envelope, the bucket keys, the field keys, the per-cell `{ value, color }`
> shape and the colour vocabulary live in the README and apply here unchanged.

## Endpoints

- `GET /reports/suppliers/expenditure-reports/expenses-summary` — requires `view-expenses-summary-report`
- `GET /reports/suppliers/expenditure-reports/expenses-summary/export` — requires
  `export-expenses-summary-report`. Takes a required `format` (`excel` | `pdf`) plus the same filters as
  the report; any other format, or none, returns `422` with a `format` error.
- Listed by [`GET /reports`](./README.md#discovering-reports--get-reports) under the supplier
  `expenditure-reports` group, after [Property Expenses](./property-expenses.md).

The two permissions are **independent**: viewing does not imply exporting, and neither is shared with
Property Expenses.

## What is counted

- **Expense totals, tax inclusive**, from posted expenses (`facility_expenses`) whose
  `transaction_at` falls in the period. **Cancelled and pending expenses are excluded**, as in
  Property Expenses.
- **Category is the expense's own `expense_category_id`**, never its expense type's — one expense type
  can carry expenses of several categories. Expenses with no category are grouped as
  **"Uncategorized"**, expenses with no type as **"Untyped"**.
- **Both buckets are the same expenses grouped differently, so they reconcile exactly**: their closing
  `Total` rows are identical, period by period, and so are their `summary.total_expenditure`.

## Filters

The six [global filters](./README.md#global-filters) plus:

| Param | Type | Meaning when omitted |
|---|---|---|
| six global filters | see the shared contract | all / current month |
| `report_cycle` | `weekly` \| `monthly` \| `quarterly` \| `annually` | **`monthly`** |
| `expense_category_id` | `expense_categories.id` | All categories |
| `currency_id` | `currencies.id` | Each property's reporting currency (see below) |

- **`report_cycle`** sets the period columns. Any other value returns `422` with a `report_cycle` error.
- **`expense_category_id`** narrows **both** buckets to that category: the category bucket then has one
  row, and the type bucket lists only the types that carried expenses of that category.
- **`currency_id`** is optional. When given, every figure is converted to it. When omitted, each
  expense is shown in its property's reporting currency — `header.currency` states it when every
  property in scope shares one, and is `null` when they do not (then pass `currency_id` to get figures
  that add up across currencies).

`header.filters` echoes all nine; `report_cycle` is echoed as the effective cycle (`monthly` when
omitted).

```
GET …/reports/suppliers/expenditure-reports/expenses-summary?period_from=2026-01-01&period_to=2026-03-31
GET …/reports/suppliers/expenditure-reports/expenses-summary?period_from=2026-01-01&period_to=2026-12-31&report_cycle=quarterly&expense_category_id=4
```

## Header additions

- `property` — `{ id, name }` when a `facility_id` is given, otherwise `null`.
- `landlord` — `{ id, name }` when a `landlord_id` is given, otherwise `null`.
- `currency` — `{ code, name }` the report's money is in, or `null` for mixed currencies (see above).

## Buckets

| `bucket` | `header.label` | One row per… |
|---|---|---|
| `expense-categories` | `By Expense Category` | expense category, then a `Total` subtotal row |
| `expense-types` | `By Expense Type` | expense type, then a `Total` subtotal row |

Always both, in that order. Rows are ordered by **total descending** (then by name). Each bucket's
`header` also carries `currency` (same as `header.currency`).

## Columns — generated from the cycle

| Column | `key` | `format` |
|---|---|---|
| Expense Category / Expense Type | `name` | string |
| *(per period)* e.g. `Jan 2026` | `period_{i}` | money |
| Total | `total` | money, `grosstotal` |

- **Period columns are `togglable: true`** and carry `period_from` / `period_to`, so a column's window
  needs no parsing.
- A period with no spend is `0`, not omitted.
- Calendar periods are clamped to the requested window, so an edge period only counts expenses inside it.
- Read the columns from `fields`; never assume how many there are.

| Period requested | `report_cycle` | Period columns |
|---|---|---|
| 1 Jan – 31 Mar 2026 | `monthly` (default) | 3 — `Jan 2026`, `Feb 2026`, `Mar 2026` |
| 1 Jan – 31 Mar 2026 | `quarterly` | 1 — `Q1 2026` |
| 1 Jan – 30 Jun 2026 | `quarterly` | 2 — `Q1 2026`, `Q2 2026` |
| 1 Jan – 31 Dec 2026 | `annually` | 1 — `2026` |
| any | `weekly` | one per ISO week, labelled e.g. `W03 2026` |

## Presentation

- The closing `Total` row → `type: subtotal`, `background_color: secondary`.
- The `total` column → `type: grosstotal`, `font-bold`.
- No cell colours.

## Summary

| Key | Where | Meaning |
|---|---|---|
| `total_expenditure` | report and both buckets | Total expenditure (tax inclusive); identical in both buckets |
| `expense_count` | report and both buckets | Expenses counted |
| `group_count` | buckets only | Rows before the `Total` row |

## Example response

Three months read monthly; Maintenance carries expenses of both categories, which is why the two
buckets have different rows but the same `Total`.

```jsonc
{
  "data": {
    "header": {
      "property": null,
      "landlord": { "id": 7, "name": "Jane Wanjiku" },
      "currency": { "code": "KES", "name": "Kenya Shilling" },
      "period": { "from": "2026-01-01", "to": "2026-03-31" },
      "filters": { "landlord_id": 7, "facility_id": null, "facility_type_id": null,
                   "properties_status": null, "period_from": "2026-01-01", "period_to": "2026-03-31",
                   "report_cycle": "monthly", "expense_category_id": null, "currency_id": null },
      "generated_at": "2026-09-18T09:12:44+03:00"
    },
    "report": [
      {
        "bucket": "expense-categories",
        "header": { "label": "By Expense Category", "currency": { "code": "KES", "name": "Kenya Shilling" } },
        "fields": [
          { "label": "Expense Category", "key": "name", "format": "string", "type": "normal", "weight": "font-normal", "background_color": "none", "alignment": "left", "visible": true, "togglable": false },
          { "label": "Jan 2026", "key": "period_0", "format": "money", "type": "normal", "weight": "font-normal", "background_color": "none", "alignment": "right", "visible": true, "togglable": true, "period_from": "2026-01-01", "period_to": "2026-01-31" },
          { "label": "Feb 2026", "key": "period_1", "format": "money", "type": "normal", "weight": "font-normal", "background_color": "none", "alignment": "right", "visible": true, "togglable": true, "period_from": "2026-02-01", "period_to": "2026-02-28" },
          { "label": "Mar 2026", "key": "period_2", "format": "money", "type": "normal", "weight": "font-normal", "background_color": "none", "alignment": "right", "visible": true, "togglable": true, "period_from": "2026-03-01", "period_to": "2026-03-31" },
          { "label": "Total", "key": "total", "format": "money", "type": "grosstotal", "weight": "font-bold", "background_color": "none", "alignment": "right", "visible": true, "togglable": false }
        ],
        "items": [
          { "name": { "value": "Utilities" }, "period_0": { "value": 1000 }, "period_1": { "value": 250 }, "period_2": { "value": 0 }, "total": { "value": 1250 }, "type": "normal" },
          { "name": { "value": "Repairs" }, "period_0": { "value": 0 }, "period_1": { "value": 400 }, "period_2": { "value": 600 }, "total": { "value": 1000 }, "type": "normal" },
          { "name": { "value": "Total" }, "period_0": { "value": 1000 }, "period_1": { "value": 650 }, "period_2": { "value": 600 }, "total": { "value": 2250 }, "type": "subtotal", "background_color": "secondary" }
        ],
        "summary": { "total_expenditure": 2250, "expense_count": 4, "group_count": 2 }
      },
      {
        "bucket": "expense-types",
        "header": { "label": "By Expense Type", "currency": { "code": "KES", "name": "Kenya Shilling" } },
        "fields": [
          { "label": "Expense Type", "key": "name", "format": "string", /* …same period and total columns… */ }
        ],
        "items": [
          { "name": { "value": "Power" }, "period_0": { "value": 1000 }, "period_1": { "value": 0 }, "period_2": { "value": 0 }, "total": { "value": 1000 }, "type": "normal" },
          { "name": { "value": "Maintenance" }, "period_0": { "value": 0 }, "period_1": { "value": 250 }, "period_2": { "value": 600 }, "total": { "value": 850 }, "type": "normal" },
          { "name": { "value": "Plumbing" }, "period_0": { "value": 0 }, "period_1": { "value": 400 }, "period_2": { "value": 0 }, "total": { "value": 400 }, "type": "normal" },
          { "name": { "value": "Total" }, "period_0": { "value": 1000 }, "period_1": { "value": 650 }, "period_2": { "value": 600 }, "total": { "value": 2250 }, "type": "subtotal", "background_color": "secondary" }
        ],
        "summary": { "total_expenditure": 2250, "expense_count": 4, "group_count": 3 }
      }
    ],
    "summary": { "total_expenditure": 2250, "expense_count": 4 }
  }
}
```

Sending `report_cycle=quarterly` over the same window returns one `Q1 2026` column instead of three;
the totals do not change.
