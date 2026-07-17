# Income & Expenditure Report API

Domain: `Property Management > Reports > Landlords`

Base route:

`/api/v1/app/{company}/property-management/reports`

The Income & Expenditure report is a **cash-basis** statement of what each property received and spent
over a period, reconciled down to the amount remittable to the landlord. It is **read-only** and
computed on the fly.

> **Read [the shared reports contract](./README.md) first.** This page only documents what is specific
> to this report — its extra filters, its matrix layout, its property-scoped buckets, and its
> reconciliation. The response envelope (`header` + a `report` array of buckets + `summary`), the
> bucket keys (`bucket`/`header`/`fields`/`items`/`summary`), the field keys, the per-cell
> `{ value, color }` shape, and the colour/format vocabulary all live in the README and apply here
> unchanged.

## Endpoints

- `GET /reports/landlords/income-and-expenditure`

## What makes this report different

Unlike a per-lease report, this one is a **matrix**:

- **Rows** are category lines grouped into three sections — **Income Received** (one row per lease
  component), **Expenses** (one row per expense type), and **Summary** (the reconciliation). Each
  section ends in a `Sub Total`.
- **Columns** are **time periods** (`report_type`) plus a `Total`. The period columns are
  [dynamic columns](./README.md#fields--column-definitions): keys `period_0`, `period_1`,
  … each carrying `period_from` / `period_to`.
- **Buckets are property-scoped.** With a `facility_id` you get one `property_{id}` bucket; without
  one you get an `overall` roll-up **plus** a `property_{id}` bucket per property in scope.

## Get the report

`GET /api/v1/app/{company}/property-management/reports/landlords/income-and-expenditure`

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
| `report_type` | enum `weekly` \| `monthly` \| `quarterly` \| `annually` | **`monthly`** — how period columns are grouped |
| `account_id` | `expense_categories.id` — the **Account** | All accounts |
| `currency_id` | `currencies.id` | **Required when `facility_id` is omitted** (see currency below) |

- **`account_id`** scopes both sides of the statement to one account: **income** to lease components
  mapped to that expense category (`lease_components.expense_category_id`), and **expenses** to expense
  types under it (`facility_expense_types.expense_category_id`).
- **`currency_id`** is **required only when no `facility_id` is given**. Requesting a multi-property
  roll-up without it returns `422` with a `currency_id` error.

Example:

```
GET …/reports/landlords/income-and-expenditure?facility_id=1&period_from=2026-01-01&period_to=2026-12-31&report_type=monthly
GET …/reports/landlords/income-and-expenditure?landlord_id=7&period_from=2026-01-01&period_to=2026-12-31&currency_id=1
```

### Currency

- **Single property** (`facility_id` set): everything is in that property's reporting currency
  (`facilities.reporting_currency_id`); `currency_id` is ignored and the currency sits on
  `header.property.currency`.
- **Multiple properties** (no `facility_id`): every bucket — `overall` and each `property_{id}` — is
  converted to the requested `currency_id`, so the property buckets sum to `overall`. The currency is
  surfaced at `header.currency`.

## Header additions

Beyond the shared header fields, this report adds:

- `landlord` — `{ id, name }` when a `landlord_id` filter is supplied, otherwise `null`.

## Buckets

**This report's buckets are data-driven** — how many you get, and their ids, depend on the properties
in scope. Read them from the `report` array; never assume a fixed set.

| `bucket` | When present | `header.label` | One row per… | Columns |
|---|---|---|---|---|
| `overall` | only when `facility_id` is omitted | `Overall` | category line / summary line, summed across properties | `label`, one `period_{i}` per period, `total` |
| `property_{id}` | always (one per property in scope) | the property's name | category line / summary line | same |

Order: `overall` first when present, then one bucket per property **sorted by property name**.

Bucket `header` additions beyond the standard `label`:

- `property` — `{ id, name }` on each `property_{id}` bucket. **Absent on `overall`** (a roll-up has
  no single property). This is how you identify which property a bucket is for — don't parse the
  `bucket` id.

Each bucket's `summary` holds that bucket's own KPI scalars (same keys as the report-level `summary`
below). For a single-property report they match `data.summary`; for a roll-up, `data.summary` mirrors
the `overall` bucket.

Every bucket shares the same column set. Each row carries the standard row metadata (`type`, and
`background_color` when not `none`) **plus a report-specific `section` key** (`income` | `expenses` |
`summary`) so a renderer can group the three sections. Section banner rows (`Income Received`,
`Expenses`, `Summary`) are a single `label` cell with `col_span` covering every column — the data
cells are omitted (see the [shared contract](./README.md#items--rows) for `col_span`).

### Reconciliation (the Summary section)

Computed on the fly per period column:

```
Surplus                     = Income Sub Total − Expenses Sub Total
Less: Tax Withheld By Tenants = income received via a withholding payment method
                                (payment_methods.is_withholding = true)
Less: Non-Remittable Income = income on components outside the contract's remittable_components
Add:  Non-Remittable Expenses = expenses whose category is NOT in the contract's deductible_expenses
                                (deductible_expenses are netted off before remitting, so everything
                                else is added back — matching ComputeRemittableAmountService)
Remittable Amount           = Surplus − Tax Withheld − Non-Remittable Income + Non-Remittable Expenses
Less Advance Remittance     = advance FacilityRemittance amounts falling in the column
Nett Remittance             = Remittable Amount − Advance Remittance
```

Notes:

- Income is **confirmed** `LeaseCollection` only; expenses exclude `cancelled` / `pending`.
- The management fee is **not** a separate line — it is already an ordinary expense row, so it sits
  inside the Expenses Sub Total.
- When a property has no management contract (or empty lists), Non-Remittable Income / Expenses are 0.

### This report's presentation choices

- Section banner rows → `type: subtotal`, `background_color: primary`.
- `Sub Total` and `Remittable Amount` rows → `type: subtotal`, `background_color: secondary`.
- `Nett Remittance` row → `type: grosstotal`, `background_color: secondary`.
- The `total` column → `type: grosstotal`, `font-bold`.

## Sample response

Trimmed to the `header`, one property bucket (monthly, two months), and the `summary`.

```json
{
  "data": {
    "header": {
      "property": {
        "id": 1,
        "name": "ACK Gardens",
        "currency": { "code": "KES", "name": "Kenya Shilling" }
      },
      "landlord": { "id": 7, "name": "ACK Trustees" },
      "period": { "from": "2026-01-01", "to": "2026-02-28" },
      "filters": {
        "landlord_id": 7,
        "facility_id": 1,
        "facility_type_id": null,
        "properties_status": null,
        "account_id": null,
        "report_type": "monthly",
        "currency_id": null
      },
      "generated_at": "2026-07-16T14:52:57+03:00"
    },
    "report": [
      {
        "bucket": "property_1",
        "header": {
          "label": "ACK Gardens",
          "property": { "id": 1, "name": "ACK Gardens" }
        },
        "fields": [
          { "label": "", "key": "label", "format": "string", "type": "normal", "weight": "font-normal", "background_color": "none", "alignment": "left", "visible": true, "togglable": false },
          { "label": "Jan 2026", "key": "period_0", "format": "money", "type": "normal", "weight": "font-normal", "background_color": "none", "alignment": "right", "visible": true, "togglable": false, "period_from": "2026-01-01", "period_to": "2026-01-31" },
          { "label": "Feb 2026", "key": "period_1", "format": "money", "type": "normal", "weight": "font-normal", "background_color": "none", "alignment": "right", "visible": true, "togglable": false, "period_from": "2026-02-01", "period_to": "2026-02-28" },
          { "label": "Total", "key": "total", "format": "money", "type": "grosstotal", "weight": "font-bold", "background_color": "none", "alignment": "right", "visible": true, "togglable": false }
        ],
        "items": [
          { "label": { "value": "Income Received", "col_span": 4 }, "section": "income", "type": "subtotal", "background_color": "primary" },
          { "label": { "value": "Rent" }, "period_0": { "value": 9125836.72 }, "period_1": { "value": 5591722.92 }, "total": { "value": 14717559.64 }, "section": "income", "type": "normal" },
          { "label": { "value": "Sub Total" }, "period_0": { "value": 12438645.80 }, "period_1": { "value": 8689246.46 }, "total": { "value": 21127892.26 }, "section": "income", "type": "subtotal", "background_color": "secondary" },

          { "label": { "value": "Expenses", "col_span": 4 }, "section": "expenses", "type": "subtotal", "background_color": "primary" },
          { "label": { "value": "Electricity" }, "period_0": { "value": 1048741.20 }, "period_1": { "value": 1177796.59 }, "total": { "value": 2226537.79 }, "section": "expenses", "type": "normal" },
          { "label": { "value": "Sub Total" }, "period_0": { "value": 3423437.25 }, "period_1": { "value": 3588420.63 }, "total": { "value": 7011857.88 }, "section": "expenses", "type": "subtotal", "background_color": "secondary" },

          { "label": { "value": "Summary", "col_span": 4 }, "section": "summary", "type": "subtotal", "background_color": "primary" },
          { "label": { "value": "Surplus" }, "period_0": { "value": 9015208.55 }, "period_1": { "value": 5100825.83 }, "total": { "value": 14116034.38 }, "section": "summary", "type": "subtotal" },
          { "label": { "value": "Less: Tax Withheld By Tenants" }, "period_0": { "value": 355973.00 }, "period_1": { "value": 210794.00 }, "total": { "value": 566767.00 }, "section": "summary", "type": "normal" },
          { "label": { "value": "Less: Non-Remittable Income" }, "period_0": { "value": 159307.00 }, "period_1": { "value": 239839.64 }, "total": { "value": 399146.64 }, "section": "summary", "type": "normal" },
          { "label": { "value": "Add: Non-Remittable Expenses" }, "period_0": { "value": 0 }, "period_1": { "value": 0 }, "total": { "value": 0 }, "section": "summary", "type": "normal" },
          { "label": { "value": "Remittable Amount" }, "period_0": { "value": 8499928.55 }, "period_1": { "value": 4650192.19 }, "total": { "value": 13150120.74 }, "section": "summary", "type": "subtotal", "background_color": "secondary" },
          { "label": { "value": "Less Advance Remittance" }, "period_0": { "value": 0 }, "period_1": { "value": 0 }, "total": { "value": 0 }, "section": "summary", "type": "normal" },
          { "label": { "value": "Nett Remittance" }, "period_0": { "value": 8499928.55 }, "period_1": { "value": 4650192.19 }, "total": { "value": 13150120.74 }, "section": "summary", "type": "grosstotal", "background_color": "secondary" }
        ],
        "summary": {
          "total_income": 21127892.26,
          "total_expenses": 7011857.88,
          "total_surplus": 14116034.38,
          "total_remittable": 13150120.74,
          "nett_remittance": 13150120.74
        }
      }
    ],
    "summary": {
      "total_income": 21127892.26,
      "total_expenses": 7011857.88,
      "total_surplus": 14116034.38,
      "total_remittable": 13150120.74,
      "nett_remittance": 13150120.74
    }
  }
}
```

For how to render any of this generically (columns from `fields`, cell `value`/`color`, row and
column tints, subtotal/grosstotal emphasis), see the
[render guide in the contract](./README.md#how-to-render-report-template).
