# Reports — the shared contract

Domain: `Property Management > Reports`

Base route:

`/api/v1/app/{company}/property-management/reports`

**Read this once.** Every report in this module shares one self-describing response envelope, one set
of global filters, and one presentation vocabulary (formats, colors, weights, background tints). A
single generic table renderer — the reusable **report template** — can display this report and every
future one without hardcoding columns or colours. Each per-report page (e.g.
[billings-and-collections.md](./billings-and-collections.md),
[income-and-expenditure.md](./income-and-expenditure.md)) then only documents what is specific to
that report: its extra filters, its buckets, and its numbers.

## What a report is

- **Read-only** and **computed on the fly** — no persistence, no side effects.
- **Self-describing** — the response tells you its own columns (`fields`) and rows (`items`), so the
  UI renders generically.
- **Discoverable** — every report has a stable string key; the frontend calls the endpoint and
  renders the returned buckets. Never assume a fixed column set — read `fields`.
- **Returned whole** — there is no pagination, sort, or include; a report is computed for the
  resolved period and returned in full.

---

## Global filters

Every report supports these six filters (defined on the shared `App\Data\Reports\ReportFilterData`).
Individual reports may add their own on top (e.g. Billings & Collections adds `leases_status`,
`expense_category_id`, `lease_component_id`; Income & Expenditure adds `report_type` for period
grouping, `account_id`, and a `currency_id` required for multi-property roll-ups).

| Param | Type | Meaning when omitted |
|---|---|---|
| `landlord_id` | `users.id` | All landlords |
| `facility_id` | `facilities.id` | All properties |
| `facility_type_id` | `facility_types.id` | All property types |
| `properties_status` | enum `App\Enums\Status` (`active`, `inactive`, …) | Any property status |
| `period_from` | date `Y-m-d` | **Start of the current month** |
| `period_to` | date `Y-m-d` | **End of the current month** |

Rules that hold for **all** reports:

- All filters are **optional** and passed as **flat** query params (not nested under `filter[...]`).
- `null` means "all" (for ids/types) or "any" (for statuses).
- The period defaults to the **current month** when `period_from` / `period_to` are omitted.
- **Currency is never a filter.** It is resolved from the property's reporting currency
  (`facilities.reporting_currency_id`) and surfaced in the `header`.

Example:

```
GET …/reports/tenants/billings-and-collections?facility_id=1&period_from=2026-07-01&period_to=2026-07-31
```

---

## The response envelope

```jsonc
{
  "data": {
    "header":  { /* who/what/when the report is for */ },
    "report":  {
      "<bucket>": {
        "fields": [ /* column definitions, in display order */ ],
        "items":  [ /* rows; keys match each field's `key` */ ]
      }
    },
    "summary": { /* report-level totals for headline KPI tiles */ }
  }
}
```

### `header`

- `property` — `{ id, name, currency: { code, name } }`, or `null` when no single property is
  selected. **Currency lives on the property** (its reporting currency). When `property` is `null`,
  a top-level `header.currency` fallback (the company default currency, or a report-selected currency)
  is included instead.
- `period` — `{ from, to }` (the resolved period, after defaults).
- `filters` — every filter echoed back (nulls included) so the UI can show the active scope.
- `generated_at` — ISO-8601 timestamp.
- A report **may add** header fields — e.g. Income & Expenditure adds `landlord` (`{ id, name }` when a
  `landlord_id` filter is applied, else `null`). Per-report pages document any such additions.

### `report.<bucket>.fields` — column definitions

Render your table columns from this array, **in order**. Each field:

| Key | Type | Notes |
|---|---|---|
| `label` | string | Column header text. |
| `key` | string | Matches the property holding the cell in each `items` row. Stable machine id (derived from the entity id, not its name). |
| `format` | `string` \| `integer` \| `money` | Drives value formatting. See [Formats](#formats). |
| `type` | `normal` \| `subtotal` \| `grosstotal` | Visual role of the column. See [Field & row type](#field--row-type). Default `normal`. |
| `weight` | Tailwind font weight | e.g. `font-normal`, `font-medium`, `font-bold`. Default `font-normal`. |
| `background_color` | color enum | Tints the **whole column**. See [Colors](#colors). Default `none`. |
| `alignment` | `left` \| `center` \| `right` | Convention: text → left, numbers → right. |
| `visible` | boolean | Initial visibility. Hidden columns should not render by default. |
| `togglable` | boolean | If `true`, offer it in a column picker; if `false`, it is fixed. |

**Dynamic columns:** some buckets append runtime columns (one per account or per component) with keys
like `account_2_billings`, `component_2`. These always come back `togglable: true` and carry an
explicit entity ref (`account_id` = `expense_categories.id`, `component_id` = `lease_components.id`)
so you can group or deep-link without parsing the key. **Treat `fields` as the source of truth.**

### `report.<bucket>.items` — rows

Each row is keyed by the field `key`s, **plus row-level metadata**:

- `type` — `normal` | `subtotal` | `grosstotal` (see below). Data rows are `normal`; the totals row
  is `subtotal` (or `grosstotal` where a report has a distinct grand total). Always present.
- `background_color` — color enum tinting the **whole row** (e.g. `secondary` on the totals row).
  **Optional — omitted when `none`.** A missing key means no row tint.

A row **need not contain every field key.** A key may be omitted because a preceding cell spans it
(`col_span`, below) or because the row simply doesn't populate it — render such columns empty.

Every **cell** (each field-keyed value) is an object. Only `value` is guaranteed:

```jsonc
{ "value": 4800, "color": "danger" }   // fully specified
{ "value": 4800 }                      // no special colour, single column
{ "value": "Income Received", "col_span": 5 }  // spans 5 columns
```

- `value` — the raw value; format it by the field's `format`. A `money` value may be a number or a
  decimal string — coerce to number before formatting.
- `color` — a color enum applied to that individual cell (e.g. an overdue balance in `danger`).
  **Optional — omitted when `none`.** A missing key means no cell colour.
- `col_span` — a positive integer > 1; the cell occupies that many columns and the row **omits** the
  next `col_span − 1` field keys. **Optional — omitted when `1`** (the default). Used for full-width
  banner rows (e.g. a section header spanning the whole table).

> **Payload convention:** absent `color` / `background_color` ⇒ `none`; absent `col_span` ⇒ `1`. The
> backend drops these keys when they hold their default to keep the response small — never assume a
> key is present, and never treat its absence as anything but the default.

### `summary`

Report-level totals for headline tiles (plain scalars, not cells) — e.g. `lease_count`,
`total_billings`, `total_collections`.

---

## Presentation reference

Three placements share **one color palette**, so a dev learns the meanings once:

- **cell `color`** — a single value,
- **field `background_color`** — a whole column,
- **row `background_color`** — a whole row.

### Formats

| `format` | Meaning |
|---|---|
| `string` | Plain text. Left-align. |
| `integer` | Whole number. |
| `money` | Decimal amount in **currency units** (not cents), formatted with the `header` currency. Right-align. |

### Field & row type

| `type` | Meaning | Suggested emphasis |
|---|---|---|
| `normal` | Ordinary data column / data row. | none |
| `subtotal` | A section/period total (e.g. Total Billings) or the per-bucket totals row. | medium weight, tinted row |
| `grosstotal` | A grand / net total (e.g. Gross Total, Balance C/F) or a final grand-total row. | bold, stronger tint |

`type` appears on **both** fields (the column's role) and rows (the row's role). The backend also sets
`weight` on emphasised columns; the template may add its own emphasis keyed off `type`.

### Colors

The enum is `none | primary | secondary | success | danger | warning | info`. Meanings and a
**suggested** Tailwind mapping (adopt one mapping in the template so every report looks consistent —
the backend sends the semantic token, the frontend owns the exact classes):

| Color | Meaning (semantic) | Suggested text class | Suggested bg class |
|---|---|---|---|
| `none` | Default — no emphasis. | inherit | none |
| `primary` | Neutral emphasis / brand highlight. | `text-primary-600` | `bg-primary-50` |
| `secondary` | Muted band, e.g. a totals/footer row. | `text-gray-700` | `bg-gray-100` |
| `success` | Positive — paid, in credit, on target. | `text-green-600` | `bg-green-50` |
| `danger` | Negative — overdue, in arrears, negative balance. | `text-red-600` | `bg-red-50` |
| `warning` | Caution — due soon, partial. | `text-amber-600` | `bg-amber-50` |
| `info` | Informational highlight. | `text-blue-600` | `bg-blue-50` |

### Weight

`weight` is a Tailwind font-weight token (`font-thin` … `font-black`, commonly `font-normal`,
`font-medium`, `font-bold`). The backend applies heavier weights to `subtotal` / `grosstotal` columns.

> The backend chooses `format`, `color`, `weight`, and `background_color` per report — they are
> report-specific presentation choices, not a fixed global scheme. The template must render whatever
> the payload declares, not assume a set.

---

## How to render (report template)

The generic algorithm the reusable frontend template implements, for any report:

1. Read `header.property.currency` (or top-level `header.currency` fallback) for money formatting;
   show `header.period` and active `header.filters` as the report scope.
2. For each bucket you display, build columns from `fields` **in order**; skip `visible: false`
   columns by default and expose `togglable: true` columns in a column picker.
3. For each row in `items`:
   - apply the row's `background_color` if present (whole-row tint); absent ⇒ no tint.
   - walk the `fields` in order keeping a **skip counter**. For each field:
     - if the skip counter is positive, decrement it and render nothing (a previous cell spans here);
     - else read the row's cell for that field `key`. If the key is **absent**, render an empty cell.
       Otherwise apply (in order) the column's `background_color`, then the cell's `color` if present,
       then the field's `weight` and `alignment`; format `value` by the field's `format`. If the cell
       has `col_span: N`, render it across N columns and set the skip counter to `N − 1`.
   - render `type: "subtotal"` / `type: "grosstotal"` rows as footer / emphasised rows.
4. Use `summary` for headline KPI tiles.

Remember: a missing `color` / `background_color` means `none`, and a missing `col_span` means `1`.

**Never hardcode columns or a fixed palette — `fields` and the payload are the source of truth.**

For a concrete, worked example see [billings-and-collections.md](./billings-and-collections.md).
