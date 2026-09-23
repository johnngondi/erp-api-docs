# Tenant Withholding Report API

Domain: `Property Management > Reports > Tenants`

Base route:

`/api/v1/app/{company}/property-management/reports`

Tenant Withholding lists **tax a tenant withheld at source** instead of paying it over, and whether
the certificate for it is on file. It is **read-only** and computed on the fly.

> **Read [the shared reports contract](./README.md) first.** This page documents what is specific to
> this report. The response envelope, the bucket keys, the field keys, the per-cell `{ value, color }`
> shape and the colour vocabulary live in the README and apply here unchanged.

## One report, two nav entries

The nav shows **Withholding VAT** and **Rental Withholding** under Tenant Withholding. They are
**preset filters on a single endpoint**, not two reports:

| | |
|---|---|
| Endpoint | one — `/reports/tenants/tenant-withholding` |
| Permissions | one pair — `view-tenant-withholding-report` / `export-tenant-withholding-report` |
| Nav entries | two, each supplying a preset `payment_method_code` |

[`GET /reports`](./README.md#discovering-reports--get-reports) returns them as **preset** nodes under
the report, each with a `preset_filters` payload. Open the report's own `url` with those filters
applied — a preset has no URL or permission of its own.

```jsonc
"tenant-withholding-vat":    { "type": "preset", "preset_filters": { "payment_method_code": "VAT_WITHHOLDING" } }
"tenant-withholding-rental": { "type": "preset", "preset_filters": { "payment_method_code": "RENTAL_WITHHOLDING" } }
```

A third withholding method later is a config entry, not a new report.

## Methods are resolved by code, never by id

The two methods are the seeded `payment_methods` rows with codes **`VAT_WITHHOLDING`** and
**`RENTAL_WITHHOLDING`**. The report matches on those codes.

**Their ids differ between installations**, so nothing here — the query, the config presets, the
frontend — may key on one. That is why the presets carry `payment_method_code` rather than
`payment_method_id`.

Only these two methods are ever in scope. **Asking for a non-withholding method returns no rows**,
and the bucket is titled `All Withholdings` rather than implying such a withholding exists.

Cancelled receipts are excluded — nothing was withheld.

## The certificate is the receipt's existing upload

**Nothing new is stored for certificates.** No certificate table, no certificate column, no upload
flow was added, and none should be.

Both withholding methods are already seeded `proof_of_payment_type: 'document'`, so the receipt
**already carries** its certificate as the proof-of-payment upload. This report surfaces and links
it:

| Column | Source |
|---|---|
| `certificate_number` | `facility_receipts.transaction_number` |
| `certificate_document` | the URL of the receipt's `pop_upload_id` upload |

### A missing certificate is the exception this report exists to surface

Where a withholding receipt has **no proof-of-payment upload**:

- the **whole row** is tinted `danger`, and
- `certificate_document` is **omitted entirely** from the row.

An absent key renders as an empty cell, per the shared contract — which is the honest rendering.
Don't substitute an empty string or a dead link; the row tint already says what is wrong, and
`summary.missing_certificates` counts them.

## Filters

The six [global filters](./README.md#global-filters) plus two ways to pick one method:

| Param | Type | Meaning when omitted |
|---|---|---|
| six global filters | see the shared contract | all / current month |
| `payment_method_code` | `VAT_WITHHOLDING` \| `RENTAL_WITHHOLDING` | Both methods |
| `payment_method_id` | `payment_methods.id` | Both methods |

`payment_method_code` is what the nav presets use. `payment_method_id` is there for ad-hoc use when
the caller already holds a real id. **Given both, the id wins** — it is the more specific request.

```
GET …/reports/tenants/tenant-withholding?facility_id=1&payment_method_code=VAT_WITHHOLDING
```

## Buckets

One bucket:

| `bucket` | `header.label` | Rows |
|---|---|---|
| `overview` | `All Withholdings`, or `{method name} Withheld` when one is selected | one row per withholding receipt in date order, then a `subtotal` row |

The label is how a preset view announces itself — `VAT Withholding Withheld`,
`Rental Withholding Withheld`.

## Columns

| `key` | `label` | `format` | Notes |
|---|---|---|---|
| `date` | `Date` | `string` | `12 Jul, 2026` — the receipt's transaction date |
| `tenant` | `Tenant` | `string` | The paying user |
| `lease_id` | `Lease` | `string` | **A list** — see below. `—` when the receipt is allocated to nothing |
| `property` | `Property` | `string` | |
| `invoice_no` | `Invoice No.` | `string` | **Every** invoice the receipt was allocated against, comma-separated; `—` where it was allocated to none |
| `amount_withheld` | `Amount Withheld` | `money` | The receipt amount |
| `payment_method` | `Payment Method` | `string` | `VAT Withholding` / `Rental Withholding` |
| `certificate_number` | `Certificate No.` | `string` | The receipt's transaction number |
| `certificate_document` | `Certificate` | `string` | URL of the proof-of-payment upload. **Omitted when absent** |

The subtotal's label spans the five text columns (`col_span: 5`) and carries only
`amount_withheld`.

### Why `lease_id` is a list here

**A receipt is not held against a lease.** It is taken on a property from a paying user, and only
reaches a lease through the invoices it settles — `facility_receipt_allocations` → `facility_invoices`
→ `lease_id`. A receipt can settle invoices on more than one lease, so the column lists every lease
it touches (`"12, 15"`), exactly as `invoice_no` lists every invoice, rather than passing the first
off as the only one. That is why its `format` is `string` where the other reports use `id`.

## Summary

| Key | Meaning |
|---|---|
| `receipt_count` | Withholding receipts listed |
| `total_withheld` | Sum of `amount_withheld` |
| `missing_certificates` | Rows with no proof-of-payment upload — the `danger` rows |

## Example — the bucket's rows

```jsonc
"items": [
  { "type": "normal",
    "date": { "value": "12 Jul, 2026" }, "tenant": { "value": "Acme Traders Ltd" },
    "property": { "value": "KAHAWA HOUSE" }, "invoice_no": { "value": "9911" },
    "amount_withheld": { "value": 8816.00 }, "payment_method": { "value": "VAT Withholding" },
    "certificate_number": { "value": "WHT2026071200491" },
    "certificate_document": { "value": "https://…/uploads/91/whv-cert-0491.pdf" } },

  // No proof of payment on file: whole row danger, and no certificate_document key at all.
  { "type": "normal", "background_color": "danger",
    "date": { "value": "18 Jul, 2026" }, "tenant": { "value": "Njiru Hardware" },
    "property": { "value": "KAHAWA HOUSE" }, "invoice_no": { "value": "9912" },
    "amount_withheld": { "value": 3480.00 }, "payment_method": { "value": "Rental Withholding" },
    "certificate_number": { "value": "RWT2026071800112" } },

  { "type": "subtotal", "background_color": "secondary",
    "date": { "value": "2 receipts", "col_span": 4 },
    "amount_withheld": { "value": 12296.00 } }
]
```

## Export

`…/tenant-withholding/export?format=excel|pdf`, taking every filter the view endpoint takes —
`payment_method_code` included, so a preset view exports as the preset. The frontend exports the
current view by replaying the query string with `format` appended.
