# Property Expenses Report API

Domain: `Property Management > Reports > Landlords`

Base route:

`/api/v1/app/{company}/property-management/reports`

The Property Expenses report is a per-property **expense register** with two cross-property summaries —
by expense **category**, and by expense **type** broken down into **sub-types**. It is **read-only**
and computed on the fly.

> **Read [the shared reports contract](./README.md) first.** This page only documents what is specific
> to this report — its extra filters, its buckets, its per-bucket currency, and its presentation. The
> response envelope (`header` + a `report` array of buckets + `summary`), the bucket keys
> (`bucket`/`header`/`fields`/`items`/`summary`), the field keys, the per-cell `{ value, color }`
> shape, and the colour/format vocabulary all live in the README and apply here unchanged.

## Endpoints

- `GET /reports/landlords/property-expenses`

## What makes this report different

- **Fixed summary buckets + data-driven property buckets.** Two summaries always come first
  (`summary_by_category`, `summary_by_type`), then one `property_{id}` bucket per property in scope
  (only the selected one when a single `facility_id` is given). There is **no `overall`** transaction
  bucket — the summaries carry the roll-up.
- **The type summary is two-level.** In `summary_by_type` each expense type is a full-width **banner
  row**, with one row per **sub-type** beneath it, closed by a per-type **subtotal** row. Expenses
  with no sub-type roll up under an **"Other {type name}"** row (e.g. "Other Electricity").
- **Per-bucket currency.** Property buckets are shown in each property's own reporting currency, while
  the summaries are in one target currency — so different buckets in the same response may use
  **different currencies**. Each bucket declares its own `header.currency`.

## Get the report

`GET /api/v1/app/{company}/property-management/reports/landlords/property-expenses`

### Filters (query params)

All filters are **optional** and flat (not nested under `filter[...]`). The six
[global filters](./README.md#global-filters) apply; this report adds:

| Param | Type | Meaning when omitted |
|---|---|---|
| `landlord_id` | `users.id` | All landlords (global) |
| `facility_id` | `facilities.id` | All properties (global) — drives single vs. all property buckets |
| `facility_type_id` | `facility_types.id` | All property types (global) |
| `properties_status` | enum `App\Enums\Status` | Any property status (global) |
| `period_from` | date `Y-m-d` | Start of the current month (global) |
| `period_to` | date `Y-m-d` | End of the current month (global) |
| `expense_category_id` | `expense_categories.id` | All categories |
| `expense_type_id` | `facility_expense_types.id` | All types |
| `currency_id` | `currencies.id` | **Required when `facility_id` is omitted** (see currency below) |

- **`expense_category_id` / `expense_type_id`** narrow every bucket to that category / type
  (`facility_expenses.expense_category_id` / `facility_expenses.expense_type_id`).
- **`currency_id`** is **required only when no `facility_id` is given**. Requesting a multi-property
  roll-up without it returns `422` with a `currency_id` error.
- **Cancelled and pending expenses are excluded** — they are not yet realised costs.

Example:

```
GET …/reports/landlords/property-expenses?facility_id=1&period_from=2026-01-01&period_to=2026-01-31
GET …/reports/landlords/property-expenses?landlord_id=7&period_from=2026-01-01&period_to=2026-01-31&currency_id=1
```

### Currency

Each expense carries its own `currency_id`. Every amount is first converted to **its facility's
reporting currency** (`facilities.reporting_currency_id`) — that is what the `property_{id}` buckets
display. The summaries then convert those figures to a **target currency**:

- **Single property** (`facility_id` set): the target is that property's reporting currency; `currency_id`
  is ignored. `header.property.currency` holds it.
- **Multiple properties** (no `facility_id`): the target is the requested `currency_id`, surfaced at
  `header.currency`. Each `property_{id}` bucket still reports in **its own** reporting currency.

Because buckets can be in different currencies, **every bucket's `header.currency` (`{ code, name }`)
states the currency its own money is in** — format each bucket's `money` cells with that, not the
top-level `header.currency`.

## Header additions

Beyond the shared header fields, this report adds:

- `landlord` — `{ id, name }` when a `landlord_id` filter is supplied, otherwise `null`.

## Buckets

| `bucket` | When present | `header.label` | One row per… | Columns |
|---|---|---|---|---|
| `summary_by_category` | always | `Summary by Category` | expense category (+ a totals row) | `label`, `amount`, `tax`, `total` |
| `summary_by_type` | always | `Summary by Type` | a banner per type, then a row per sub-type, then a per-type subtotal row (+ a final totals row) | `label`, `amount`, `tax`, `total` |
| `property_{id}` | one per property in scope | the property's name | expense (+ a totals row) | `invoice_number`, `date`, `notes`, `vendor`, `category`, `expense_type`, `amount`, `tax`, `total`, `invoice` |

Order: the two summaries first, then one `property_{id}` bucket per property **sorted by property
name**. Rows in both summaries are ordered **descending by total**.

Bucket `header` additions beyond the standard `label`:

- `currency` — `{ code, name }` on **every** bucket: the currency that bucket's money is expressed in.
- `property` — `{ id, name }` on each `property_{id}` bucket (identifies the property — don't parse the
  `bucket` id).

Each bucket's `summary` holds that bucket's own KPI scalars (same keys as the report-level `summary`).

### Column notes

- **`summary_by_type` banner rows** are a single `label` cell with `col_span: 4` (covering all four
  columns); the amount cells are omitted. They carry `type: subtotal`, `background_color: primary`.
- **`property_{id}` totals row** is a `Total` label with `col_span: 6` (covering the six leading text
  columns), then the `amount` / `tax` / `total` sums; `invoice` is omitted.
- **`invoice`** cell value is `{ file_name, source_url }`, or `null` when no invoice is attached.
- The expense-type column key is **`expense_type`** (not `type`) — `type` is reserved for row metadata.

### This report's presentation choices

- Type banner rows → `type: subtotal`, `background_color: primary`.
- Per-type subtotal rows (in `summary_by_type`) → `type: subtotal`, `background_color: secondary`.
- Every bucket's final totals row → `type: grosstotal`, `background_color: secondary`.
- The `total` column → `type: grosstotal`, `font-bold`.

## `summary`

Report-level and per-bucket, with the same shape: `total_amount`, `total_tax`, `total`,
`expense_count`. The report-level and the two summary buckets are in the target currency; each
`property_{id}` bucket's summary is in that facility's reporting currency.

## Sample response

Trimmed to the `header`, the two summary buckets, one property bucket, and the `summary`
(single-property request, so all buckets share one currency).

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
      "period": { "from": "2026-01-01", "to": "2026-01-31" },
      "filters": {
        "landlord_id": null,
        "facility_id": 1,
        "facility_type_id": null,
        "properties_status": null,
        "expense_category_id": null,
        "expense_type_id": null,
        "currency_id": null
      },
      "generated_at": "2026-07-18T14:52:57+03:00"
    },
    "report": [
      {
        "bucket": "summary_by_category",
        "header": { "label": "Summary by Category", "currency": { "code": "KES", "name": "Kenya Shilling" } },
        "fields": [
          { "label": "Category", "key": "label", "format": "string", "type": "normal", "weight": "font-normal", "background_color": "none", "alignment": "left", "visible": true, "togglable": false },
          { "label": "Amount", "key": "amount", "format": "money", "type": "normal", "weight": "font-normal", "background_color": "none", "alignment": "right", "visible": true, "togglable": false },
          { "label": "Tax", "key": "tax", "format": "money", "type": "normal", "weight": "font-normal", "background_color": "none", "alignment": "right", "visible": true, "togglable": false },
          { "label": "Total", "key": "total", "format": "money", "type": "grosstotal", "weight": "font-bold", "background_color": "none", "alignment": "right", "visible": true, "togglable": false }
        ],
        "items": [
          { "label": { "value": "Utilities" }, "amount": { "value": 1300 }, "tax": { "value": 180 }, "total": { "value": 1480 }, "type": "normal" },
          { "label": { "value": "Repairs" }, "amount": { "value": 900 }, "tax": { "value": 135 }, "total": { "value": 1035 }, "type": "normal" },
          { "label": { "value": "Total" }, "amount": { "value": 2200 }, "tax": { "value": 315 }, "total": { "value": 2515 }, "type": "grosstotal", "background_color": "secondary" }
        ],
        "summary": { "total_amount": 2200, "total_tax": 315, "total": 2515, "expense_count": 4 }
      },
      {
        "bucket": "summary_by_type",
        "header": { "label": "Summary by Type", "currency": { "code": "KES", "name": "Kenya Shilling" } },
        "fields": [
          { "label": "Type / Sub-type", "key": "label", "format": "string", "type": "normal", "weight": "font-normal", "background_color": "none", "alignment": "left", "visible": true, "togglable": false },
          { "label": "Amount", "key": "amount", "format": "money", "type": "normal", "weight": "font-normal", "background_color": "none", "alignment": "right", "visible": true, "togglable": false },
          { "label": "Tax", "key": "tax", "format": "money", "type": "normal", "weight": "font-normal", "background_color": "none", "alignment": "right", "visible": true, "togglable": false },
          { "label": "Total", "key": "total", "format": "money", "type": "grosstotal", "weight": "font-bold", "background_color": "none", "alignment": "right", "visible": true, "togglable": false }
        ],
        "items": [
          { "label": { "value": "Electricity", "col_span": 4 }, "type": "subtotal", "background_color": "primary" },
          { "label": { "value": "Metered" }, "amount": { "value": 800 }, "tax": { "value": 120 }, "total": { "value": 920 }, "type": "normal" },
          { "label": { "value": "Standing" }, "amount": { "value": 400 }, "tax": { "value": 60 }, "total": { "value": 460 }, "type": "normal" },
          { "label": { "value": "Other Electricity" }, "amount": { "value": 100 }, "tax": { "value": 0 }, "total": { "value": 100 }, "type": "normal" },
          { "label": { "value": "Electricity Subtotal" }, "amount": { "value": 1300 }, "tax": { "value": 180 }, "total": { "value": 1480 }, "type": "subtotal", "background_color": "secondary" },
          { "label": { "value": "Plumbing", "col_span": 4 }, "type": "subtotal", "background_color": "primary" },
          { "label": { "value": "Pipes" }, "amount": { "value": 900 }, "tax": { "value": 135 }, "total": { "value": 1035 }, "type": "normal" },
          { "label": { "value": "Plumbing Subtotal" }, "amount": { "value": 900 }, "tax": { "value": 135 }, "total": { "value": 1035 }, "type": "subtotal", "background_color": "secondary" },
          { "label": { "value": "Total" }, "amount": { "value": 2200 }, "tax": { "value": 315 }, "total": { "value": 2515 }, "type": "grosstotal", "background_color": "secondary" }
        ],
        "summary": { "total_amount": 2200, "total_tax": 315, "total": 2515, "expense_count": 4 }
      },
      {
        "bucket": "property_1",
        "header": {
          "label": "ACK Gardens",
          "property": { "id": 1, "name": "ACK Gardens" },
          "currency": { "code": "KES", "name": "Kenya Shilling" }
        },
        "fields": [
          { "label": "Invoice #", "key": "invoice_number", "format": "string", "type": "normal", "weight": "font-normal", "background_color": "none", "alignment": "left", "visible": true, "togglable": false },
          { "label": "Date", "key": "date", "format": "string", "type": "normal", "weight": "font-normal", "background_color": "none", "alignment": "left", "visible": true, "togglable": false },
          { "label": "Notes", "key": "notes", "format": "string", "type": "normal", "weight": "font-normal", "background_color": "none", "alignment": "left", "visible": true, "togglable": false },
          { "label": "Vendor", "key": "vendor", "format": "string", "type": "normal", "weight": "font-normal", "background_color": "none", "alignment": "left", "visible": true, "togglable": false },
          { "label": "Category", "key": "category", "format": "string", "type": "normal", "weight": "font-normal", "background_color": "none", "alignment": "left", "visible": true, "togglable": false },
          { "label": "Type", "key": "expense_type", "format": "string", "type": "normal", "weight": "font-normal", "background_color": "none", "alignment": "left", "visible": true, "togglable": false },
          { "label": "Amount", "key": "amount", "format": "money", "type": "normal", "weight": "font-normal", "background_color": "none", "alignment": "right", "visible": true, "togglable": false },
          { "label": "Tax", "key": "tax", "format": "money", "type": "normal", "weight": "font-normal", "background_color": "none", "alignment": "right", "visible": true, "togglable": false },
          { "label": "Total", "key": "total", "format": "money", "type": "grosstotal", "weight": "font-bold", "background_color": "none", "alignment": "right", "visible": true, "togglable": false },
          { "label": "Invoice", "key": "invoice", "format": "string", "type": "normal", "weight": "font-normal", "background_color": "none", "alignment": "left", "visible": true, "togglable": false }
        ],
        "items": [
          { "invoice_number": { "value": "INV-014" }, "date": { "value": "16 Jan, 2026" }, "notes": { "value": "Meter top-up" }, "vendor": { "value": "Kenya Power" }, "category": { "value": "Utilities" }, "expense_type": { "value": "Electricity" }, "amount": { "value": 800 }, "tax": { "value": 120 }, "total": { "value": 920 }, "invoice": { "value": { "file_name": "kplc-jan.pdf", "source_url": "https://…/kplc-jan.pdf" } }, "type": "normal" },
          { "invoice_number": { "value": "INV-021" }, "date": { "value": "16 Jan, 2026" }, "notes": { "value": "Pipe repair" }, "vendor": { "value": "AquaFix" }, "category": { "value": "Repairs" }, "expense_type": { "value": "Plumbing" }, "amount": { "value": 900 }, "tax": { "value": 135 }, "total": { "value": 1035 }, "invoice": { "value": null }, "type": "normal" },
          { "invoice_number": { "value": "Total", "col_span": 6 }, "amount": { "value": 2200 }, "tax": { "value": 315 }, "total": { "value": 2515 }, "type": "grosstotal", "background_color": "secondary" }
        ],
        "summary": { "total_amount": 2200, "total_tax": 315, "total": 2515, "expense_count": 4 }
      }
    ],
    "summary": { "total_amount": 2200, "total_tax": 315, "total": 2515, "expense_count": 4 }
  }
}
```

For how to render any of this generically (columns from `fields`, cell `value`/`color`, row and
column tints, subtotal/grosstotal emphasis), see the
[render guide in the contract](./README.md#how-to-render-report-template).
