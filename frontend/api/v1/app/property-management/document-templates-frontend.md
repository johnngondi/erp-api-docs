# Document Templates — Frontend Integration Guide

For the Vue developer building the template designer. This is the contract: what
each endpoint returns, the layout JSON your designer must produce, how to build
the settings forms and Data tab from the registry, and how to preview/print.

**Golden rule:** the backend is the single source of truth. Fetch the registry
and payload schema — never hardcode block lists, field lists, or config options.

All paths are under `/api/v1/app/{company}/property-management/…` and require the
normal auth (Sanctum). `{company}` is the company id in the URL.

> **v2 note for anyone who built against the earlier version of this doc:** the
> block set, the canonical payload shape, and several validated keys all changed.
> `customer_details`, `document_details`, `line_items`, `summary_totals`,
> `qr_code`, `barcode`, `signature` and the old receipt-only `payment_details`
> are **gone** — replaced by per-type blocks (§1). The payload no longer has a
> `customer`/`payment` key — see §2. Read this version fresh rather than diffing.

---

## 0. The mental model

- A template has a **layout**: a JSON tree of **sections** (CSS-grid containers)
  and **blocks** (leaf content units).
- A **section** has `rows × cols` and a list of **cells**; each cell holds one
  child (a block or a nested section).
- A **block** has a `block` type key and a `config` object. What goes in `config`
  is described by that block's `config_schema` from the registry.
- Each block also carries a **`presentation`**: `fields | table | image | qr | text | divider`
  — how to draw it, independent of `is_flow` (whether it paginates). Use it to
  pick the right renderer component; don't infer it from the block key.
- The frontend renders the **palette** and every **settings form** from the
  registry. The **Data tab** is rendered from the payload schema.
- Preview and the real PDF run through the same server renderer, so the preview
  is faithful.

---

## 1. Load the palette + settings schema — `registry`

```
GET …/settings/documents/templates/registry?document_type=facility_invoice
```

```jsonc
{
  "data": {
    "document_type": "facility_invoice",
    "blocks": [
      {
        "key": "tenant_details",
        "label": "Tenant details",
        "description": "Tenant name, contacts, unit",
        "icon": "user",
        "presentation": "fields",
        "is_flow": false,
        "document_types": ["facility_invoice", "facility_receipt", "facility_credit_note"],
        "allowed": true,
        "available_fields": [
          { "key": "tenant.name", "default_label": "Tenant name", "data_path": "tenant.name", "type": "string", "format": null },
          { "key": "tenant.lease_id", "default_label": "Lease", "data_path": "tenant.lease_id", "type": "string", "format": null },
          { "key": "tenant.tax_pin", "default_label": "KRA PIN", "data_path": "tenant.tax_pin", "type": "string", "format": null },
          { "key": "tenant.phone", "default_label": "Phone", "data_path": "tenant.phone", "type": "string", "format": null },
          { "key": "tenant.email", "default_label": "Email", "data_path": "tenant.email", "type": "string", "format": null },
          { "key": "tenant.unit", "default_label": "Unit / property", "data_path": "tenant.unit", "type": "string", "format": null },
        ],
        "config_schema": {
          "heading_text": { "type": "string", "label": "Heading text", "default": null },
          "show_labels": { "type": "boolean", "label": "Show field labels", "default": true },
          "label_position": { "type": "enum", "label": "Label position", "options": ["left", "top"], "default": "left" },
          "font_size": { "type": "number", "label": "Font size", "default": 9, "min": 6, "max": 24 },
          "align": { "type": "enum", "label": "Alignment", "options": ["left", "center", "right"], "default": "left" },
          "fields": { "type": "field_list", "label": "Fields to show", "options": ["tenant.name", "…"], "default": ["tenant.name", "…"] },
          "field_labels": { "type": "label_map", "label": "Field label overrides", "default": {} },
        },
      },
      // … one entry per block valid for this document type — see §1.1 for the full 25-block catalog
    ],
    "default_page_setup": {
      "paper": "A4",
      "orientation": "portrait",
      "margins": { "top": 15, "right": 15, "bottom": 15, "left": 15 },
      "repeat_header": true,
    },
  },
}
```

- **Palette** = `data.blocks` (each has `label`, `description`, `icon`,
  `presentation`, `is_flow`). Only blocks valid for the type are returned.
- **Settings form** for a selected block = render a control per `config_schema`
  entry (see §4).
- `default_page_setup` seeds the Page tab when starting a new template.

> Re-fetch the registry when the user switches `document_type`.

### 1.1 The 25 blocks

**Shared / layout — all four types**

| Key | Presentation | Fields / config |
| --- | --- | --- |
| `logo` | image | `company.logo_url`; `max_height_mm`, `min_height_mm` (bounds the `<img>` itself — distinct from the `style.min_height` every block also accepts, §4.3), `align`, `source_url` (absolute http(s) URL that replaces the company logo for this template; `""` = use `company.logo_url`; other schemes are a `422`) |
| `company_header` | fields | `company.name`, `.tax_pin`, `.address`, `.phone`, `.email`, `.website`, `.tagline`. No logo toggle — logo is its own block. |
| `free_text` | text | `heading_text` + static `content` |
| `divider_line` | divider | `orientation`, `line_color`, `line_width`, `margin_top`/`margin_bottom` (px nudge, not a flow margin) |

**Party blocks**

| Key | Types | Fields |
| --- | --- | --- |
| `landlord_details` | all 4 | `landlord.name`, `.tax_pin`, `.phone`, `.email`, `property.name`, `.address`, `.lr_number` |
| `tenant_details` | invoice, receipt, credit note | `tenant.name`, `.lease_id`, `.tax_pin`, `.phone`, `.email`, `.unit` |
| `vendor_details` | lpo | `vendor.name`, `.tax_pin`, `.phone`, `.email`, `.address` |

**Per-type detail blocks** (fixed `document.*` fields, no per-type variance in the class)

| Key | Type | Fields |
| --- | --- | --- |
| `invoice_details` | invoice | `document.number`, `.issued_at`, `.due_at`, `.status`, `.currency` |
| `credit_note_details` | credit note | `document.number`, `.issued_at`, `.status`, `.currency`, `.invoice_number` |
| `receipt_details` | receipt | `document.number`, `.issued_at`, `.served_by`, `.status`, `.currency` |
| `lpo_details` | lpo | `document.number`, `.issued_at`, `.due_at` (delivery), `.status`, `.currency` |

**Content blocks**

| Key | Types | Presentation | Notes |
| --- | --- | --- | --- |
| `fiscal_qr` | invoice, credit note | qr | Fixed to `document.verify_url`; config `size_mm`, `align`, `caption`, `show_cu_invoice_number`, `show_cu_serial_number` (stacks `document.cu_invoice_number`/`.cu_serial_number` beneath the code) |
| `document_notes` | invoice, credit note, lpo | text | Binds `document.notes` |
| `invoice_items` | invoice | table, **flow** | `items.notes`, `.quantity`, `.amount`, `.tax`, `.total`; `show_utility_readings` (default true) renders meter-reading images beneath a row where `is_utility_bill` is true and at least one reading carries an `image_url` — a utility line with no captured images adds no extra row |
| `credit_note_items` | credit note | table, **flow** | `items.notes`, `.quantity`, `.amount`, `.tax`, `.total` |
| `lpo_items` | lpo | table, **flow** | `items.title`, `.notes`, `.quantity`, `.amount`, `.tax`, `.total` |
| `totals` | invoice, credit note, lpo | fields | `totals.amount`, `.tax`, `.total`, `.paid`, `.balance` — **no receipt**. Renders through the shared field-list, so every fields-styling key applies; registry defaults keep the classic look (`value_align: "right"`, `field_styles: { "totals.total": { bold: true } }`), both editable per template |
| `tax_summary` | invoice, credit note, lpo | table | `tax_summary.name`, `.rate`, `.taxable`, `.tax` |
| `payments` | invoice | table | `payments.type`, `.number`, `.date`, `.method`, `.reference`, `.amount` |
| `invoice_ageing` | invoice | table | `ageing.label`, `.amount`; `bucket_layout` (`columns` default \| `rows`) — `columns` transposes the table into the classic ageing analysis: one column per bucket, amount beneath its label |
| `payment_details` | invoice | table | `bank_accounts.component`, `.bank_name`, `.branch`, `.account_name`, `.account_number` — where to pay; `display` (`fields` default \| `table`) — `fields` renders each account as an inline label/value group (the tenant_details look), `table` keeps the column grid |
| `bank_account_details` | receipt | fields | `payment_account.bank_name`, `.branch`, `.account_name`, `.account_number` — where the payment landed |
| `payment_transactions` | receipt | table | `transactions.transaction_date`, `.transaction_number`, `.method`, `.amount` |
| `invoice_allocations` | credit note, receipt | table | `allocations.invoice_number`, `.invoice_date`, `.invoice_total`, `.allocated`, `.balance` |
| `signatories` | **lpo only** | fields | See §1.3 |

Only the three `*_items` blocks (`invoice_items`, `credit_note_items`, `lpo_items`)
have `is_flow: true` and paginate. Every other table block is fixed-height (use
`min_rows` if you want it to look populated regardless — §4.3).

### 1.2 Resolving `presentation` → renderer component

Dispatch on `presentation`, never on the block `key` — the backend can add a
new block to any of these six buckets without a frontend change as long as you
resolve this way:

| `presentation` | What it draws | Data it reads |
| --- | --- | --- |
| `fields` | A label/value list (vertical, or `label_position: "top"`), one row per entry in `config.fields`, in that order | `data_get(payload, field.key)` for each of the block's `available_fields[]` |
| `table` | A header row (from `config.fields`) + one row per item in the payload array the block is bound to (`items`, `tax_summary`, `payments`, `ageing.buckets`, `bank_accounts`, `payment_account`-adjacent `transactions`, `allocations` — see §2.1 for which array each block reads) | The matching top-level payload array; `config.min_rows`/`min_row_height` pad blank trailing rows so a short table still looks ruled |
| `image` | A single `<img>` (today only `logo`) | `payload.company.logo_url`, sized by `config.max_height_mm`/`min_height_mm` |
| `qr` | A generated QR code + optional caption/CU-number lines (today only `fiscal_qr`) | `payload.document.verify_url` (+ `.cu_invoice_number`/`.cu_serial_number` when their `show_*` toggles are on) |
| `text` | Free-flowing text — `free_text`'s static `config.content`, or `document_notes`'s `payload.document.notes` | Static config for `free_text`; a payload field for `document_notes` |
| `divider` | A horizontal or vertical rule, no data binding | none — `config.orientation`/`line_color`/`line_width`/`margin_top`/`margin_bottom` only |

`signatories` is the one exception: it reports `presentation: "fields"` but has
no `available_fields`/`config.fields` at all — its N signature boxes are fully
computed server-side from `payload.approval_chain[]` (§1.3). Detect it by
`key === "signatories"`, not by presentation, and render it as its own
component.

### 1.3 `signatories` (LPO only)

Not a fields/table hybrid — it renders N signature boxes computed from
`payload.approval_chain[]` (§2.1) at render time, driven by one config key:

| Config | Type | Default | Meaning |
| --- | --- | --- | --- |
| `signatory_count` | number, 1–8 | 3 | How many signatories to show |
| `show_date` | boolean | true | Show the date under each name |
| `align` | enum | `left` | Box alignment |

The selection is **Created / Reviewed / Approved** — position 1 is always the
procurement request's creator; for `signatory_count ≥ 2` the last approval step
is labeled Approved; for `≥ 3` the second-to-last is labeled Reviewed; beyond
that, extra slots are appended (still labeled Approved) pulling from earlier in
the chain. A short chain renders fewer than `signatory_count` — it never pads
with blank boxes. There's no field list to configure here; the algorithm is
fixed server-side.

---

## 2. Build the Data tab — `payload-schema`

```
GET …/settings/documents/templates/facility_invoice/payload-schema
```

```jsonc
{
  "data": {
    "document_type": "facility_invoice",
    "schema": {
      "company.name": "string",
      "company.logo_url": "image",
      "landlord.name": "string",
      "property.name": "string",
      "tenant.tax_pin": "string",
      "document.number": "string",
      "document.issued_at": "date",
      "items": "array",
      "items.notes": "string",
      "items.amount": "money",
      "totals.total": "money",
      "tax_summary": "array",
      "payments": "array",
      "ageing.label": "string",
      "bank_accounts": "array",
      // … full dot-path map, see §2.1 for the shape per type
    },
    "sample": {
      "company": { "name": "Savanna Hardware Ltd", "tax_pin": "P051443821R", "logo_url": null },
      "landlord": { "name": "Kariuki Estates Ltd" },
      "property": { "name": "Westlands Court" },
      "tenant": { "name": "Wanjiku Traders", "tax_pin": "P051234567X" },
      "document": { "number": "INV0481", "issued_at": "10 Jun, 2026" },
      "items": [
        { "notes": "Rent — June 2026", "quantity": 1, "amount": "43,860.00", "total": "50,877.60" },
      ],
      "totals": { "amount": "43,860.00", "total": "50,877.60" },
      // …
    },
  },
}
```

- `schema` is a flat `dot-path → type` map. Build the Data tab tree by splitting
  on `.`. `items.*`/`tax_summary.*`/`payments.*`/etc. keys describe the per-row
  columns of their respective arrays.
- A field is **bound** when its key appears in some block's `config.fields` in the
  current layout — use that to render the "used / available" indicators.
- `sample` is the same fake payload the server uses for preview — when the
  template belongs to a company with real letterhead data, `sample.company`
  reflects it (name/PIN/address/logo), so the designer preview shows the user's
  own branding from the first render, not a placeholder.

Field `type` values: `string | money | date | datetime | image | number | array | badge`.

> **`badge` values are objects**, not strings: `{ value, label, color }`, where
> `color` is a semantic name (`success | warning | danger | primary | secondary | info`)
> taken from the status enum. Renderers draw them as colored pills — the shared
> field-list does this for `document.status` on every `*_details` block — and
> the Data tab shows the `label`.

### 2.1 Canonical payload shape (per top-level key)

```
company:        { name, tax_pin, address, phone, email, website, tagline, logo_url }
landlord:       { name, tax_pin, phone, email }
property:       { name, address, lr_number, city, country }
tenant:         { name, lease_id, tax_pin, phone, email, unit }        # invoice/cn/receipt
vendor:         { name, tax_pin, phone, email, address }               # lpo
document:       { number, raw_number, issued_at, due_at, status, notes, currency,
                   invoice_number, cu_invoice_number, cu_serial_number, verify_url, served_by }
                # status is a badge object { value, label, color }, not a string
items[]:        { description, notes, quantity, unit_price, amount, tax, tax_rate, total,
                   is_utility_bill,
                   previous_reading: { value, read_at, image_url },
                   current_reading:  { value, read_at, image_url } }
totals:         { amount, tax, total, paid, balance }
tax_summary[]:  { name, rate, taxable, tax }
payments[]:     { type, number, date, method, reference, amount }      # invoice
allocations[]:  { invoice_number, invoice_date, invoice_total, allocated, balance }  # cn/receipt
ageing:         { as_at, buckets: [ { label, amount } ] }              # invoice
bank_accounts[]:{ component, bank_name, branch, account_name, account_number }  # invoice —
                # filtered to THIS invoice: only accounts collecting a component that is on the
                # items (catch-all accounts always qualify, labeled "All charges"); each distinct
                # account appears once, `component` listing the invoiced components it collects
payment_account:{ bank_name, branch, account_name, account_number }    # receipt
transactions[]: { transaction_date, transaction_number, method, amount }  # receipt
approval_chain[]: { name, date }                                       # lpo — raw, see §1.3
```

There is **no `customer` key and no `payment` key** — those were replaced by
`tenant`/`landlord` and `payment_account`/`payment_details`/`bank_account_details`
respectively. `document.number` is a formatted, zero-padded id
(`INV0001`/`CN0112`/`RCP0481`/`LPO0219`) computed at render time — don't expect
it to match any number you send elsewhere.

---

## 3. The layout JSON you produce

This is what you POST/PATCH as `layout`. The root node is a **section**.

### Section node

```jsonc
{
  "type": "section",
  "rows": 1, // integer ≥ 1
  "cols": 2, // integer ≥ 1
  "flow": false, // true only for a *_items flow section
  "style": {
    // all optional
    "spacing": 8, // number; grid gap AND section padding, in px
    "background_color": "#e7f3f3", // "#rrggbb" or "" (transparent); adds 6px radius
    "min_height": 25, // mm — forces the section to at least this tall (§4.3)
    "margin_top": 0, // px
    "margin_bottom": 0, // px
    "label": "Header", // designer-only, never rendered
    "vertical_dividers": [
      { "after_col": 1, "line_color": "#ced5de", "line_width": 1, "margin_left": 0, "margin_right": 0 }
    ],
    "horizontal_dividers": [
      { "after_row": 1, "line_color": "#ced5de", "line_width": 1 }
    ],
  },
  "cells": [
    {
      "row": 1,
      "col": 1, // 1-based, within the grid
      "row_span": 1,
      "col_span": 1,
      "valign": "top", // top | middle | bottom
      "manual_page_break_before": false, // force a page break before this cell
      "child": {
        /* block or nested section, or null for an empty cell */
      },
    },
  ],
}
```

### Block node

```jsonc
{
  "type": "block",
  "block": "tenant_details", // a key from the registry, allowed for this type
  "config": {
    "heading_text": "Billed to",
    "show_labels": true,
    "fields": ["tenant.name", "tenant.tax_pin"],
    "field_labels": { "tenant.tax_pin": "PIN Number" },
    // presentational extras (see §4.2) may be added to any block's config
    "text_spacing": 6,
    "background_color": "#ffffff",
  },
  "style": {
    // optional — box border and/or min-height, see §4.2/§4.3
    "border_enabled": true,
    "border_color": "#cbd5e1", // "#rrggbb"
    "border_width": 1, // px
    "border_sides": ["bottom"], // subset of top|right|bottom|left; empty/absent = all four
    "min_height": 20, // mm
  },
}
```

Rules the server enforces (so enforce them in the editor too):

- `rows`/`cols` ≥ 1; every cell's `row`/`col` within bounds; spans must not
  overflow the grid; **cells must not overlap**.
- `block` must exist in the registry **and be allowed for the document type**.
- Every key in `config.fields` (and every key of `config.field_labels`) must be
  one of that block's `available_fields[].key`.
- `config` may contain the keys in the block's `config_schema` **plus** the shared
  presentational keys (§4.2), each with the right value type. Any other key is a
  `422`.
- **Numeric config values are clamped into their spec's `min`/`max`, not
  rejected** — if your editor already clamps client-side, the two will agree; if
  you send an out-of-range number anyway, the server silently clamps it and the
  *saved* template reflects the clamped value (re-read the response, don't
  assume an echo of what you sent).
- `style` (on a section or a block) and `manual_page_break_before` (on a cell) are
  validated strictly — unknown keys / bad colors are rejected (not clamped).
  Colors are `#rgb`/`#rrggbb` (or `""` for "none"); divider `after_col`/`after_row`
  must be inside the grid.

Only blocks with `is_flow: true` (the three `*_items` blocks) paginate; put one
in a section you mark `"flow": true`. Other blocks stay on their page. Use a
cell's `manual_page_break_before: true` to force a hard page break before a
root-level section.

---

## 4. Rendering settings forms from `config_schema`

Map each `config_schema` entry to a control by its `type`:

| `type`       | Control                         | Notes                                                                      |
| ------------ | -------------------------------- | --------------------------------------------------------------------------- |
| `string`     | text input                      |                                                                            |
| `text`       | textarea                        | e.g. `free_text.content`                                                   |
| `boolean`    | toggle                          | `default` is a bool                                                        |
| `number`     | number input                    | honour `min`/`max` — the server clamps anyway, but match it client-side    |
| `enum`       | select / segmented              | choices in `options`                                                       |
| `field_list` | checkbox list                   | `options` = selectable field keys; value is the **ordered** subset to show |
| `label_map`  | per-field label override inputs | object `{ fieldKey: "Custom label" }`                                      |
| `color`      | color picker                    | value is `#rgb`/`#rrggbb`, or `""` for none                                |
| `url`        | text input                      | absolute `http(s)` URL, or `""` for unset; other schemes are rejected      |

The special keys you'll touch most:

- **`fields`** — which `available_fields` are shown, **in that order** (the
  renderer honours the list's order, so reordering in the designer reorders the
  PDF). Omit it to mean "all defaults"; an explicit `[]` means "show none". For
  table-presentation blocks these are the **columns**.
- **`field_labels`** — per-field label overrides (see §5).
- Common appearance keys present on most blocks: `heading_text`, `show_labels`,
  `label_position`, `font_size`, `align`.

Block-specific extras you'll see in `config_schema`:

- `logo`: `align`, `max_height_mm`, `min_height_mm`, `source_url` (absolute URL logo override)
- `fiscal_qr`: `size_mm`, `align`, `caption`, `show_cu_invoice_number`, `show_cu_serial_number`
- `divider_line`: `orientation` (`horizontal|vertical`), `line_color`, `line_width`, `margin_top`, `margin_bottom`
- `free_text`: `content`
- `signatories`: `signatory_count`, `show_date`, `align` (§1.3)
- **Every table-presentation block** (`invoice_items`, `credit_note_items`,
  `lpo_items`, `tax_summary`, `payments`, `invoice_ageing`, `payment_details`,
  `payment_transactions`, `invoice_allocations`): `min_rows`, `min_row_height`,
  `show_row_numbers`, `zebra`, `background_color` (header fill), `label_color`
  (header text), `table_borders` (`horizontal` default | `vertical` | `all` |
  `none` — which cell rules draw), `table_border_color` (recolors the drawn
  rules; empty = default greys), `corner_radius` (px, 0–24 clamped — rounds the
  table's corners; 0 = square), plus `show_utility_readings` on `invoice_items`
  only and `bucket_layout` on `invoice_ageing` only (§1.1).

Always read `default` from the schema to initialize a freshly-dropped block.

### 4.2 Shared presentational keys & styling

Beyond each block's own `config_schema`, the server accepts a **global set of
presentational keys** on any block's `config`, plus `style` objects on sections
and blocks. Don't hardcode these against a block — they're accepted everywhere and
rendered where they make sense.

**Presentational `config` keys (any block):**

| Key                   | Type     | Effect                                                            |
| --------------------- | -------- | ---------------------------------------------------------------- |
| `text_color`          | color    | Value text color                                                 |
| `label_color`         | color    | Label text color                                                 |
| `background_color`    | color    | Block background fill (`""` = transparent)                       |
| `text_spacing`        | number   | 0–16 → line-height `1.2 + clamp(x,0,16)/20`                       |
| `label_width_balance` | number   | −4…4, shifts the label/value column split (0 = balanced)         |
| `label_align`         | enum     | `left\|center\|right` — label alignment                          |
| `value_align` / `text_align` | enum | value alignment                                              |
| `padding`             | number   | 0–48 px (clamped); explicit inner padding, overriding the automatic 6px a fill/border adds — `0` removes it, and it applies even with no fill/border |
| `field_styles`        | map      | `{ fieldKey: { bold?: bool, color?: hex, font_size?: pt, separator?: bool, separator_width?: px } }` — per-field value emphasis and an optional horizontal rule under that field's row, rendered on fields-presentation blocks (payment_details' inline display included); `font_size` clamps to 6–24, `separator_width` to 1–8 (absent = 1), keys must be that block's fields, anything else is a `422` |

Legacy aliases still accepted: `label_size_balance` (→ `label_width_balance`),
`field_label_overrides` (→ `field_labels`).

When a block sets a `background_color` or a border (below), the renderer adds a
`6px` radius + `6px` padding to the block box automatically. Table-presentation
blocks paint their own header surface, so a block-level `background_color` only
tints the table header, not the whole box.

**Section `style`**: `spacing` (gap **and** padding), `background_color`,
`min_height` (mm), `margin_top`/`margin_bottom` (px), `label` (designer-only,
ignored at render), `vertical_dividers[]` (`after_col`, `line_color`,
`line_width`, `margin_left`, `margin_right` — the last two nudge the divider
sideways, they don't add flow spacing), `horizontal_dividers[]` (`after_row`,
`line_color`, `line_width`).

**Block `style`**: `border_enabled`, `border_color`, `border_width`,
`border_sides[]` (subset of `top|right|bottom|left`; empty/absent = all four),
`min_height` (mm — forces the block's box to exist and be at least this tall,
even with no border/background set).

**Cell `manual_page_break_before`**: `true` forces a page break before the cell.

### 4.3 Min-height & blank-row padding

Two independent mechanisms, both mm-based:

| What | Where | Effect |
| --- | --- | --- |
| A section or block looking too short | `style.min_height` | CSS `min-height` on the section/block box |
| A table looking sparse | `config.min_rows` + `config.min_row_height` | Pads blank rows up to `min_rows`, each `min_row_height` tall, so the table keeps a consistent ruled cadence regardless of row count |

Use both on a facility's first invoice — a 2-line invoice with `min_rows: 8` on
`invoice_items` and `min_height: 90` on its section prints with the same
visual weight as a full one.

---

## 5. Labels (how the label next to a value is chosen)

Resolution order, per field:

1. `config.field_labels[fieldKey]` — the per-template override the user types in
   the inspector.
2. otherwise the field's `default_label` from the registry.

So to show a label in the editor, use `field_labels[key] ?? default_label`. To
override, write into `config.field_labels`. (Localization of defaults is a
server-side extension point; you don't manage it.)

`show_labels: false` hides labels entirely; `label_position` places them
`left` of or `top`-aligned above the value.

---

## 6. Save (create / update)

```
POST  …/settings/documents/templates
PATCH …/settings/documents/templates/{id}
```

Request body:

```jsonc
{
  "name": "Counter Receipt — A4",
  "document_type": "facility_invoice", // POST only; immutable after create
  "facility_ids": null, // null = all facilities; or [10, 20]
  "is_default": false,
  "is_active": true,
  "page_setup": {
    "paper": "A4",
    "orientation": "portrait",
    "margins": { "top": 15, "right": 18, "bottom": 15, "left": 18 },
    "repeat_header": true,
  },
  "layout": {
    /* the section/block tree from §3 */
  },
}
```

- Do **not** send `company_id` or `created_by` — the server sets them.
- `document_type` is fixed once created; PATCH ignores it.

Success (single-item envelope):

```jsonc
{ "data": { "message": "Document template created successfully.", "document_template": { "id": 12, "name": "…", "layout": { … }, "is_default": false, "schema_version": 2, … } } }
```

**Read `layout` back from the response** — as noted in §3, out-of-range numeric
config values are clamped server-side, so the persisted layout can differ from
what you sent.

### Validation errors — `422`

Errors are keyed by **the exact path in your layout**, so you can jump the user
to the offending node:

```jsonc
{
  "message": "…",
  "errors": {
    "layout.cells[0].child.config.fields[0]": [
      "layout.cells[0].child.config.fields[0]: unknown field \"tenant.foo\" for block \"tenant_details\"",
    ],
    "layout.cells[2].row": [
      "layout.cells[2].row: Cell row 5 is outside the grid (1–1).",
    ],
  },
}
```

Parse the keys (`cells[i]`, `child`, `config`, `fields[j]`) to highlight the node
in the canvas.

---

## 7. List / pick / default / duplicate

```
GET    …/settings/documents/templates?filter[document_type]=facility_invoice
GET    …/settings/documents/templates?filter[facility_id]=10      // company-wide + facility 10's
POST   …/settings/documents/templates/{id}/set-default            // unsets the previous default
POST   …/settings/documents/templates/{id}/duplicate              // clones (never default)
DELETE …/settings/documents/templates/{id}                        // soft delete
```

- **Multiple templates per `document_type` are expected** — the list is your
  print picker. `is_default` only marks the auto-selected one.
- `filter[facility_id]=X` returns templates usable by that facility (company-wide
  `facility_ids: null` **plus** those whose array contains `X`).
- List is paginated: `{ data: [...], links, meta }`. Pass `?per_page=`.

Each list/show item (`DocumentTemplateResource`) includes:
`id, name, document_type, document_type_label, facility_ids,
applies_to_all_facilities, is_default, is_active, page_setup, layout,
schema_version, created_at, updated_at`.

---

## 8. Preview & print

One endpoint, used for both — preview and PDF go through the same renderer so
they can't drift.

```
POST …/document-templates/{id}/render
```

| Want                                          | Query / body                                             |
| --------------------------------------------- | ---------------------------------------------------------- |
| **PDF of a real document**                    | body `{ "resource_id": 481 }` → `application/pdf` stream |
| **Preview with sample data**                  | `?preview=1` (no `resource_id` needed) → PDF            |
| **Preview as HTML** (for an in-canvas iframe) | `?preview=1&format=html` → `text/html`                  |
| **Real document as HTML**                     | `?format=html` + `{ "resource_id": 481 }`               |

- `resource_id` is the id of the underlying document (invoice/receipt/lpo/credit
  note). The server authorizes that the template's company matches and that it's
  usable by the document's facility; a foreign/invalid id returns `422` with the
  error keyed on `resource_id` and a type-specific message (e.g. `Receipt not
  found for this company.`). The legacy `model_id` key is still accepted as a
  fallback.
- For a live designer preview, save (or use a transient render of) the current
  layout, then hit `?preview=1&format=html` and drop the HTML into an iframe.
  The preview's `sample.company` reflects the current company's real letterhead
  (§2), so this is a faithful first look, not a generic placeholder.

---

## 9. Quick start checklist

1. On open: `GET registry?document_type=…` → build palette + default page setup.
2. `GET …/{type}/payload-schema` → build the Data tab tree + sample values.
3. User drags blocks into section cells → maintain the **layout tree** (§3).
4. Selecting a block → render its settings form from `config_schema` (§4); writes
   go into that node's `config`.
5. Labels: show `field_labels[key] ?? default_label`; edits go to
   `config.field_labels` (§5).
6. Save via POST/PATCH; on `422`, map error keys back onto nodes (§6); on
   success, re-read `layout` for any server-side clamping.
7. Preview with `POST render?preview=1&format=html`; print with
   `POST render` + `{ resource_id }`.
