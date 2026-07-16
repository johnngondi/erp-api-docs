# Customizable Document Templates

A document-template designer backend. Users visually compose printable layouts
for four document types — **Invoice, Receipt, LPO, Credit Note** — out of nested
**sections** (CSS-grid containers) and **blocks** (predefined content units),
save them as named **templates**, and pick one at print time to produce a PDF.

- The layout is a **declarative JSON tree**, never rendered HTML.
- A server-authoritative **Block Registry** is the single source of truth for
  block types, their fields, default labels and config schemas. The frontend
  renders its entire palette + settings forms from the registry; it hardcodes
  nothing.
- Rendering is pure: `template layout + payload → PDF`. The payload comes from
  running the real model through its existing API Resource, re-shaped into a
  canonical form by a per-type **PayloadMapper**.

---

## 1. Architecture at a glance

```
                    ┌─────────────────────────────────────────────┐
  Vue designer ───► │  GET  …/settings/documents/templates/registry        │  palette + config schemas
                    │  GET  …/settings/documents/templates/{type}/payload-schema │  canonical fields + sample
                    │  CRUD …/settings/documents/templates                 │  store/update validate layout
                    │  POST …/document-templates/{t}/render                 │  PDF / preview
                    └─────────────────────────────────────────────┘
                                          │
   layout JSON ──► LayoutValidator ──► persisted (document_templates.layout)
                                          │
   print ─► DocumentRenderer ─► PayloadMapper(model→canonical) ─► walk tree ─► Blade per block ─► Browsershot ─► PDF
                                     ▲                                  ▲
                                  Resource                        BlockRegistry
```

Key files:

| Concern | Path |
| --- | --- |
| Model | `app/Models/DocumentTemplate.php` |
| Config (type map, page setup, PDF) | `config/document_templates.php` |
| Block registry | `app/Services/DocumentTemplates/BlockRegistry.php` |
| Block types | `app/Services/DocumentTemplates/Blocks/` |
| Fields | `app/Services/DocumentTemplates/Fields/Field.php` |
| Payload mappers | `app/Services/DocumentTemplates/Payload/` |
| Layout validation | `app/Services/DocumentTemplates/LayoutValidator.php` |
| Rendering | `app/Services/DocumentTemplates/DocumentRenderer.php` |
| QR generation | `app/Services/DocumentTemplates/QrCodeGenerator.php` |
| Blade partials | `resources/views/document-blocks/`, `resources/views/documents/template.blade.php` |
| Data (DTO/validation) | `app/Data/PropertyManagement/DocumentTemplate/` |
| Actions | `app/Actions/PropertyManagement/DocumentTemplate/` |
| Controllers | `…/Settings/Document/DocumentTemplateController.php`, `…/PropertyManagement/DocumentTemplateRenderController.php` |
| Policy | `app/Policies/DocumentTemplatePolicy.php` |
| Seeder | `database/seeders/DocumentTemplateSeeder.php` |

---

## 2. Data model

`document_templates` (migration: `database/migrations/2026_06_29_000000_create_document_templates_table.php`):

| Column | Type | Notes |
| --- | --- | --- |
| `name` | string | |
| `document_type` | string | one of `facility_invoice`, `facility_receipt`, `facility_lpo`, `facility_credit_note`. Stable key, resolved to model/resource/mapper via config (keeps JSON portable). |
| `company_id` | FK, not null | hard tenant boundary (via `CompanyScoped`). |
| `facility_ids` | json, nullable | applicability **within** the company: `NULL` ⇒ all facilities; array ⇒ only those. Not row-level tenancy. |
| `created_by` | FK users, nullable | |
| `is_default` | bool | at most **one default per (`company_id`, `document_type`)** — enforced in code, not a DB unique. |
| `is_active` | bool | |
| `page_setup` | json | paper, orientation, margins (mm), repeat-header. |
| `layout` | json | the section/block tree. |
| `schema_version` | uint | stamped on save for forward migration. |
| timestamps + soft deletes | | |

Index: `(company_id, document_type, is_active)`.

> **Versioning:** a `document_template_versions` table is intentionally deferred
> (TODO in the migration). `schema_version` is stamped now so layouts can be
> migrated forward later.

### Model scopes & helpers (`DocumentTemplate`)

- `forDocumentType($key)`, `active()`, `defaults()`
- `usableByFacility($facilityId)` → `facility_ids IS NULL OR JSON_CONTAINS(facility_ids, $id)`
- `appliesToAllFacilities()`, `modelClass()`, `resourceClass()`, `mapperClass()`
- `makeDefault()` — in a transaction, unsets any sibling default for the same
  `(company_id, document_type)`, then sets this one.

### Applicability & default resolution

- `company_id` is the hard boundary. `facility_ids` narrows applicability.
- The print picker / `index?filter[facility_id]=…` returns company-wide **plus**
  that facility's templates (`usableByFacility`).
- Default at print time: prefer a facility-specific default, else the
  company-wide (`facility_ids = NULL`) default.

---

## 3. The Block Registry

Server-authoritative. Resolved as a singleton in `AppServiceProvider` with an
explicit list of block instances.

### Field (`Fields/Field.php`)

A toggleable field a block exposes: `key` (dot-path into the canonical payload,
e.g. `customer.tax_pin`), `default_label` (passed through `__()`), `data_path`,
`type` (`string|money|date|datetime|image|number|array`), optional `format`.

### BlockType (`Blocks/BlockType.php`)

Each block declares:

- `key()`, `label()`, `description()`, `icon()`
- `documentTypes()` — which types it applies to (shared blocks use the
  `SharedAcrossDocuments` trait → all configured types)
- `availableFields()` — `Field[]` the user can toggle
- `configSchema()` — declarative options the frontend renders a form from and
  `LayoutValidator` checks against
- `view()` — Blade partial (`document-blocks.{key}`)
- `isFlow()` — only `line_items` returns true (the paginating section)
- `resolveLabel($fieldKey, $config)` — 3-tier label resolution (see §5)

### Blocks shipped

| Block | Types | Notes |
| --- | --- | --- |
| `company_header` | all | logo + name/PIN/address fields |
| `logo` | all | image block from `company.logo_url` |
| `customer_details` | invoice, receipt, credit note | the "Billed to" party |
| `tenant_details` | invoice, receipt, credit note | tenant name/contacts + `tenant.unit` |
| `vendor_details` | lpo | the counterparty for LPOs |
| `document_details` | all | number, date, served-by, status, notes |
| `receipt_details` | receipt | receipt no./date/method (binds existing `document.*`/`payment.*` paths) |
| `line_items` | all | **flow** section — paginates across pages |
| `totals` | all | subtotal/discount/VAT/total/paid/balance |
| `summary_totals` | all | same `totals.*` fields, boxed summary presentation |
| `payment_details` | receipt | method, reference, account |
| `qr_code` | all | generated server-side (see §6) |
| `barcode` | all | Code 128 SVG generated server-side (see §6) |
| `free_text` | all | notes/terms/footer |
| `signature` | all | name + signature line |
| `divider_line` | all | horizontal/vertical rule |

### Serialized form (`BlockRegistry::toArray($documentType)`)

Each block →
`{ key, label, description, icon, is_flow, document_types, allowed, available_fields[], config_schema }`.
This is exactly what the designer palette + Data tab consume.

---

## 4. Canonical payload & mappers

The frontend Data tab and all blocks bind against a **canonical payload shape**,
not raw resource keys:

```
company:  { name, tax_pin, address, phone, email, logo_url }
customer: { name, email, phone, address, tax_pin }     # invoice/receipt/credit note
tenant:   { name, phone, email, unit }                 # invoice/receipt/credit note
vendor:   { name, email, phone, address, tax_pin }     # lpo
document: { number, reference, issued_at, due_at, served_by, status, notes, verify_url }
items:    [ { description, quantity, unit_price, tax, total } ]
totals:   { subtotal, discount, tax, total, paid, balance }
payment:  { method, reference, account }                # receipt
```

A per-type `PayloadMapper` (`Payload/*PayloadMapper.php`) re-shapes the existing
Resource/model output into this shape. **Mappers re-shape only — they never
recompute money/field logic** (money/totals are taken from the Resource output;
identity fields are read from the eager-loaded relations declared in config
`with`). Each mapper exposes:

- `map(Model, array $resource): array` — real payload
- `static schema(): array` — `dot-path => type`, drives the payload-schema endpoint
- `static sample(): array` — representative fake payload for preview

> **Note:** the identity dot-paths (e.g. `lease.facility.company.kra_pin`) are
> best-effort and isolated to the mappers. If a column name differs, fix it in
> one place.

---

## 5. Labels — three-tier resolution

In priority order (implemented in `BlockType::resolveLabel()` and used by every
partial):

1. **Per-template override** — `config.field_labels[fieldKey]` in the layout JSON.
2. **Registry default** — the field's `default_label`, run through `__()`.
3. *(extension point)* localization via translation keys.

Blocks also honour `show_labels`, `label_position` (`left`/`top`), `heading_text`,
`font_size`, `align` from their `configSchema`.

---

## 6. QR & barcode generation

`qr_code` config:

```jsonc
{ "source": "field", "field": "document.verify_url", "size_mm": 28, "caption": "Scan to verify", "align": "center" }
// or
{ "source": "static", "value": "https://…" }
```

`QrCodeGenerator::render($value, $px)` produces an **inline SVG** via
`bacon/bacon-qr-code` (offline, embeds in the PDF). If the library is missing it
falls back to a network QR `<img>` (prefer the library for offline PDFs).

`barcode` config (`source` is `field|value`):

```jsonc
{ "source": "field", "field": "document.number", "size_mm": 60, "height_mm": 15, "caption": "Scan at till", "align": "center" }
```

`BarcodeGenerator::render($value, $widthPx, $heightPx)` encodes **Code 128
(subset B)** into an inline SVG of bars — fully self-contained (no package),
printable ASCII only (other characters are stripped).

---

## 7. Saving & validation

CRUD goes through Spatie **Data** (validation/DTO) → **Action** (deep validation +
persistence), matching the project's `ApprovalTemplate` pattern.

- `CreateDocumentTemplateData` / `UpdateDocumentTemplateData` validate `name`,
  `document_type` (in config keys), `facility_ids` (each must belong to the
  current company), `is_default`, `page_setup`, `layout`. `company_id` /
  `created_by` are **set server-side**, never accepted from the client.
  `document_type` is immutable after creation.
- `LayoutValidator::validate($layout, $documentType)` recursively asserts:
  - each node is a valid `section` or `block`;
  - grids are sane (rows/cols ≥ 1, cells in bounds, spans don't overflow, no
    overlap);
  - each block exists in the registry **and is allowed for the document type**;
  - every field key in a block's config exists in `availableFields()`;
  - config conforms to `configSchema` **plus** the shared presentational keys
    (colors, `text_spacing`, `label_width_balance`, `*_align`), each typed; any
    other key is rejected;
  - section `style` (`spacing`, `background_color`, `vertical_dividers[]`,
    `horizontal_dividers[]`), block `style` (`border_*`), and a cell's
    `manual_page_break_before` are validated strictly (unknown keys / bad hex
    colors / out-of-grid divider positions are rejected).

Presentational rendering: a block-box wrapper in `DocumentRenderer` applies border
/ background / text color / line-height / alignment (radius + padding when a border
or background is set); `line_items` and `summary_totals` paint their own surface,
so the wrapper skips their background. Colors accept `#rgb`/`#rrggbb` or `""`.

Errors are field-pathed, e.g.
`layout.cells[0].child.config.fields[0]: unknown field "customer.foo" for block "customer_details"`.

### Layout JSON shape

```jsonc
{
  "type": "section",
  "rows": 4, "cols": 1, "flow": false,
  "cells": [
    { "row": 1, "col": 1, "row_span": 1, "col_span": 1, "valign": "top",
      "child": {
        "type": "section", "rows": 1, "cols": 2,
        "cells": [
          { "row": 1, "col": 1, "child": { "type": "block", "block": "company_header", "config": { "show_logo": true } } },
          { "row": 1, "col": 2, "child": { "type": "block", "block": "document_details", "config": { "heading_text": "Invoice" } } }
        ]
      }
    }
    // … parties section, line_items (flow) section, totals/signature section
  ]
}
```

---

## 8. Rendering pipeline

`DocumentRenderer`:

1. `payload(template, model, preview)` → mapper `sample()` for preview, else run
   the model (eager-loaded per config `with`) through its Resource and `map()`.
2. Recursively walk `layout` → CSS-grid `<div>`s for sections, each block's
   Blade partial for leaves.
3. Wrap in `resources/views/documents/template.blade.php` (an `@page` size +
   print stylesheet; the `line_items` table repeats its header across pages).
4. `pdf()` pipes the HTML to **Browsershot** (headless Chrome) using
   `page_setup` margins/paper/orientation.

`html()` and `pdf()` share one code path, so **preview and PDF cannot drift** —
`?preview=1` renders sample data through the same renderer.

A real render is driven by `{ resource_id }` (the id of the underlying
invoice/receipt/lpo/credit note; the legacy `model_id` key is still accepted as
a fallback). `DocumentTemplateRenderController::resolveModel()` loads it via the
type's `model` + `with` config, and enforces that the template's company and
facility apply. A foreign/missing id returns `422` keyed on `resource_id` with a
type-specific message from the type's `label` (e.g. `Receipt not found for this
company.`).

> **Infra:** Browsershot needs Node + Puppeteer/Chrome on the host. Configure via
> `DOCUMENT_TEMPLATES_NODE_BINARY`, `DOCUMENT_TEMPLATES_NPM_BINARY`,
> `DOCUMENT_TEMPLATES_CHROME_PATH`, `DOCUMENT_TEMPLATES_PDF_TIMEOUT`.

---

## 9. API

All routes sit under `/api/v1/app/{company}/property-management/…`, company-scoped
and authorized by `DocumentTemplatePolicy` (`*-document-template` permissions).

| Method & path | Action |
| --- | --- |
| `GET  settings/documents/templates` | list (filters: `document_type`, `facility_id`, `is_active`, `is_default`, `name`) |
| `POST settings/documents/templates` | create |
| `GET  settings/documents/templates/{documentTemplate}` | show |
| `PATCH settings/documents/templates/{documentTemplate}` | update |
| `DELETE settings/documents/templates/{documentTemplate}` | soft delete |
| `POST settings/documents/templates/{documentTemplate}/set-default` | set default (unsets sibling) |
| `POST settings/documents/templates/{documentTemplate}/duplicate` | clone |
| `GET  settings/documents/templates/registry?document_type=…` | palette + config schemas + default page setup |
| `GET  settings/documents/templates/{type}/payload-schema` | canonical schema + sample |
| `POST document-templates/{documentTemplate}/render` | render PDF (`{ resource_id }`, or legacy `model_id`), `?preview=1`, `?format=html` |

Single-item responses use the `DataResource` envelope (`{ message?, document_template }`);
lists are paginated (`data`/`links`/`meta`).

### Portal printing (tenant & vendor)

Tenant and vendor portal users can list + print the documents they own. These
routes have **no `{company}` segment**, so the `CompanyScoped` global scope adds
no filter — the company + facility are derived from the *source document* instead,
and access is enforced by each portal's ownership query (**not** the permission-only
`view` policy, which is only a coarse gate).

| Method & path | Action |
| --- | --- |
| `GET  api/v1/tenant/document-templates?document_type=&resource_id=` | list templates usable for a tenant-owned invoice/receipt/credit-note |
| `POST api/v1/tenant/document-templates/{documentTemplate}/render` | render `{ resource_id }` (or legacy `model_id`), `?format=html` |
| `GET  api/v1/vendor/document-templates?document_type=&resource_id=` | list templates usable for a vendor-owned LPO |
| `POST api/v1/vendor/document-templates/{documentTemplate}/render` | render `{ resource_id }`, `?format=html` |

- **Per-portal type allow-list** — tenant: `facility_invoice`, `facility_receipt`,
  `facility_credit_note`; vendor: `facility_lpo`. Rendering a template of any other
  type returns `403`.
- **Ownership** — the `resource_id` is resolved through the portal's ownership
  scope (tenant: `lease.user_id`; receipts also via `paying_user_id`; vendor:
  `vendor_id`). A foreign/unknown id returns `404`. The template must also belong to
  the document's company and apply to its facility, else `403`.
- No `?preview=1` here — portals always render a real, owned document. Shared logic
  lives in `App\Http\Controllers\Api\V1\Concerns\PortalDocumentController`; each
  portal subclass only supplies its `scopes()` map.
- **Landlord** is not yet supported — its documents (remittances, payment-vouchers,
  collections, expenses) have no `document_templates.types` entry.

---

## 10. Seeding

`DocumentTemplateSeeder` (registered in `DatabaseSeeder`) seeds **one company-wide
default template per document type** (`facility_ids = NULL`, `is_default = true`)
for every company, with a nested layout exercising a flow `line_items` section
and `totals`. Permissions live in `storage/app/seeders/permissions.json`
(`view/create/update/delete-document-template`).

---

## 11. Extending

- **New block:** add a `BlockType` subclass under `Blocks/`, register it in the
  `BlockRegistry` singleton (`AppServiceProvider`), add a
  `resources/views/document-blocks/{key}.blade.php` partial. The palette + Data
  tab + validation pick it up automatically.
- **New document type:** add an entry to `config/document_templates.php`
  (`model`, `resource`, `mapper`, `with`, `facility_path`, `party`), a
  `PayloadMapper`, and (optionally) seed a default. Shared blocks become
  available automatically.
- **Pagination beyond "one flow section":** out of scope for v1 — the `isFlow()`
  hook on blocks is the extension point.

---

## 12. Tests

`tests/Feature/PropertyManagementDocumentTemplateTest.php` covers: seeded
defaults; registry blocks per type; payload schema; CRUD; multiple templates per
type; bad-layout rejection (unknown block/field, out-of-bounds, overlap);
block-not-allowed; label override resolution; single-default guard;
facility applicability; preview HTML; and PDF generation (auto-skipped when
headless Chrome is unavailable).

> Tests run in console, where `CompanyScoped`'s global scope and auto
> `company_id` are disabled — tests assert scoping at the query/scope level and
> set `company_id` explicitly.
