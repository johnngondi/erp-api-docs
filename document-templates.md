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
   layout JSON ──► LayoutValidator ──► clamped + persisted (document_templates.layout)
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
| Payload mappers | `app/Services/DocumentTemplates/Payload/` (shared logic in `Payload/Concerns/`) |
| Layout validation | `app/Services/DocumentTemplates/LayoutValidator.php` |
| Rendering | `app/Services/DocumentTemplates/DocumentRenderer.php` |
| QR generation | `app/Services/DocumentTemplates/QrCodeGenerator.php` |
| Blade partials | `resources/views/document-blocks/`, `resources/views/documents/template.blade.php` |
| Data (DTO/validation) | `app/Data/PropertyManagement/DocumentTemplate/` |
| Actions | `app/Actions/PropertyManagement/DocumentTemplate/` |
| Controllers | `…/Settings/Document/DocumentTemplateController.php`, `…/PropertyManagement/DocumentTemplateRenderController.php` |
| Policy | `app/Policies/DocumentTemplatePolicy.php` |
| Seeder | `database/seeders/DocumentTemplateSeeder.php` (per-company body in `SeedDefaultDocumentTemplatesAction`) |
| Layout migration | `app/Console/Commands/MigrateDocumentTemplateLayoutsCommand.php` |
| Facility bank accounts (fuels `payment_details`) | `app/Models/FacilityBankAccount.php`, see `docs/frontend/api/v1/app/property-management/facilities/bank-accounts.md` |

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
| `schema_version` | uint | **2** as of this rollout — bumped on every template touched by `document-templates:migrate-layouts` or saved after the v2 registry shipped. `config('document_templates.schema_version')` is the current value new templates get. |
| timestamps + soft deletes | | |

Index: `(company_id, document_type, is_active)`.

`facility_bank_accounts` (Phase 0, new): the facility-level projection of which
bank accounts collect rent/charges — see
`docs/frontend/api/v1/app/property-management/facilities/bank-accounts.md`. The
`payment_details` block (invoice) reads this table via `Facility::bankAccounts()`.

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
explicit list of 35 block instances.

### Field (`Fields/Field.php`)

A toggleable field a block exposes: `key` (dot-path into the canonical payload,
e.g. `tenant.tax_pin`), `default_label` (passed through `__()`), `data_path`,
`type` (`string|money|date|datetime|image|number|array`), optional `format`.

### BlockType (`Blocks/BlockType.php`)

Each block declares:

- `key()`, `label()`, `description()`, `icon()`
- `presentation()` — how the frontend should draw it: `fields|table|image|qr|text|divider|sign_off`.
  Orthogonal to `isFlow()`: several table blocks (`tax_summary`, `payments`,
  `invoice_ageing`, …) are tabular but must not paginate.
- `documentTypes()` — which types it applies to (shared blocks use the
  `SharedAcrossDocuments` trait → all configured types)
- `availableFields()` — `Field[]` the user can toggle
- `configSchema()` — declarative options the frontend renders a form from and
  `LayoutValidator` checks against. Table blocks build theirs via the shared
  `tableSchema()` helper (`min_rows`, `min_row_height`, `show_row_numbers`,
  `zebra`, `background_color`, `label_color`, merged over `appearanceSchema()`).
- `view()` — Blade partial (`document-blocks.{key}`)
- `isFlow()` — the `*_items` blocks (`invoice_items`, `credit_note_items`,
  `lpo_items`, `voucher_items`) plus the two remittance tables
  (`remittance_collections`, `remittance_expenses`) return true (the
  paginating sections)
- `resolveLabel($fieldKey, $config)` — 3-tier label resolution (see §5)

### Blocks shipped (35)

**Shared / layout — all six types**

| Block | Presentation | Notes |
| --- | --- | --- |
| `logo` | image | `company.logo_url`; `max_height_mm`, `min_height_mm` (own image sizing, distinct from the shared `style.min_height` block-box mechanism — see §4/§7), `align` |
| `company_header` | fields | `company.name`, `.tax_pin`, `.address`, `.phone`, `.email`, `.website`, `.tagline`. No `show_logo` toggle — the logo is its own block. |
| `document_title` | text | The document's own name in large type. Defaults to the mapper's `document.title` (TAX INVOICE / RECEIPT / CREDIT NOTE / LOCAL PURCHASE ORDER / PAYMENT VOUCHER / REMITTANCE ADVICE); `text` overrides it, `uppercase` (default true), `font_size` (8–48), `align`. |
| `free_text` | text | `heading_text` + static `content` |
| `divider_line` | divider | `orientation`, `line_color`, `line_width`, `margin_top`/`margin_bottom` (a visual nudge via `transform`, not a flow margin) |

**Party blocks**

| Block | Types | Fields |
| --- | --- | --- |
| `landlord_details` | invoice, receipt, credit note, lpo, remittance | `landlord.name`, `.tax_pin`, `.phone`, `.email`, `property.name`, `.address`, `.lr_number`. Deliberately **not** `SharedAcrossDocuments`: a payment voucher has no landlord of its own, so the block would render only empty rows there. |
| `payee_details` | payment voucher | `payee.name`, `.tax_pin`, `.address` — who the voucher pays (a user, per the voucher's `payable_as`) |
| `tenant_details` | invoice, receipt, credit note | `tenant.name`, `.lease_id`, `.tax_pin`, `.phone`, `.email`, `.unit`. `unit` is the distinct `facility_spaces.name` of the lease's items (see §5.1), **not** the property name |
| `vendor_details` | lpo | `vendor.name`, `.tax_pin`, `.phone`, `.email`, `.address` |

**Per-type detail blocks**

| Block | Type | Fields |
| --- | --- | --- |
| `invoice_details` | invoice | `document.number` (`INV0001`), `.issued_at`, `.due_at`, `.status`, `.currency` |
| `credit_note_details` | credit note | `document.number` (`CN0001`), `.issued_at`, `.status`, `.currency`, `.invoice_number` |
| `receipt_details` | receipt | `document.number` (`RCP0001`), `.issued_at`, `.served_by`, `.status`, `.currency` |
| `lpo_details` | lpo | `document.number` (`LPO0001`), `.issued_at`, `.due_at` (delivery), `.status`, `.currency` |
| `lpo_title` | lpo | `lpo.title` (the order's title: a direct LPO's as entered, a workflow LPO's copied from its request), `document.notes` |
| `voucher_details` | payment voucher | `document.number` (`PV0001`), `.credit_account`, `.method`, `.reference`, `.paid_at`, `.status` |
| `remittance_details` | remittance | `document.number` (`REM0001`), `landlord.name`, `property.name`, `document.period`, `.memo` (e.g. `August 2026`), `.issued_at`, `.status` |

**Content blocks**

| Block | Types | Presentation | Notes |
| --- | --- | --- | --- |
| `fiscal_qr` | invoice, credit note | qr | QR of `document.verify_url`, then `document.cu_invoice_number`/`.cu_serial_number` stacked beneath (toggle via `show_cu_invoice_number`/`show_cu_serial_number`) |
| `document_notes` | invoice, credit note, lpo | text | Binds `document.notes` |
| `invoice_items` | invoice | table, **flow** | `items.notes`, `.quantity`, `.amount`, `.tax`, `.total`. Meter photos are no longer drawn in the table; see `meter_reading_images` |
| `meter_reading_images` | invoice | image, **flow** | Reads `items[]` directly (no field list). One group per `is_utility_bill` item with at least one `image_url`, titled with the item description and `· Meter {number} ({name})`. Beneath the title, "Previous reading" (left) and "Current reading" (right) photos sit side by side, each with its reading value and date, followed by "Consumption". A side with no photo shows a "No photo captured" box. `image_url` is only set when the upload's `source_url` file exists on the storage disk. When no utility item has a stored photo, the block renders nothing. Because `BlockType::hidesWhenEmpty()` is true, the renderer also skips the block box: border, background, padding and `min_height`. Config: `heading_text` (default "Meter readings"; `""` hides it), `font_size`, `image_height_mm` (20–150, default 70), `show_reading_values`, `show_consumption`, `show_meter_details` (all default true). Placement is up to the designer, e.g. right after `invoice_items` or at the end of the document |
| `credit_note_items` | credit note | table, **flow** | `items.notes`, `.quantity`, `.amount`, `.tax`, `.total` |
| `lpo_items` | lpo | table, **flow** | `items.title`, `.notes`, `.quantity`, `.amount`, `.tax`, `.total` |
| `totals` | invoice, credit note, lpo | fields | `totals.amount`, `.tax`, `.total`, `.paid`, `.balance` (no receipt — receipts have no totals block) |
| `tax_summary` | invoice, credit note, lpo | table | `tax_summary.name`, `.rate`, `.taxable`, `.tax` |
| `payments` | invoice | table | `payments.type`, `.number`, `.date`, `.method`, `.reference`, `.amount` — receipt + credit-note allocations against this invoice |
| `invoice_ageing` | invoice | table | `ageing.label`, `.amount` — Current/1-30/31-60/61-90/90+ buckets |
| `payment_details` | invoice | table | The facility's **collection** `facility_bank_accounts`, one row per lease component: `bank_accounts.component`, `.bank_name`, `.branch`, `.account_name`, `.account_number`, `.paybill_number` |
| `mobile_money` | invoice | table | M-Pesa paybill rows: `mobile_money.paybill_number`, `.account_number`, `.bank_name`, `.component`. Agent-held collection account + company `paybill_number` setting → company paybill against the invoice number; otherwise each pay-to account's `banks.paybill_number` against its account number. `hidesWhenEmpty` |
| `bank_account_details` | receipt | fields | Where the receipt's money landed: `payment_account.bank_name`, `.branch`, `.account_name`, `.account_number` |
| `payment_transactions` | receipt | table | `transactions.transaction_date`, `.transaction_number`, `.method`, `.amount` |
| `invoice_allocations` | credit note, receipt | table | Invoices this document cleared: `allocations.invoice_number`, `.invoice_date`, `.invoice_total`, `.allocated`, `.balance` |
| `component_allocations` | receipt | table | What the receipt settled per charge: `component_allocations.component`, `.amount`, with the grand total in `totals.allocated`. VAT is the mapper's last row, not a column. See §4.5 |
| `signatories` | **lpo only** | fields | See §4.1 — replaces the generic signature block on LPOs specifically |
| `sign_off` | payment voucher, remittance | sign_off | Rows of labelled fill-in slots to sign by hand. See §4.3 |
| `voucher_items` | payment voucher | table, **flow** | What the voucher pays, with data-driven withholding columns and a total-paid footer. See §4.2 |
| `remittance_summary` | remittance | fields | `summary.is_advance`, `.total_collections`, `.total_expenses`, `.advance_remittances`, `.total_withheld`, `.total_remitted`. See §4.4 for where each comes from |
| `remittance_collections` | remittance | table, **flow** | `collections.receipt_number`, `.transaction_date`, `.credit_account`, `.method`, `.reference`, `.tenant`, `.amount` |
| `remittance_expenses` | remittance | table, **flow** | `expenses.bill_number`, `.transaction_date`, `.reference`, `.supplier`, `.service`, `.amount` |

### Serialized form (`BlockRegistry::toArray($documentType)`)

Each block →
`{ key, label, description, icon, presentation, is_flow, document_types, allowed, available_fields[], config_schema }`.
This is exactly what the designer palette + Data tab consume.

---

## 4. Canonical payload & mappers

The frontend Data tab and all blocks bind against a **canonical payload shape**,
not raw resource keys:

```
company:        { name, tax_pin, address, postal_address, phone, email, website, tagline,
                  twitter, facebook, logo_url, services[], brand_color, text_color_on_brand_bg,
                  accent_color, text_color_on_accent_bg, timezone }
                # services, the palette and timezone are print-only (see §8.1) and are not
                # in the bindable field catalog; the rest are
landlord:       { name, tax_pin, phone, email }
property:       { name, address, lr_number, city, country }
tenant:         { name, lease_id, tax_pin, phone, email, unit }        # invoice/cn/receipt
vendor:         { name, tax_pin, phone, email, address }               # lpo
payee:          { name, tax_pin, address, phone, email }               # payment voucher
document:       { title, number, raw_number, issued_at, due_at, status, notes, currency,
                   invoice_number, cu_invoice_number, cu_serial_number, verify_url, served_by }
items[]:        { description, notes, quantity, unit_price, amount, tax, tax_rate, total,
                   is_utility_bill, previous_reading: {value, read_at, image_url},
                   current_reading: {value, read_at, image_url},
                   meter: {name, number}, consumption }
totals:         { amount, tax, total, paid, balance }
tax_summary[]:  { name, rate, taxable, tax }
payments[]:     { type, number, date, method, reference, amount }      # invoice
allocations[]:  { invoice_number, invoice_date, invoice_total, allocated, balance }  # cn/receipt
ageing:         { as_at, buckets: [ { label, amount } ] }              # invoice
bank_accounts[]:{ component, bank_name, branch, account_name, account_number, paybill_number }  # invoice
mobile_money[]: { component, bank_name, paybill_number, account_number }  # invoice —
                # agent holds the collection account (facility managementContract
                # .collection_account_holder = agent) AND the company `paybill_number`
                # setting is set → one row: company paybill, account_number = invoice
                # number (INV0481), component "All charges", bank_name null.
                # Otherwise → one row per bank_accounts[] row whose bank has a
                # paybill_number, account_number = that bank account's number.
                # Empty array when no paybill is known (the block is then hidden)
payment_account:{ bank_name, branch, account_name, account_number }    # receipt
transactions[]: { transaction_date, transaction_number, method, amount }  # receipt
lpo:            { title }                                              # lpo
approval_chain[]: { name, date }                                       # lpo, see §4.1
signoff:        { prepared_by, prepared_at }                           # voucher, remittance
withholding_columns[]: [ "WHT Professional", … ]                       # voucher, see §4.2
summary:        { is_advance, total_collections, total_expenses,
                   advance_remittances, total_withheld, total_remitted }  # remittance, see §4.4
collections[]:  { receipt_number, transaction_date, credit_account, method,
                   reference, tenant, amount }                          # remittance
expenses[]:     { bill_number, transaction_date, reference, supplier, service, amount }  # remittance
```

`document.title` is a constant per document type, supplied by the mapper and
rendered by the `document_title` master block.

`document.number` is the **D1** formatted number — `{PREFIX}` + the row id
zero-padded to 4 digits (`INV0001`, `CN0112`, `RCP0481`, `LPO0219`, `PV0193`,
`REM0212`; the remittance advice also formats bill ids as `BILL0774`), computed at
render time in the mapper (`FormatsDocumentNumbers::prefixed()`). There is no
stored sequence column; ids ≥ 10000 are not truncated, just not zero-padded
beyond their natural width.

A per-type `PayloadMapper` (`Payload/*PayloadMapper.php`) re-shapes the existing
Resource/model output into this shape. **Mappers re-shape only — they never
recompute money/field logic** (money/totals are taken from the model's own
columns; identity fields are read from the eager-loaded relations declared in
config `with`). Shared logic lives in `Payload/Concerns/`:

- `ResolvesCompanyPayload` — the one place that reads the real `companies`
  columns (`company_phone`, `company_email`, `company_website`,
  `company_tagline`, `tax_pin`, `profile_photo_url`) and resolves an absolute
  logo URL, falling back to `/images/logo-placeholder.svg` rather than an
  external network fetch (which would stall Browsershot).
- `ResolvesLandlordPayload` — `landlord`/`property` blocks, from
  `facility.landlord` (a `User`) and the `Facility` itself.
- `FormatsDocumentNumbers` — the D1 `prefixed()` helper.
- `BuildsTaxSummary` — groups an item collection by `taxType`, summing
  `amount`→`taxable` and `tax`→`tax`.
- `BuildsAgeing` — no AR-ageing engine exists elsewhere in the codebase; this
  places a single invoice's own outstanding balance into whichever
  Current/1-30/31-60/61-90/90+ bucket matches how many days past `due_at`
  today is (all five buckets always returned, zeroed where they don't apply).

Each mapper exposes:

- `map(Model, array $resource): array` — real payload
- `static schema(): array` — `dot-path => type`, drives the payload-schema endpoint
- `static sample(?Company $company = null): array` — representative fake
  payload for preview; when a company is given, its real `companyPayload()`
  block is merged over the placeholder one so the designer preview shows the
  user's own letterhead from the first render.

### 4.1 LPO signatories

`signatories` is LPO-only and replaces the generic `signature` block there
specifically. The template designer sets `config.signatory_count` (1-8,
default 3); the block computes **Created / Reviewed / Approved** from the raw,
ordered `approval_chain[]` the mapper builds — index 0 is always the
procurement request's creator (`procurementRequest.owner`), followed by each
approval step in chain order (`procurementRequest.steps`, ordered by `id` —
there is no `step_order` column on this table, and it is **not** the generic
`Approvable`/`ApprovalStep` framework `FacilityInvoice`/`Loo`/the LPO's own
approval use). Selection is intentionally computed at **render time** in
`signatories.blade.php`, not in the mapper, because `signatory_count` is a
per-block-instance config, not a document-level property:

- N=1 → `[Created]`
- N=2 → `[Created, Approved(last step)]`
- N=3 → `[Created, Reviewed(2nd-to-last step), Approved(last step)]`
- N>3 → extra `Approved` slots are appended **after** the fixed trio, pulled
  from the **front** of the step chain forward, never re-using the last two
  steps already shown
- Never pads with blanks — a short chain renders fewer than N signatories

`CreateLpoAction` only ever runs once a procurement request's last step
approves, so by the time an LPO document exists the chain is always complete;
no status filtering is needed when building `approval_chain`.

---

### 4.2 Voucher items & withholding columns

A payment voucher pays a mix of supplier bills (`FacilityBill`) and landlord
remittances (`FacilityRemittance`) through one polymorphic item table, so
`PaymentVoucherPayloadMapper` builds each row from whichever model it points at:

| Column | Bill | Remittance |
| --- | --- | --- |
| `items.transaction_date` | `item.transaction_at` ?: `bill.invoice_date` ?: `bill.created_at` | `item.transaction_at` ?: `remittance.created_at` |
| `items.property` | `bill.facility.name` | `remittance.facility.name` |
| `items.memo` | `item.notes` ?: `bill.notes` | `item.notes` ?: `remittance.notes` |
| `items.total` | `bill.total` | `remittance.remittable_amount` |
| `items.payable` | `bill->payableAmount()` | `remittance.remittable_amount` |
| `items.paid` | `item.amount` | `item.amount` |

The item relation is loaded with a nested `morphWith` inside `map()` rather than
through config `with`: config is cached in production and cannot hold a closure.

**Withholdings are data-driven columns.** Which taxes a bill carries varies by
company and by bill, so they cannot be declared as `Field`s. The mapper
publishes `withholding_columns` — the distinct withholding tax names across the
whole voucher, in first-seen order — and each item carries a `withholdings` map
keyed by the same names. The `voucher_items` Blade turns each into a column,
inserted directly after `items.total` (or appended last if that column is
deselected), printing `—` where a tax does not apply so remittance rows stay
aligned. `config.withholding_columns` switches this:

- `per_tax` (default) — one column per tax
- `total` — a single combined column, labelled by `withholding_total_label`
- `none` — no withholding column at all

Withholdings are filtered on **`amount > 0`, not `balance > 0`**: one that has
since been paid or remitted still belongs on the voucher, which is a record of
what was withheld from that bill at the time. `items.payable` is
`FacilityBill::payableAmount()`, which is balance-based and so only nets off
withholdings that are *still outstanding*. On a bill whose withholding has
already been remitted the two therefore do not reconcile, by design: the
withholding column says what was withheld, `payable` says what is owed today.

The footer row is the voucher's own `totals.paid` (`vouchers.amount`), not a
re-sum of the rows — toggled by `show_total_row`, labelled by `total_label`.

### 4.3 Sign-off rows

`sign_off` is one block used several times per document (the voucher's
"Approval details" and "Recipient details"), because the only difference
between them is the heading and the row labels. `config.rows` is a list of
`{ cells: [{ label, source }] }`; a cell with a `source` prints that payload
value, a cell without one prints a ruled line to write on that runs to the
edge of its cell.

```json
{"type": "block", "block": "sign_off", "config": {
    "heading_text": "Approval details",
    "rows": [
        {"cells": [
            {"label": "Prepared By", "source": "signoff.prepared_by"},
            {"label": "Sign",        "source": null},
            {"label": "Date",        "source": "signoff.prepared_at"}
        ]},
        {"cells": [
            {"label": "Checked By", "source": null},
            {"label": "Sign",       "source": null},
            {"label": "Date",       "source": null}
        ]}
    ]
}}
```

`source` may only be one of the block's own `availableFields()` keys
(`signoff.prepared_by`, `signoff.prepared_at`) or `null` — the `LayoutValidator`
enforces that along with at most 12 rows, at most 4 cells per row, and a
non-blank label of at most 60 characters on every cell.

### 4.4 Remittance summary figures

Three of the six summary values are stored columns on `facility_remittances`;
two are not stored anywhere and are computed by the mapper at render time.

| Field | Source |
| --- | --- |
| `summary.is_advance` | `is_advance`, as Yes/No |
| `summary.total_collections` | **`total_income`** — note the column name |
| `summary.total_expenses` | `total_expenses` (already includes the management-fee expense, which `CreateRemittanceAction` posts as a bill and attaches — so it is also a row in the Expenses table) |
| `summary.total_remitted` | `remittable_amount` |
| `summary.advance_remittances` | **computed** — advance remittances for this facility whose period sits inside this remittance's period, excluding pending ones and this row itself. Mirrors the query `ComputeRemittableAmountService` ran at creation time, so the printed figure matches what was actually deducted. Zero on an advance remittance itself. |
| `summary.total_withheld` | **computed** — the sum of this remittance's own receipts taken through a payment method with `payment_methods.is_withholding` (withholding VAT, withholding rental income): collected, but paid to KRA rather than into the account. Scoped to the remittance's own receipts so it reconciles against the Collections table printed alongside it. |

### 4.5 Receipt component allocations

`invoice_allocations` says which invoices a receipt cleared; `component_allocations`
says what the money was *for*. `ReceiptPayloadMapper::componentAllocationRows()`
walks `receiptAllocations → componentAllocations → leaseItemComponent →
leaseComponent`, and:

- skips allocations whose status is `ReceiptAllocationStatus::Cancelled` — that
  money is no longer applied to an invoice, so it names neither a unit nor a
  component;
- groups by lease-component name, summing the row `amount` (net). A component
  with no resolvable name falls back to "Unallocated";
- appends **VAT** as a final component row, the summed `tax`, only when it is
  above zero — VAT is data, not a column, so numbering, zebra striping and
  borders treat it like any other row;
- publishes the grand total as `totals.allocated`, summed from the stored `total`
  column (not `amount + tax`) so the printed figure reconciles with the rows.

The block draws Component on the left and Amount on the right, with the total as
a `<tfoot>` row. `show_total_row` (default true) and `total_label` (default
"Total") control that footer; `show_row_numbers` defaults to **false** here,
against the shared table default, because a two-column table reads better
without a `#` gutter.

The footer itself is `partials/table.blade.php`'s optional `$footerRows` —
`[['label' =>, 'value' =>, 'grand' => bool]]`, rows with a null value dropped,
`grand` rendering bold. Every other table block omits it and is unchanged.

---

---

## 5. Labels — three-tier resolution

In priority order (implemented in `BlockType::resolveLabel()` and used by every
partial):

1. **Per-template override** — `config.field_labels[fieldKey]` in the layout JSON.
2. **Registry default** — the field's `default_label`, run through `__()`.
3. *(extension point)* localization via translation keys.

Blocks also honour `show_labels`, `label_position` (`left`/`top`), `heading_text`,
`font_size`, `align`, `show_empty_fields` from their `configSchema`.

### Empty fields — "Not Available"

`partials/field-list.blade.php` used to drop any row whose value was null or
`''`, so a field the designer had deliberately placed simply vanished when the
data was missing. It now keeps the row and prints a faded
`<span class="dt-empty">Not Available</span>` — but only when the template picked
its fields, i.e. `config.fields` is an explicit list. A block left on "all
defaults" (no `config.fields`) still drops empties, so the seeded templates did
not sprout `Lease: Not Available` rows the day this shipped.

`config.show_empty_fields` (boolean, `appearanceSchema`) overrides that rule in
either direction; leaving it unset is the auto behaviour above. A value counts
as empty when it is `null`, `''` or `[]`.

Out of scope on purpose: `mobile_money` keeps its own inline loop and its own
`@continue` on empty values — it repeats per paybill and `hidesWhenEmpty()`, so
placeholders there would stop the block from ever hiding. Table blocks keep
blank cells.

### 5.1 Unit names

`tenant.unit` is resolved by `Payload\Concerns\ResolvesUnitNames`: the distinct,
comma-joined `facility_spaces.name` behind a lease's `leaseItems`
(`facility_space_id` is nullable, so every hop is null-safe and blanks are
dropped). It returns `null` — not a dash — when nothing resolves, so the empty
value reaches the field list and renders as the placeholder above.

- **Invoice / credit note** — the document's own `lease`.
- **Receipt** — the leases behind its allocated invoices, minus the cancelled
  allocations, distinct across all of them.

The eager loads this needs live in `config/document_templates.php` under each
type's `with` (`lease.leaseItems.facilitySpace`, and on the receipt
`receiptAllocations.invoice.lease.leaseItems.facilitySpace`).

---

## 6. QR generation

`fiscal_qr` (invoice/credit-note) is fixed to `document.verify_url` — no
`source`/`field`/`value` config (unlike the old generic `qr_code` it replaced).
Config: `size_mm`, `align`, `caption`, `show_cu_invoice_number`,
`show_cu_serial_number`.

`QrCodeGenerator::render($value, $px)` produces an **inline SVG** via
`bacon/bacon-qr-code` (offline, embeds in the PDF). If the library is missing it
falls back to a network QR `<img>` (prefer the library for offline PDFs).

> No barcode block ships in v2 (`BarcodeBlock` was removed along with the
> generic `signature`/`qr_code`/`document_details`/`line_items`/
> `customer_details`/`summary_totals`/receipt-only `payment_details` blocks —
> none of the four document specs called for one). `BarcodeGenerator` remains
> in the codebase if a future block needs it.

---

## 7. Saving & validation

CRUD goes through Spatie **Data** (validation/DTO) → **Action** (deep validation +
persistence), matching the project's `ApprovalTemplate` pattern.

- `CreateDocumentTemplateData` / `UpdateDocumentTemplateData` validate `name`,
  `document_type` (in config keys), `facility_ids` (each must belong to the
  current company), `is_default`, `page_setup`, `layout`. `company_id` /
  `created_by` are **set server-side**, never accepted from the client.
  `document_type` is immutable after creation.
- `LayoutValidator::validate($layout, $documentType): array` recursively
  asserts, and **returns the (possibly clamped) layout** — callers must save
  the returned value, not the original input:
  - each node is a valid `section` or `block`;
  - grids are sane (rows/cols ≥ 1, cells in bounds, spans don't overflow, no
    overlap);
  - each block exists in the registry **and is allowed for the document type**;
  - every field key in a block's config exists in `availableFields()`;
  - config conforms to `configSchema` **plus** the shared presentational keys
    (colors, `text_spacing`, `label_width_balance`, `*_align`), each typed; any
    other key is rejected;
  - **numeric config values with a `min`/`max` in their spec are clamped into
    range, not rejected** — a designer-side clamp and a server rejection must
    agree;
  - section `style` (`spacing`, `background_color`, `min_height`, `margin_top`,
    `margin_bottom`, `label` (designer-only, ignored at render),
    `vertical_dividers[]` incl. `margin_left`/`margin_right`,
    `horizontal_dividers[]`), block `style` (`border_*`, `min_height`), and a
    cell's `manual_page_break_before` are validated strictly (unknown keys /
    bad hex colors / out-of-grid divider positions are rejected, not clamped).

Presentational rendering: a block-box wrapper in `DocumentRenderer` applies border
/ background / text color / line-height / alignment (radius + padding when a border
or background is set), plus `style.min_height` (mm) — forcing the wrapper `<div>`
into existence even when nothing else applies. Blocks that paint their own surface
(`rendersOwnSurface() === true`: every table-presentation block) skip the
wrapper's `background_color`. Colors accept `#rgb`/`#rrggbb` or `""`.

Errors are field-pathed, e.g.
`layout.cells[0].child.config.fields[0]: unknown field "tenant.foo" for block "tenant_details"`.

### Min-height & blank-row padding

| Node | Key | Unit |
| --- | --- | --- |
| Section | `style.min_height` | mm |
| Block (any) | `style.min_height` | mm |
| Table block | `config.min_rows` | integer row count |
| Table block | `config.min_row_height` | mm |

`resources/views/document-blocks/partials/table.blade.php` (shared by every
table-presentation block) pads with blank `<tr>`s up to `min_rows`, each
`min_row_height` mm tall, so a sparsely-populated table keeps the seeded ruled
cadence rather than collapsing.

### Layout JSON shape

```jsonc
{
  "type": "section",
  "rows": 4, "cols": 1, "flow": false,
  "style": { "min_height": 25 },
  "cells": [
    { "row": 1, "col": 1, "row_span": 1, "col_span": 1, "valign": "top",
      "child": {
        "type": "section", "rows": 1, "cols": 2,
        "cells": [
          { "row": 1, "col": 1, "child": { "type": "block", "block": "logo", "config": {} } },
          { "row": 1, "col": 2, "child": { "type": "block", "block": "company_header", "config": {} } }
        ]
      }
    }
    // … parties section, invoice_items (flow) section, tax_summary/totals section
  ]
}
```

---

## 8. Rendering pipeline

`DocumentRenderer`:

1. `payload(template, model, preview)` → mapper `sample($company)` for preview
   (company resolved from `template.company_id`), else run the model
   (eager-loaded per config `with`) through its Resource and `map()`.
2. Recursively walk `layout` → CSS-grid `<div>`s for sections, each block's
   Blade partial for leaves.
3. Wrap in `resources/views/documents/template.blade.php` (an `@page` size +
   print stylesheet; flow blocks repeat their header across pages). The shell
   also stamps the shared footer onto every page — see §8.1.
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

### 8.1 The printout footer

Every template printout carries the same footer. It is **not** a block — the
designer cannot add, move or remove it; the shell draws it on every page.

`resources/views/documents/partials/footer.blade.php`, three strips top to bottom:

| Strip | Contents | Source |
| --- | --- | --- |
| Services | The company's services, one equal-width cell each, separated by hairlines, on a coloured bar | `company.services[]` on `company.accent_color` in `company.text_color_on_accent_bg` (fall back to `config('app.company_palette')` — env `COMPANY_ACCENT_COLOR` `#ed1c24` / `COMPANY_TEXT_COLOR_ON_ACCENT_BG` `#ffffff`) |
| Contacts | Name, postal address, phone, email and website behind inline SVG icons, then `Twitter: …` and `Facebook: …` spelled out | `company.name`, `.postal_address`, `.phone`, `.email`, `.website`, `.twitter`, `.facebook` |
| Stamp | `Printed at: Wed, Sep 23, 2026 01:20 PM` on the left, `Powered by <developer>` on the right | `now()` in `company.timezone`; `config('app.developer_name')` (`APP_DEVELOPER_NAME`) |

Everything is per-company and everything is optional: a strip with nothing in it
is not drawn, and the services bar balances itself across however many services
there are (five services → five 20% cells; two → two halves).

In **preview** (`?preview=1`) the footer follows the same rule as every other
company field — anything the real company leaves unset falls back to the mapper's
placeholder company, so a company with no services still shows a sample strip.

**Pinned to the foot of every page.** The shell stamps the footer twice, and the
two copies do two different jobs:

1. A hidden copy (`visibility: hidden`) inside the shell table's `<tfoot>`.
   Chrome repeats a `table-footer-group` on every printed page, so this reserves
   the footer's exact height at the foot of each one and keeps the body from
   flowing underneath it. It is hidden rather than sized by hand so the
   reservation can never drift from what the footer actually measures — add a
   service, wrap the contacts onto a second line, and the reserve follows.
2. A `position: fixed; bottom: 0` copy outside the table, which is what paints.
   Chrome repeats a fixed-position element on every printed page, so the footer
   lands flush with the bottom of the page each time — including the last one,
   where the body usually stops well short of it.

The partial's `<style>` is wrapped in `@once`, so including it twice emits one
stylesheet. A caller that does not need the footer pinned can include it once on
its own; it then simply renders in flow.

**Reuse.** The partial carries its own `dtf-` class prefix and its own `<style>`,
so any other print view can pick it up with a single include and no stylesheet
wiring:

```blade
@include('documents.partials.footer', ['company' => $companyArray])
```

It takes `$company` (the canonical `company` payload block, every key optional)
and an optional `$printedAt` override.

**Seeding the details.** `CompanyFooterDetailsSeeder` is a disposable one-off that
fills the demo company's footer fields (services, postal address, socials, accent
colour), only where they are still blank:

```bash
php artisan db:seed --class=CompanyFooterDetailsSeeder
```

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

Facility bank accounts (`payment_details`'s data source) are managed under
`facilities/{facility}/bank-accounts` — see
`docs/frontend/api/v1/app/property-management/facilities/bank-accounts.md`.

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

`DocumentTemplateSeeder` (registered in `DatabaseSeeder`, and now also invoked
from `Company::booted()`'s `created` hook — every new company gets its six
default templates immediately, not just via a manual seeder run) delegates the
per-company body to `SeedDefaultDocumentTemplatesAction`, which builds **six
bespoke per-type layouts** (not one generic shape shared across types — see
`SeedDefaultDocumentTemplatesAction::invoiceLayout()`/`creditNoteLayout()`/
`receiptLayout()`/`lpoLayout()`/`paymentVoucherLayout()`/`remittanceLayout()`).
`facility_ids = NULL`, `is_default = true`.
Idempotent via `firstOrNew` on `(company_id, document_type, name)`.

**Re-running is non-destructive.** `layout` and `page_setup` are only written
when the row is new; flags (`is_default`, `is_active`, `schema_version`) are
written every time. That is what makes it safe to re-run the seeder across
existing companies to hand them a newly shipped document type — a customized
"— Standard" layout is left exactly as it is. After shipping a new type:

```
php artisan db:seed --class=DocumentTemplateSeeder
```

Existing companies keep their saved layouts, so they do not pick up blocks
added to the stock layouts (e.g. `document_title`); those are one drag away in
the designer palette.

The exception is `meter_reading_images`. The stock invoice layout places it in its own section right after `invoice_items`. It replaced the old `invoice_items.show_utility_readings` in-table photos, so migration `2026_09_22_000004_move_invoice_meter_images_to_own_block` upgrades every saved invoice layout through `App\Services\DocumentTemplates\Upgrades\MoveMeterImagesToOwnBlock`:
- It strips `show_utility_readings`, which the validator now rejects as unknown.
- It inserts the block in a new row directly below the `invoice_items` cell. The anchor is the cell itself, or its single-cell wrapper section. A neighbour that spans across the insertion point is stretched over the new row.
- A layout that already has the block is left alone.
- A layout that had set `show_utility_readings: false` only loses the key.

Permissions live in `storage/app/seeders/permissions.json`
(`view/create/update/delete-document-template`; facility bank accounts have
their own `*-facility-bank-account` set).

### Migrating existing saved layouts

`php artisan document-templates:migrate-layouts {--dry-run} {--company=}`
rewrites v1 block keys/fields to v2 (see the class doc-block for the full
mapping table: `customer_details→tenant_details`, `document_details→` the
per-type detail block, `line_items→` the per-type items block,
`summary_totals→totals`, `qr_code→fiscal_qr`, receipt's `payment_details→
bank_account_details`, `signature`/`barcode` removed except LPO's `signature→
signatories`). Bumps `schema_version` to 2 on every touched row; never drops a
cell (an unmappable block's cell keeps its slot with `child: null`); prints a
per-template report of every rewrite/removal; `--dry-run` writes nothing.

**Run `--dry-run` against a copy of production data and review the report
before ever running this for real** — it is the one step in this feature that
can lose a user's saved template customization.

---

## 11. Extending

- **New block:** add a `BlockType` subclass under `Blocks/`, register it in the
  `BlockRegistry` singleton (`AppServiceProvider`), add a
  `resources/views/document-blocks/{key}.blade.php` partial (reuse
  `partials/heading.blade.php` + `partials/field-list.blade.php` for a `fields`
  presentation, or `partials/table.blade.php` for a `table` presentation). The
  palette + Data tab + validation pick it up automatically.
- **New document type:** add an entry to `config/document_templates.php`
  (`model`, `resource`, `mapper`, `with`, `facility_path`, `party`), a
  `PayloadMapper`, and (optionally) seed a default. Shared blocks become
  available automatically — which is the trap: every `SharedAcrossDocuments`
  block instantly applies to the new type, so check each one actually has data
  there and give it an explicit `documentTypes()` list if not (this is why
  `landlord_details` no longer uses the trait). `facility_path` may be `null`
  for a document that has no single owning facility — the render controller and
  the portals both skip the applicability check when it resolves to empty.
  The two suite-wide tests in `PropertyManagementDocumentTemplateTest` iterate
  the config, so a new type is covered the moment it is registered.
- **Pagination beyond "one flow section" per document:** out of scope — the
  `isFlow()` hook on blocks is the extension point.

---

## 12. Tests

`tests/Feature/PropertyManagementDocumentTemplateTest.php` covers: seeded
defaults; registry blocks (+ `presentation`) per type; payload schema; CRUD;
multiple templates per type; bad-layout rejection (unknown block/field,
out-of-bounds, overlap); block-not-allowed; label override resolution;
single-default guard; facility applicability; preview HTML; PDF generation
(auto-skipped when headless Chrome is unavailable); a section `style.min_height`
round-trip rendering as CSS `min-height`; a table block's `min_rows` blank-row
padding; the Phase 5 margin/nudge keys validating and clamping; every
registered block rendering without error against its type's sample payload;
and document-number formatting including ids ≥ 10000.

`tests/Feature/DocumentTemplateSignatoriesTest.php` covers the LPO
`signatories` block's Created/Reviewed/Approved selection algorithm for
signatory counts 1/2/3/5, the short-chain no-padding fallback, and an
end-to-end case building a real procurement request + approval-step chain and
confirming `LpoPayloadMapper` produces the expected `approval_chain`.

`tests/Feature/DocumentTemplateVoucherAndRemittanceTest.php` covers the payment
voucher and remittance advice: per-tax withholding columns (including one that
has been fully paid, which must still print, and a zero-amount one, which must
not); the total-paid footer and the `per_tax`/`total`/`none` column modes;
`total_withheld` counting only withholding-method receipts; `advance_remittances`
summing in-period non-pending advances and excluding the remittance itself; the
collection/expense row mapping; `sign_off` rendering and validation; the
`document_title` payload/override/case behaviour; and that re-running the
seeder adds the new templates without overwriting a customized layout.

`tests/Feature/PropertyManagementFacilityBankAccountTest.php` covers Phase 0's
CRUD, the `remittance` purpose value, and the landlord-wizard fan-out
(including per-contract `accounts[]` overrides).

> Tests run in console, where `CompanyScoped`'s global scope and auto
> `company_id` are disabled — tests assert scoping at the query/scope level and
> set `company_id` explicitly.
