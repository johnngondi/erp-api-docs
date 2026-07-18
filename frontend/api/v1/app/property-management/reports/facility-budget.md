# Facility Budget Report API

Domain: `Property Management > Reports > Landlords`

Base route:

`/api/v1/app/{company}/property-management/reports`

The Facility Budget report is a **budget-vs-actual** statement for each property over a period. For
every time **cycle** it shows what was **budgeted**, what **actually** happened, the **surplus/deficit**,
and a **status** flag. It is **read-only** and computed on the fly.

> **Read [the shared reports contract](./README.md) first.** This page only documents what is specific
> to this report — its extra filters, its per-cycle four-column layout, its property-scoped buckets,
> and its per-bucket currency. The response envelope (`header` + a `report` array of buckets +
> `summary`), the bucket keys, the field keys, the per-cell `{ value, color }` shape, and the
> colour/format vocabulary all live in the README and apply here unchanged.

## Endpoints

- `GET /reports/landlords/facility-budget`

## What makes this report different

Like Income & Expenditure this is a **matrix**, but each period column is a **group of four**:

- **Rows** are budget lines grouped into two sections — **Income Budget** (one row per lease
  component) and **Expenditure Budget** (one row per expense type). Each section ends in a `Sub Total`.
  Rows are the union of budgeted lines and lines that have actuals, so a budgeted line with no spend
  and real activity with no budget both appear (the missing side is `0`).
- **Columns** are **time cycles** (`report_cycle`), each expanding into four columns —
  **Budget**, **Actual**, **S/D** (surplus/deficit), **Status** — followed by a **Total** group of the
  same four. The per-cycle columns are [dynamic columns](./README.md#fields--column-definitions):
  keys `period_0_budget`, `period_0_actual`, `period_0_variance`, `period_0_status`, `period_1_budget`,
  … each money column carrying `period_from` / `period_to`.
- **Buckets are property-scoped.** With a `facility_id` you get one `property_{id}` bucket; without one
  you get an `overall` roll-up **plus** a `property_{id}` bucket per property in scope.

### Where the numbers come from

- **Budget** — the stored `FacilityBudget` line items' `budgeted_amount`, prorated across the cycles
  each budget's `[period_start, period_end]` overlaps (by day-overlap).
- **Actual** — re-derived live per cycle from real transactions: confirmed collections
  (`LeaseCollection`) for income by lease component, non-cancelled expenses (`FacilityExpense`) for
  expenses by expense type. This is the same sourcing that computes a budget's stored actuals.
- **S/D (surplus/deficit)** — side-aware: Income `= actual − budget` (positive = surplus); Expense
  `= budget − actual` (positive = under budget). **Positive is favourable on both sides.**
- **Status** — `ok` | `over` | `under`, comparing the actual to the time-prorated budget expectation
  (`budget × elapsed fraction of the cycle`) against `budget_at_risk_threshold_percent`. The cell
  `color` encodes favourability: `success` when favourable (income over / expense under), `danger`
  when unfavourable, `none` when on target.

## Get the report

`GET /api/v1/app/{company}/property-management/reports/landlords/facility-budget`

### Filters (query params)

All filters are **optional** and flat (not nested under `filter[...]`). The six
[global filters](./README.md#global-filters) apply; this report adds:

| Param | Type | Meaning when omitted |
|---|---|---|
| `landlord_id` | `users.id` | All landlords (global) |
| `facility_id` | `facilities.id` | All properties (global) — drives single vs. overall buckets |
| `facility_type_id` | `facility_types.id` | All property types (global) |
| `properties_status` | enum `App\Enums\Status` | Any property status (global) |
| `period_from` | date `Y-m-d` | Start of the current month (global) |
| `period_to` | date `Y-m-d` | End of the current month (global) |
| `report_cycle` | enum `weekly` \| `monthly` \| `quarterly` \| `annually` | **`annually`** — how cycle columns are grouped |
| `currency_id` | `currencies.id` | **Required when `facility_id` is omitted** (see currency below) |

Example:

```
GET …/reports/landlords/facility-budget?facility_id=1&period_from=2026-01-01&period_to=2026-12-31&report_cycle=quarterly
GET …/reports/landlords/facility-budget?currency_id=1&period_from=2026-01-01&period_to=2026-12-31
```

### Currency (per bucket)

**Currency lives on each bucket here, not just the report header** — unlike Income & Expenditure,
per-property buckets are **not** converted.

- **Single property** (`facility_id` set): the one bucket is in that property's reporting currency
  (`facilities.reporting_currency_id`); `currency_id` is ignored. The currency sits on
  `header.property.currency` and on the bucket's `header.property.currency`.
- **Multiple properties** (no `facility_id`): the **`overall`** bucket is converted to the requested
  `currency_id` (surfaced at `header.currency` and `overall.header.currency`); **each `property_{id}`
  bucket stays in its own budget currency**, surfaced at that bucket's `header.property.currency`. So
  the per-property figures sum to `overall` **only after** converting each to the selected currency.
  Requesting a multi-property roll-up without `currency_id` returns `422` with a `currency_id` error.

> Because buckets can have different currencies, format each bucket's money cells with **that bucket's**
> currency (`header.property.currency`, or `header.currency` on `overall`), not a single report-level
> currency.

## Header additions

Beyond the shared header fields, this report adds:

- `landlord` — `{ id, name }` when a `landlord_id` filter is supplied, otherwise `null`.

## Buckets

**Data-driven** — how many, and their ids, depend on the properties in scope. Read them from the
`report` array; never assume a fixed set.

| `bucket` | When present | `header.label` | One row per… | Columns |
|---|---|---|---|---|
| `overall` | only when `facility_id` is omitted | `Overall` | budget line, summed across properties (converted) | `label`, four per cycle, four `total_*` |
| `property_{id}` | always (one per property in scope) | the property's name | budget line (native currency) | same |

Order: `overall` first when present, then one bucket per property **sorted by property name**.

Bucket `header` additions beyond the standard `label`:

- `property` — `{ id, name, currency: { code, name } }` on each `property_{id}` bucket. **Absent on
  `overall`**, which instead carries `currency` — `{ code, name }` of the selected roll-up currency.

Each bucket's `summary` holds that bucket's own KPI scalars (same keys as the report-level `summary`).

Every bucket shares the same column set. Each row carries the standard row metadata (`type`, and
`background_color` when not `none`) **plus a report-specific `section` key** (`income` | `expenses`).
Section banner rows (`Income Budget`, `Expenditure Budget`) are a single `label` cell with `col_span`
covering every column — the data cells are omitted.

### This report's presentation choices

- Section banner rows → `type: subtotal`, `background_color: primary`.
- `Sub Total` rows → `type: subtotal`, `background_color: secondary`.
- The `total_*` columns → `type: grosstotal`, `font-bold`.
- Status cells carry a `color` (`success` / `danger`) only when the line is off-target favourably or
  unfavourably; on-target status omits `color` (renders neutral).

### `summary` scalars

`total_income_budget`, `total_income_actual`, `total_expense_budget`, `total_expense_actual`,
`net_budget` (= income − expense budget), `net_actual` (= income − expense actual).

## Sample response

Trimmed to the `header`, one property bucket (annually, one cycle), and the `summary`.

```json
{
  "data": {
    "header": {
      "property": {
        "id": 1,
        "name": "ACK Gardens",
        "currency": { "code": "KES", "name": "Kenya Shilling" }
      },
      "landlord": null,
      "period": { "from": "2026-01-01", "to": "2026-12-31" },
      "filters": {
        "landlord_id": null,
        "facility_id": 1,
        "facility_type_id": null,
        "properties_status": null,
        "report_cycle": "annually",
        "currency_id": null
      },
      "generated_at": "2026-07-18T14:52:57+03:00"
    },
    "report": [
      {
        "bucket": "property_1",
        "header": {
          "label": "ACK Gardens",
          "property": { "id": 1, "name": "ACK Gardens", "currency": { "code": "KES", "name": "Kenya Shilling" } }
        },
        "fields": [
          { "label": "", "key": "label", "format": "string", "type": "normal", "weight": "font-normal", "background_color": "none", "alignment": "left", "visible": true, "togglable": false },
          { "label": "2026 Budget", "key": "period_0_budget", "format": "money", "type": "normal", "weight": "font-normal", "background_color": "none", "alignment": "right", "visible": true, "togglable": false, "period_from": "2026-01-01", "period_to": "2026-12-31" },
          { "label": "2026 Actual", "key": "period_0_actual", "format": "money", "alignment": "right", "togglable": false, "period_from": "2026-01-01", "period_to": "2026-12-31" },
          { "label": "2026 S/D", "key": "period_0_variance", "format": "money", "alignment": "right", "togglable": false, "period_from": "2026-01-01", "period_to": "2026-12-31" },
          { "label": "2026 Status", "key": "period_0_status", "format": "string", "alignment": "center", "togglable": false, "period_from": "2026-01-01", "period_to": "2026-12-31" },
          { "label": "Total Budget", "key": "total_budget", "format": "money", "type": "grosstotal", "weight": "font-bold", "alignment": "right" },
          { "label": "Total Actual", "key": "total_actual", "format": "money", "type": "grosstotal", "weight": "font-bold", "alignment": "right" },
          { "label": "Total S/D", "key": "total_variance", "format": "money", "type": "grosstotal", "weight": "font-bold", "alignment": "right" },
          { "label": "Total Status", "key": "total_status", "format": "string", "type": "grosstotal", "weight": "font-bold", "alignment": "center" }
        ],
        "items": [
          { "label": { "value": "Income Budget", "col_span": 9 }, "section": "income", "type": "subtotal", "background_color": "primary" },
          { "label": { "value": "Rent" }, "period_0_budget": { "value": 1000 }, "period_0_actual": { "value": 1200 }, "period_0_variance": { "value": 200 }, "period_0_status": { "value": "over", "color": "success" }, "total_budget": { "value": 1000 }, "total_actual": { "value": 1200 }, "total_variance": { "value": 200 }, "total_status": { "value": "over", "color": "success" }, "section": "income", "type": "normal" },
          { "label": { "value": "Sub Total" }, "period_0_budget": { "value": 1000 }, "period_0_actual": { "value": 1200 }, "period_0_variance": { "value": 200 }, "period_0_status": { "value": "over", "color": "success" }, "total_budget": { "value": 1000 }, "total_actual": { "value": 1200 }, "total_variance": { "value": 200 }, "total_status": { "value": "over", "color": "success" }, "section": "income", "type": "subtotal", "background_color": "secondary" },

          { "label": { "value": "Expenditure Budget", "col_span": 9 }, "section": "expenses", "type": "subtotal", "background_color": "primary" },
          { "label": { "value": "Electricity" }, "period_0_budget": { "value": 500 }, "period_0_actual": { "value": 400 }, "period_0_variance": { "value": 100 }, "period_0_status": { "value": "under", "color": "success" }, "total_budget": { "value": 500 }, "total_actual": { "value": 400 }, "total_variance": { "value": 100 }, "total_status": { "value": "under", "color": "success" }, "section": "expenses", "type": "normal" },
          { "label": { "value": "Sub Total" }, "period_0_budget": { "value": 500 }, "period_0_actual": { "value": 400 }, "period_0_variance": { "value": 100 }, "period_0_status": { "value": "under", "color": "success" }, "total_budget": { "value": 500 }, "total_actual": { "value": 400 }, "total_variance": { "value": 100 }, "total_status": { "value": "under", "color": "success" }, "section": "expenses", "type": "subtotal", "background_color": "secondary" }
        ],
        "summary": {
          "total_income_budget": 1000,
          "total_income_actual": 1200,
          "total_expense_budget": 500,
          "total_expense_actual": 400,
          "net_budget": 500,
          "net_actual": 800
        }
      }
    ],
    "summary": {
      "total_income_budget": 1000,
      "total_income_actual": 1200,
      "total_expense_budget": 500,
      "total_expense_actual": 400,
      "net_budget": 500,
      "net_actual": 800
    }
  }
}
```

For how to render any of this generically, see the
[render guide in the contract](./README.md#how-to-render-report-template). Note the per-bucket
currency caveat above.
