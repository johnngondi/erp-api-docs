# Output VAT Analysis Report API

Domain: `Property Management > Reports > Tenants`

Base route:

`/api/v1/app/{company}/property-management/reports`

Output VAT Analysis is the **tax register of VAT charged to tenants** — the counterpart of
[Input VAT Analysis](./vat-analysis-input.md) on the supplier side. It is **read-only** and computed
on the fly.

> **Read [the shared reports contract](./README.md) first.** This page documents what is specific to
> this report. The response envelope, the bucket keys, the field keys, the per-cell `{ value, color }`
> shape and the colour vocabulary live in the README and apply here unchanged.

## Endpoints

- `GET /reports/tenants/vat-analysis-output` — requires `view-vat-analysis-output-report`
- `GET /reports/tenants/vat-analysis-output/export` — requires `export-vat-analysis-output-report`.
  Takes a required `format` (`excel` | `pdf`); any other value, or none, returns `422`.
- Listed by [`GET /reports`](./README.md#discovering-reports--get-reports).

The two permissions are **independent**, and neither is shared with
[Tenant Withholding](./tenant-withholding.md).

## Accrual basis

**A document counts when it was invoiced, never when it was paid.** Nothing in this report looks at
receipts — an unpaid invoice is declared exactly like a paid one, because the VAT became payable when
it was charged.

**`pending` and `cancelled` documents are excluded.** `pending` is the default status of a new
invoice or credit note: it has been raised but **not posted**, so declaring VAT on it would declare
tax on a document that was never issued. It appears here from the moment it moves to `unpaid`.
`cancelled` never will be — it was never charged.

**Except a cancelled credit note signed on the ETR.** KRA already holds it, so it stays in the
report as a negative line. Cancelling it raises a new invoice for the same amount, which appears as
an ordinary invoice. The two net out, as they do on KRA's side.

Every other status counts, whatever its receipt state: `unpaid`, `partially paid` and `paid` are all
declared alike. This matches [Input VAT Analysis](./vat-analysis-input.md), and matches the arrears
reports — [Ageing](./ageing-report.md) and [Debtors Listing](./debtors-listing.md) are folded from
the same invoices and leave `pending` out too, so one invoice cannot be absent from one report and
present in another.

Documents are dated and windowed on **`created_at`** — when the document was raised.

## Credit notes are negative lines, inline

**A credit note appears in the same list as the invoices**, in date order, with **negative**
`taxable_value`, `vat` and `total`. It is not in a section of its own, and it is not omitted.

The subtotal therefore *includes* the negative lines, and that is the point: the VAT payable for the
period is what was charged **less** what was credited back.

```
Invoice   380,000  VAT  60,800
Invoice   150,000  VAT  24,000
Credit    -50,000  VAT  -8,000   ← inline, negative
──────────────────────────────
Subtotal  480,000  VAT  76,800   ← 60,800 + 24,000 − 8,000
```

If your subtotal reads 84,800, the credit note is being added rather than netted off.

Do not build a separate credit-note section, and do not filter credit notes out of the register to
"clean it up" — the register is the return.

## Signed and not signed

**A document is signed when it carries a `cu_invoice_number`**, and only signed documents are
declared. The number is shown in the `cu_invoice_no` column (`null` when unsigned). There is **no
ETR status column**: it was removed in favour of the number, so a row carries no `etr_status` key.

Use the [`etr_status` filter](#filters) to list only the signed documents, or only the rest.
Credit notes carry `cu_invoice_number` too and are judged the same way.

## Filters

The [global filters](./README.md#global-filters) plus one:

| Param | Type | Meaning when omitted |
|---|---|---|
| [global filters](./README.md#global-filters) — `landlord_ids`, `facility_ids`, `facility_type_id`, `properties_status`, period | see the shared contract | all / current month |
| `buckets` | list of this report's bucket ids | Every bucket — see [Selecting buckets](./README.md#selecting-buckets--buckets) |
| `tax_id` | `taxes.id` | All taxes |
| `etr_status` | `signed` \| `not signed` | Signed and not signed |
| `facility_block_ids` / `facility_wing_ids` / `facility_floor_ids` | lists of `facility_blocks.id` / `facility_wings.id` / `facility_floors.id` | The whole property — only with exactly one property selected; see [below](#blocks-wings-and-floors) |

`tax_id` narrows to **documents carrying at least one line on that tax**; the row still shows the
whole document, not just the matching line. Same behaviour as Input VAT Analysis.

`etr_status` splits on the CU invoice number:

| Dropdown | Query | Keeps |
|---|---|---|
| All | *(omit the param)* | every document |
| Signed | `etr_status=signed` | documents with a `cu_invoice_number`: the ones to declare |
| Not signed | `etr_status=not%20signed` | documents without one, or with an empty one, **including those whose signing failed** |

Any other value, `failed` included, returns `422`. It is a flat query param like the others, not
`filter[etr_status]`. The subtotal and `summary` follow it, and `header.filters.etr_status` echoes
it (`null` when omitted).

```
GET …/reports/tenants/vat-analysis-output?facility_ids[]=1&period_from=2026-07-01&period_to=2026-07-31&tax_id=1
GET …/reports/tenants/vat-analysis-output?facility_ids[]=1&period_from=2026-07-01&period_to=2026-07-31&etr_status=signed
```

### Blocks, wings and floors

With exactly one property selected, `facility_block_ids`, `facility_wing_ids` and
`facility_floor_ids` narrow the report to part of it (several nodes are a union). Each lease counts
**whole** under the node of its **primary space** — the space of the item holding Rent, else
Service Charge, else the lowest-id spaced item. A lease with no space is in the property run
and out of every block / wing / floor run, so the block runs plus those leases add up to the property run.
Invoices and credit notes follow their lease.
Rules, `422`s and the add-up guarantee:
[Splitting by block / wing / floor](./README.md#splitting-by-block--wing--floor). The pickers
are filled from [`GET …/reports/filter-options/structure`](./README.md#picking-blocks-wings-and-floors).

## Buckets

One bucket:

| `bucket` | `header.label` | Rows |
|---|---|---|
| `overview` | `Output VAT` | every invoice and credit note in the period, one per row, ordered by tenant, then document date, then id, then a `subtotal` row |

**Row order.** The default is `header.sort.default` `["tenant"]`, so a tenant's invoices and
credit notes read together. Ties fall back to the order the report builds, which is document date,
then id. Re-sort with [`sort[overview]`](./README.md#sorting-rows--sort), for example
`sort[overview]=-vat` for the largest VAT first. Blanks go last and the subtotal
row stays last. [`columns`](./README.md#choosing-columns--columns) picks the columns shown and
exported.

## Columns

| `key` | `label` | `format` | Notes |
|---|---|---|---|
| `invoice_date` | `Invoice Date` | `string` | `01 Jul, 2026` — from `created_at` |
| `tenant` | `Tenant` | `string` | |
| `lease_id` | `Lease` | `id` | The lease the document was raised on |
| `landlord_identifier` | `Landlord ID` | `string` | What the landlord calls the tenant in their own system (`leases.landlord_identifier`) on that lease; `null` when it has none. `togglable: true`; [hidden as empty](./README.md#empty-columns-start-hidden) when no row has one |
| `invoice_no` | `Invoice No.` | `id` | The document's id. Carries [`link`](./README.md#items--rows): `{ "type": "invoice" }` or `{ "type": "credit_note" }` with the id, when the user may open that kind of document (`view-facility-invoice` / `view-facility-credit-note`). Click it to open the document |
| `cu_invoice_no` | `CU Invoice No.` | `string` | The CU invoice number, `null` when unsigned. Replaces the old `etr_status` column |
| `pin_no` | `PIN No.` | `string` | The tenant's `users.tax_pin` |
| `description` | `Desc.` | `string` | `togglable: true` |
| `taxable_value` | `Taxable Value` | `money` | **Negative** for credit notes |
| `vat` | `VAT` | `money` | **Negative** for credit notes |
| `total` | `Total` | `money` | `type: subtotal` |

> **`invoice_no` is the document's id**, not a formatted document number. `facility_invoices` has no
> `invoice_number` column — `cu_invoice_number` is the only number it carries, and that has its own
> column here.
>
> **Invoices and credit notes share this column but not their numbering**, so invoice 34 and
> credit note 34 can both appear. Open the document from `invoice_no.link`, whose `type` says which
> one it is, never from the value alone.

The subtotal's label spans the eight text columns (`col_span: 8`) and carries only the three money
sums.

## Summary

| Key | Meaning |
|---|---|
| `document_count` | Invoices and credit notes listed |
| `total_taxable` | Sum of `taxable_value`, credit notes netted off |
| `total_vat` | Sum of `vat` — **the output VAT to declare** |
| `total` | Sum of `total` |

## Example — the bucket's rows

```jsonc
"items": [
  { "type": "normal", "invoice_date": { "value": "01 Jul, 2026" }, "tenant": { "value": "Acme Traders Ltd" },
    "lease_id": { "value": 412 }, "landlord_identifier": { "value": "VAT-88" },
    "invoice_no": { "value": 9911, "link": { "type": "invoice", "id": 9911 } }, "cu_invoice_no": { "value": "0100042118000009911" },
    "pin_no": { "value": "P051112223A" }, "description": { "value": "Rent & SC — July 2026" },
    "taxable_value": { "value": 380000.00 }, "vat": { "value": 60800.00 }, "total": { "value": 440800.00 } },

  { "type": "normal", "invoice_date": { "value": "03 Jul, 2026" }, "tenant": { "value": "Njiru Hardware" },
    "lease_id": { "value": 388 }, "landlord_identifier": { "value": null },
    "invoice_no": { "value": 9912, "link": { "type": "invoice", "id": 9912 } }, "cu_invoice_no": { "value": null },
    "pin_no": { "value": "P054445556B" }, "description": { "value": "Rent — July 2026" },
    "taxable_value": { "value": 150000.00 }, "vat": { "value": 24000.00 }, "total": { "value": 174000.00 } },

  { "type": "normal", "invoice_date": { "value": "10 Jul, 2026" }, "tenant": { "value": "Acme Traders Ltd" },
    "lease_id": { "value": 412 }, "landlord_identifier": { "value": "VAT-88" },
    "invoice_no": { "value": 34, "link": { "type": "credit_note", "id": 34 } }, "cu_invoice_no": { "value": "0100042118000000034" },
    "pin_no": { "value": "P051112223A" }, "description": { "value": "Credit note — SC adjustment" },
    "taxable_value": { "value": -50000.00 }, "vat": { "value": -8000.00 }, "total": { "value": -58000.00 } },

  { "type": "subtotal", "background_color": "secondary",
    "invoice_date": { "value": "3 documents", "col_span": 8 },
    "taxable_value": { "value": 480000.00 }, "vat": { "value": 76800.00 }, "total": { "value": 556800.00 } }
]
```

## Export

`…/vat-analysis-output/export?format=excel|pdf`, taking every filter the view endpoint takes. The
frontend exports exactly what is on screen by replaying the query string with `format` appended.
