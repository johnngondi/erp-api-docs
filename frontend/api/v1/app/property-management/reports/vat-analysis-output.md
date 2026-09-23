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

## `etr_status` is derived, not stored

**There is no ETR column on these tables and none was added.** The status is computed per document
from `cu_invoice_number`:

| `cu_invoice_number` | `etr_status` | `color` |
|---|---|---|
| present | `signed` | `success` |
| null or empty | `not signed` | `danger` |

It is a **status string**, not a number or money — `format: string`, centred. The raw value sits
beside it in its own `cu_invoice_no` column, so you can show both without deriving anything on the
frontend.

Credit notes carry `cu_invoice_number` too and are judged the same way.

## Filters

The six [global filters](./README.md#global-filters) plus one:

| Param | Type | Meaning when omitted |
|---|---|---|
| six global filters | see the shared contract | all / current month |
| `tax_id` | `taxes.id` | All taxes |

`tax_id` narrows to **documents carrying at least one line on that tax**; the row still shows the
whole document, not just the matching line. Same behaviour as Input VAT Analysis.

```
GET …/reports/tenants/vat-analysis-output?facility_id=1&period_from=2026-07-01&period_to=2026-07-31&tax_id=1
```

## Buckets

One bucket:

| `bucket` | `header.label` | Rows |
|---|---|---|
| `overview` | `Output VAT` | every invoice and credit note in the period, one per row in date order, then a `subtotal` row |

## Columns

| `key` | `label` | `format` | Notes |
|---|---|---|---|
| `invoice_date` | `Invoice Date` | `string` | `01 Jul, 2026` — from `created_at` |
| `tenant` | `Tenant` | `string` | |
| `lease_id` | `Lease` | `id` | The lease the document was raised on |
| `invoice_no` | `Invoice No.` | `id` | The document's id |
| `cu_invoice_no` | `CU Invoice No.` | `string` | Raw value, `null` when unsigned |
| `pin_no` | `PIN No.` | `string` | The tenant's `users.tax_pin` |
| `description` | `Desc.` | `string` | `togglable: true` |
| `etr_status` | `ETR Status` | `string` | Derived — see above |
| `taxable_value` | `Taxable Value` | `money` | **Negative** for credit notes |
| `vat` | `VAT` | `money` | **Negative** for credit notes |
| `total` | `Total` | `money` | `type: subtotal` |

> **`invoice_no` is the document's id**, not a formatted document number. `facility_invoices` has no
> `invoice_number` column — `cu_invoice_number` is the only number it carries, and that has its own
> column here.

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
    "invoice_no": { "value": 9911 }, "cu_invoice_no": { "value": "0100042118000009911" },
    "pin_no": { "value": "P051112223A" }, "description": { "value": "Rent & SC — July 2026" },
    "etr_status": { "value": "signed", "color": "success" },
    "taxable_value": { "value": 380000.00 }, "vat": { "value": 60800.00 }, "total": { "value": 440800.00 } },

  { "type": "normal", "invoice_date": { "value": "03 Jul, 2026" }, "tenant": { "value": "Njiru Hardware" },
    "invoice_no": { "value": 9912 }, "cu_invoice_no": { "value": null },
    "pin_no": { "value": "P054445556B" }, "description": { "value": "Rent — July 2026" },
    "etr_status": { "value": "not signed", "color": "danger" },
    "taxable_value": { "value": 150000.00 }, "vat": { "value": 24000.00 }, "total": { "value": 174000.00 } },

  { "type": "normal", "invoice_date": { "value": "10 Jul, 2026" }, "tenant": { "value": "Acme Traders Ltd" },
    "invoice_no": { "value": 34 }, "cu_invoice_no": { "value": "0100042118000000034" },
    "pin_no": { "value": "P051112223A" }, "description": { "value": "Credit note — SC adjustment" },
    "etr_status": { "value": "signed", "color": "success" },
    "taxable_value": { "value": -50000.00 }, "vat": { "value": -8000.00 }, "total": { "value": -58000.00 } },

  { "type": "subtotal", "background_color": "secondary",
    "invoice_date": { "value": "3 documents", "col_span": 8 },
    "taxable_value": { "value": 480000.00 }, "vat": { "value": 76800.00 }, "total": { "value": 556800.00 } }
]
```

## Export

`…/vat-analysis-output/export?format=excel|pdf`, taking every filter the view endpoint takes. The
frontend exports exactly what is on screen by replaying the query string with `format` appended.
