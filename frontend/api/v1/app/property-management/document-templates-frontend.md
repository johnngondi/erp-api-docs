# Document Templates — Frontend Integration Guide

For the Vue developer building the template designer. This is the contract: what
each endpoint returns, the layout JSON your designer must produce, how to build
the settings forms and Data tab from the registry, and how to preview/print.

**Golden rule:** the backend is the single source of truth. Fetch the registry
and payload schema — never hardcode block lists, field lists, or config options.

All paths are under `/api/v1/app/{company}/property-management/…` and require the
normal auth (Sanctum). `{company}` is the company id in the URL.

---

## 0. The mental model

- A template has a **layout**: a JSON tree of **sections** (CSS-grid containers)
  and **blocks** (leaf content units).
- A **section** has `rows × cols` and a list of **cells**; each cell holds one
  child (a block or a nested section).
- A **block** has a `block` type key and a `config` object. What goes in `config`
  is described by that block's `config_schema` from the registry.
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
        "key": "customer_details",
        "label": "Customer details",
        "description": "Pick fields to show",
        "icon": "user",
        "is_flow": false,
        "document_types": [
          "facility_invoice",
          "facility_receipt",
          "facility_credit_note",
        ],
        "allowed": true,
        "available_fields": [
          {
            "key": "customer.name",
            "default_label": "Customer name",
            "data_path": "customer.name",
            "type": "string",
            "format": null,
          },
          {
            "key": "customer.email",
            "default_label": "Email",
            "data_path": "customer.email",
            "type": "string",
            "format": null,
          },
          {
            "key": "customer.tax_pin",
            "default_label": "KRA PIN",
            "data_path": "customer.tax_pin",
            "type": "string",
            "format": null,
          },
        ],
        "config_schema": {
          "heading_text": {
            "type": "string",
            "label": "Heading text",
            "default": null,
          },
          "show_labels": {
            "type": "boolean",
            "label": "Show field labels",
            "default": true,
          },
          "label_position": {
            "type": "enum",
            "label": "Label position",
            "options": ["left", "top"],
            "default": "left",
          },
          "font_size": {
            "type": "number",
            "label": "Font size",
            "default": 9,
            "min": 6,
            "max": 24,
          },
          "align": {
            "type": "enum",
            "label": "Alignment",
            "options": ["left", "center", "right"],
            "default": "left",
          },
          "fields": {
            "type": "field_list",
            "label": "Fields to show",
            "options": ["customer.name", "customer.email", "customer.tax_pin"],
            "default": ["customer.name", "customer.email", "customer.tax_pin"],
          },
          "field_labels": {
            "type": "label_map",
            "label": "Field label overrides",
            "default": {},
          },
        },
      },
      // … one entry per block valid for this document type
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
  `is_flow`). Only blocks valid for the type are returned.
- **Settings form** for a selected block = render a control per `config_schema`
  entry (see §4).
- `default_page_setup` seeds the Page tab when starting a new template.

> Re-fetch the registry when the user switches `document_type`.

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
      "customer.tax_pin": "string",
      "document.number": "string",
      "document.issued_at": "date",
      "items": "array",
      "items.description": "string",
      "items.unit_price": "money",
      "totals.total": "money",
      // …
    },
    "sample": {
      "company": {
        "name": "Savanna Hardware Ltd",
        "tax_pin": "P051443821R",
        "logo_url": null,
      },
      "customer": { "name": "Wanjiku Traders", "tax_pin": "P051234567X" },
      "document": { "number": "INV-2026-00481", "issued_at": "10 Jun, 2026" },
      "items": [
        {
          "description": "Cement — Bamburi 50kg",
          "quantity": 12,
          "unit_price": "780.00",
          "total": "9,360.00",
        },
      ],
      "totals": { "subtotal": "43,860.00", "total": "50,877.60" },
    },
  },
}
```

- `schema` is a flat `dot-path → type` map. Build the Data tab tree by splitting
  on `.`. `items.*` keys describe the per-row columns of the `items` array.
- A field is **bound** when its key appears in some block's `config.fields` in the
  current layout — use that to render the "used / available" indicators.
- `sample` is the same fake payload the server uses for preview, so your in-app
  preview values match the PDF.

Field `type` values: `string | money | date | datetime | image | number | array`.

---

## 3. The layout JSON you produce

This is what you POST/PATCH as `layout`. The root node is a **section**.

### Section node

```jsonc
{
  "type": "section",
  "rows": 1, // integer ≥ 1
  "cols": 2, // integer ≥ 1
  "flow": false, // true only for the line-items flow section
  "style": {
    // all optional — see §4.2
    "spacing": 8, // number; grid gap AND section padding, in px
    "background_color": "#e7f3f3", // "#rrggbb" or "" (transparent); adds 6px radius
    "vertical_dividers": [
      { "after_col": 1, "line_color": "#ced5de", "line_width": 1 }
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
  "block": "customer_details", // a key from the registry, allowed for this type
  "config": {
    "heading_text": "Billed to",
    "show_labels": true,
    "fields": ["customer.name", "customer.tax_pin"],
    "field_labels": { "customer.tax_pin": "PIN Number" },
    // presentational extras (see §4.2) may be added to any block's config
    "text_spacing": 6,
    "background_color": "#ffffff",
  },
  "style": {
    // optional box border — see §4.2
    "border_enabled": true,
    "border_color": "#cbd5e1", // "#rrggbb"
    "border_width": 1, // px
    "border_sides": ["bottom"], // subset of top|right|bottom|left; empty/absent = all four
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
- `style` (on a section or a block) and `manual_page_break_before` (on a cell) are
  validated strictly — unknown keys / bad colors are rejected. Colors are
  `#rgb`/`#rrggbb` (or `""` for "none"); divider `after_col`/`after_row` must be
  inside the grid.

Only the **`line_items`** block (the one with `is_flow: true`) paginates; put it
in a section you mark `"flow": true`. Other blocks stay on their page. Use a cell's
`manual_page_break_before: true` to force a hard page break before a root-level
section.

---

## 4. Rendering settings forms from `config_schema`

Map each `config_schema` entry to a control by its `type`:

| `type`       | Control                         | Notes                                                                      |
| ------------ | ------------------------------- | -------------------------------------------------------------------------- |
| `string`     | text input                      |                                                                            |
| `text`       | textarea                        | e.g. `free_text.content`                                                   |
| `boolean`    | toggle                          | `default` is a bool                                                        |
| `number`     | number input                    | honour `min`/`max`                                                         |
| `enum`       | select / segmented              | choices in `options`                                                       |
| `field_list` | checkbox list                   | `options` = selectable field keys; value is the **ordered** subset to show |
| `label_map`  | per-field label override inputs | object `{ fieldKey: "Custom label" }`                                      |
| `color`      | color picker                    | value is `#rgb`/`#rrggbb`, or `""` for none                                |

The special keys you'll touch most:

- **`fields`** — which `available_fields` are shown, in order. Omit it to mean
  "all defaults". For `line_items` these are the **columns**.
- **`field_labels`** — per-field label overrides (see §5).
- Common appearance keys present on most blocks: `heading_text`, `show_labels`,
  `label_position`, `font_size`, `align`.

Block-specific extras you'll see in `config_schema`:

- `company_header`: `show_logo`
- `logo`: `align`, `max_height_mm`
- `line_items`: `show_row_numbers`, `zebra`, `background_color` (header fill),
  `label_color` (header text), `label_width_balance`
- `qr_code`: `source` (`field|static`), `field` (payload dot-path), `value`
  (static text/link), `size_mm`, `caption`, `align`
- `barcode`: `source` (`field|value`), `field` (payload dot-path), `value`
  (static text), `size_mm` (width), `height_mm`, `caption`, `align` — encodes
  **Code 128** server-side into an inline SVG
- `divider_line`: `orientation` (`horizontal|vertical`), `line_color`, `line_width`
- `summary_totals`: same fields as `totals`, rendered as a boxed summary
- `free_text`: `content`
- `signature`: `label`, `show_line`, `name`

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

Legacy aliases still accepted: `label_size_balance` (→ `label_width_balance`),
`field_label_overrides` (→ `field_labels`).

When a block sets a `background_color` or a border (below), the renderer adds a
`6px` radius + `6px` padding to the block box automatically.

**Section `style`** (see §3): `spacing` (gap **and** padding), `background_color`,
`vertical_dividers[]` (`after_col`, `line_color`, `line_width`),
`horizontal_dividers[]` (`after_row`, `line_color`, `line_width`).

**Block `style`** (see §3): `border_enabled`, `border_color`, `border_width`,
`border_sides[]` (subset of `top|right|bottom|left`; empty/absent = all four).

**Cell `manual_page_break_before`**: `true` forces a page break before the cell.

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
{ "data": { "message": "Document template created successfully.", "document_template": { "id": 12, "name": "…", "layout": { … }, "is_default": false, … } } }
```

### Validation errors — `422`

Errors are keyed by **the exact path in your layout**, so you can jump the user
to the offending node:

```jsonc
{
  "message": "…",
  "errors": {
    "layout.cells[0].child.config.fields[0]": [
      "layout.cells[0].child.config.fields[0]: unknown field \"customer.foo\" for block \"customer_details\"",
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
| --------------------------------------------- | ------------------------------------------------------- |
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

---

## 9. Quick start checklist

1. On open: `GET registry?document_type=…` → build palette + default page setup.
2. `GET …/{type}/payload-schema` → build the Data tab tree + sample values.
3. User drags blocks into section cells → maintain the **layout tree** (§3).
4. Selecting a block → render its settings form from `config_schema` (§4); writes
   go into that node's `config`.
5. Labels: show `field_labels[key] ?? default_label`; edits go to
   `config.field_labels` (§5).
6. Save via POST/PATCH; on `422`, map error keys back onto nodes (§6).
7. Preview with `POST render?preview=1&format=html`; print with
   `POST render` + `{ resource_id }`.

```

```
