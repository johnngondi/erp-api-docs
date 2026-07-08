# Utility Billing (Meter-Reading Based) API

Domain: `Property Management > Lease Management`

Base route:

`/api/v1/app/{company}/property-management/lease-management/charges`

> **Transition note:** This is a new billing flow that computes utility bills directly
> from recorded meter readings. It coexists with the spreadsheet-based charge import
> (see [charges-imports.md](./charges-imports.md)); both remain available.

## Endpoints

- `POST /charges/utility-billing-extract` — read the uploaded provider bill with AI (step 1)
- `POST /charges/utility-billing-fetch` — preview computed rows (+ a `cache_key`) (step 2)
- `POST /charges/utility-billing-process` — bill tenants + raise the provider bill (step 3)

The UI is a 3-step wizard: **extract** the provider bill document, **fetch** the per-lease
meter readings/preview (the tenant rate is derived here from the utility's distribution
method), then **process** the billing.

## Extract Utility Bill

`POST /api/v1/app/{company}/property-management/lease-management/charges/utility-billing-extract`

Reads an uploaded utility provider bill (PDF/image) with the AI document service and returns
the fields that pre-fill the fetch/process steps. The user can edit these before continuing.

Body fields:

| Field | Required | Type | Notes |
|---|---|---|---|
| `facility_id` | Yes | integer | The property. Must exist in `facilities.id` |
| `utility_id` | Yes | integer | The utility (`lease_components.id` where `is_utility_charge = true`) |
| `bill_upload_id` | Yes | integer | `uploads.id` of the provider bill document |

The response echoes the facility utility's `distribution_method` so the FE knows which fields
drive the rate, plus a `result` object. When the document is not a legible utility bill,
`result.is_legible` is `false` and `result.feedback` carries a message; the fields are null.

### Example response

```json
{
  "message": "Utility bill extracted successfully",
  "distribution_method": "utility provider rate",
  "result": {
    "bill_date": "2026-07-01",
    "bill_number": "INV-9911",
    "bill_tax_invoice_number": "TAX-5521",
    "total_bill_amount": 1160,
    "bill_tax_amount": 160,
    "bill_consumption": 450,
    "is_legible": true,
    "feedback": null
  }
}
```

## Fetch Utility Billing Readings

`POST /api/v1/app/{company}/property-management/lease-management/charges/utility-billing-fetch`

Fetches every active lease in a property together with the meter reading for the
selected utility, computes consumption and a bill preview per lease. This is a
**preview/fetch** endpoint — it does **not** create invoices or mark readings billed.

Content type:

- `application/json`

Body fields:

| Field | Required | Type | Notes |
|---|---|---|---|
| `facility_id` | Yes | integer | The property. Must exist in `facilities.id` |
| `utility_id` | Yes | integer | The utility (`lease_components.id` where `is_utility_charge = true`). This is the meter's `utility_id` |
| `tax_id` | Yes | integer | The utility's tax (carried to the provider bill / used as a fallback). Per-lease invoice tax is driven by each `lease.tax_id`, not this value. Must exist in `taxes.id` |
| `total_bill_amount` | Cond. | number | **Total monthly bill charged** (tax-inclusive) — the sum of the current period's charges, NOT the amount payable/balance due after credits or arrears. Required unless `charge_rate` is supplied; drives the derived rate |
| `bill_consumption` | Cond. | number | Total metered consumption on the provider bill. Required for the `distribute to tenants` and `defined` methods |
| `bill_tax_amount` | Cond. | number | Tax portion of the provider bill. **Used in the rate for commercial facilities** (`total_bill_amount − bill_tax_amount`) and carried to the provider bill. Recommended whenever tax applies |
| `charge_rate` | No | number | **Optional flat override.** When supplied, it is used as the per-unit rate for every lease and the distribution method is ignored (no per-lease tax reduction) |

### How the tenant rate is derived

The per-unit `charge_rate` is **computed server-side** from the utility's
`facility_utilities.utility_bill_distribution_method`, the **facility class**
(Commercial / Residential / Mixed Use, from the facility's `facility_type`), and per-lease
taxability, unless a flat `charge_rate` override is supplied. A lease is **taxable** iff its
own `lease.tax_id` references a tax with value > 0; that tax value drives both the rate
reduction and the invoice tax so tenant charges reconcile with the provider bill.

With `T = total_bill_amount`, `X = bill_tax_amount`, `Cb = bill_consumption`,
`Ct = Σ(billable tenant consumption)`, `v = the lease's tax value`:

| Method | Facility class | Rate |
|---|---|---|
| `utility provider rate` | Commercial | `(T − X) / Ct` for every lease (tenants are taxed on the invoice) |
| `utility provider rate` | Residential | `T / Ct` for every lease (no invoice tax) |
| `utility provider rate` | Mixed Use | base `T / Ct`; taxable lease → `base·(1 − v)`, else `base` |
| `distribute to tenants` / `defined` | any | base `T / Cb`; taxable lease → `base·(1 − v)`, else `base` |

If no lease is taxable (or the utility tax is zero-rated) every branch collapses to the plain
rate with no invoice tax.

The response returns the resolved **base** rate as `resolved_charge_rate` and the applied
`distribution_method`; each row also carries its own reduced `charge_rate`, plus `is_taxable`
and `tax_id` (for the `defined` method the FE may edit the rates before processing).

The response includes a **`cache_key`** — an opaque token holding the server-computed
rows. Pass it back to `utility-billing-process` so it bills exactly what was previewed
without trusting client-supplied amounts. Each row also carries `lease_id`,
`previous_utility_meter_reading_id`, and `current_utility_meter_reading_id`.

### How readings are resolved

- For each active lease, the meter serving the lease's space(s) for the selected
  utility is used.
- **Previous reading** = the meter's latest reading where `billed_at` is set. If the
  meter has never been billed, the meter's `initial_reading` is used as the baseline.
- **Current reading** = the meter's most recent reading where `billed_at` is null.
- **Consumption** = `current_reading − previous_reading`.
- **Shared meters:** when one meter serves spaces belonging to multiple leases,
  consumption is split proportionally by space `size`.
- Amount formula: `consumption × charge_rate`; `tax = amount × v` for taxable leases and
  `0` otherwise; `total = amount + tax`. The `charge_rate` is the method-derived (and, for
  taxable leases, tax-reduced) rate — see above.

### Row fields

| Field | Type | Notes |
|---|---|---|
| `lease` | object | `{ id, user: { name }, spaces: [{ id, name, size }] }` |
| `meter` | object\|null | `null` when the lease has no meter for the utility |
| `meter.number` | string | Meter number |
| `meter.is_faulty` / `meter.fault_reason` | bool / string | Meter fault status |
| `meter.previous_reading` | object | Full reading record incl. image `source_url` (or `{ current_reading, is_initial_reading: true }` on first bill) |
| `meter.current_reading` | object\|null | Full reading record incl. image `source_url` |
| `consumption` | number\|null | Units for this lease |
| `consumption_type` | string\|null | `individual` or `shared` |
| `consumption_notes` | string\|null | Sharing breakdown when `shared` |
| `charge_rate` | number\|null | The per-unit rate applied to this lease (tax-reduced for taxable leases; editable per-lease for `defined`) |
| `is_taxable` | bool | Whether this lease is taxed on its invoice (from `lease.tax_id`) |
| `tax_id` | integer\|null | The tax applied to this lease's invoice line (`null` when not taxable) |
| `amount` / `tax` / `total` | number\|null | Computed line figures (`tax` is `0` for non-taxable leases) |
| `has_warning` / `warning_notes` | bool / string | Warns on zero/negative consumption or a faulty meter (amount is still computed) |
| `has_error` / `error_notes` | bool / string | Errors when the lease has no meter, the reading is not submitted, or no unbilled reading exists |

### Preconditions (422)

- If **no** active lease has a meter for the utility:
  `No leases in this property have a meter for the selected utility.`
- If leases have meters but **none** of the meters has been submitted for the period:
  `The readings for the meters have not been submitted for this period.`

### Example response

```json
{
  "message": "Utility billing readings fetched.",
  "cache_key": "8Kd2...q9",
  "distribution_method": "utility provider rate",
  "resolved_charge_rate": 25,
  "rows": [
    {
      "lease_id": 101,
      "previous_utility_meter_reading_id": 900,
      "current_utility_meter_reading_id": 950,
      "lease": {
        "id": 101,
        "user": { "name": "Jane Tenant" },
        "spaces": [{ "id": 12, "name": "Shop A", "size": 100 }]
      },
      "meter": {
        "number": "WM-0012",
        "is_faulty": false,
        "fault_reason": null,
        "previous_reading": {
          "id": 900,
          "current_reading": "120.00000",
          "billed_at": "2026-06-01T00:00:00.000000Z",
          "image": { "id": 44, "source_url": "https://.../uploads/uuid/preview" }
        },
        "current_reading": {
          "id": 950,
          "current_reading": "150.00000",
          "billed_at": null,
          "image": { "id": 61, "source_url": "https://.../uploads/uuid/preview" }
        }
      },
      "consumption": 30,
      "consumption_type": "individual",
      "consumption_notes": null,
      "charge_rate": 25,
      "amount": 750,
      "tax": 120,
      "total": 870,
      "has_warning": false,
      "warning_notes": null,
      "has_error": false,
      "error_notes": null
    },
    {
      "lease": { "id": 102, "user": { "name": "Bob Tenant" }, "spaces": [] },
      "meter": null,
      "consumption": null,
      "consumption_type": null,
      "consumption_notes": null,
      "charge_rate": null,
      "amount": null,
      "tax": null,
      "total": null,
      "has_warning": false,
      "warning_notes": null,
      "has_error": true,
      "error_notes": "This lease has no meter for the selected utility."
    }
  ],
  "summary": {
    "total_rows": 2,
    "billable_rows": 1,
    "error_rows": 1,
    "amount": 750,
    "tax": 120,
    "total": 870
  }
}
```

## Process Utility Billing

`POST /api/v1/app/{company}/property-management/lease-management/charges/utility-billing-process`

Bills every billable lease for the utility (creates one tenant invoice per lease, flags
its item as a utility bill and links the previous/current readings, marks the current
readings billed, and sets the lease's `utility_billed_at`) and raises **one** provider
expense bill against the utility's `FacilityContract`.

Leases whose `utility_billed_at` is already set for the current period are skipped
(cleared monthly by the `leases:reset-utility-billing` command). Rows with zero/negative
consumption or an error are skipped.

Body fields:

| Field | Required | Type | Notes |
|---|---|---|---|
| `facility_id` | Yes | integer | The property |
| `utility_id` | Yes | integer | `lease_components.id` where `is_utility_charge = true` |
| `tax_id` | Yes | integer | The utility's tax / fallback. Per-lease invoice tax is driven by each `lease.tax_id`: taxable leases are taxed, non-taxable leases are billed with a zero-rated tax |
| `notes` | No | string | Prepended into invoice/bill notes |
| `cache_key` | No | string | Token from `utility-billing-fetch`. Bills the cached rows when it matches the same `facility_id`/`utility_id`/`tax_id`/`distribution_method`/`charge_rate`/`total_bill_amount`/`bill_tax_amount`/`bill_consumption`; otherwise the server recomputes. Consumed (forgotten) after processing |
| `charge_rate` | No | number | Optional flat rate override (same semantics as fetch). When omitted, the cached/derived per-lease rates are used |
| `rates` | No | array | Per-lease rate overrides for the `defined` method: `[{ "lease_id": 101, "charge_rate": 30 }]`. Consumption stays server-authoritative; only the rate is overridden |
| `bill_upload_id` | No | integer | `uploads.id` of the provider's bill document, stored on the contract bill as `invoice_upload_id` |
| `bill_date` | No | date | Provider bill date, stored on the contract bill as `invoice_date` |
| `bill_number` | No | string | Provider bill/reference number, stored on the contract bill as `invoice_number` |
| `bill_tax_invoice_number` | No | string | Provider tax invoice number, stored on the contract bill as `tax_invoice_number` |
| `total_bill_amount` | No | number | Total monthly bill charged — **tax-inclusive**, not the amount payable after credits. When omitted, no provider bill is raised |
| `bill_tax_amount` | No | number | Tax portion of the provider bill |
| `bill_consumption` | No | number | Total metered consumption on the provider bill (stored as the bill line quantity) |

Provider bill math (gross-inclusive): `amount = total_bill_amount − bill_tax_amount`,
`tax = bill_tax_amount`, `total = total_bill_amount` (`tax_type = fixed`). The contract
is matched by `facility_id` + `utility_id` (fallback: `facility_id` + the utility's
`vendor_id` from `facility_utilities`). If no contract is found, tenant invoices are
still created and a warning is returned.

### Example response

```json
{
  "message": "Billed 4 lease(s). Skipped 1.",
  "invoice_ids": [9001, 9002, 9003, 9004],
  "bill_id": 555,
  "summary": {
    "billed_leases": 4,
    "skipped_leases": 1,
    "amount": 21250,
    "tax": 3400,
    "total": 24650
  },
  "warnings": [],
  "errors": []
}
```

## Frontend Error Handling

Apply shared rules in `docs/frontend/app/README.md`.
