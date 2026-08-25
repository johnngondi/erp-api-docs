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
| `bill_consumption` | Cond. | number | Total metered consumption on the provider bill. Required for the `utility provider rate` and `defined` methods |
| `bill_tax_amount` | Cond. | number | Tax portion of the provider bill. **Used in the rate for commercial facilities** (`total_bill_amount − bill_tax_amount`) and carried to the provider bill. Recommended whenever tax applies |
| `charge_rate` | No | number | **Optional flat override.** When supplied, it is used as the per-unit rate for every lease, the distribution method is ignored (no per-lease tax reduction) and no indirect consumption is computed |

### How the tenant rate is derived

The per-unit `charge_rate` is **computed server-side** from the utility's
`facility_utilities.utility_bill_distribution_method`, the **facility class**
(Commercial / Residential / Mixed Use, from the facility's `facility_type`), and per-lease
taxability, unless a flat `charge_rate` override is supplied. A lease is **taxable** iff its
own `lease.tax_id` references a tax with value > 0; that tax value drives both the rate
reduction and the invoice tax so tenant charges reconcile with the provider bill.

The two main methods differ in **what the bill is divided by**:

- **`utility provider rate`** — tenants pay their own metered units at the provider's own
  per-unit price, so the denominator is the **bill's** consumption. Units the provider
  billed that no tenant meter accounts for become *indirect consumption* (see below).
- **`distribute to tenants`** — the **whole** bill is spread over what tenants actually
  used, so the denominator is **tenant** consumption. Nothing is left over.

With `T = total_bill_amount`, `X = bill_tax_amount`, `Cb = bill_consumption`,
`Ct = Σ(billable tenant consumption)`, `v = the lease's tax value`:

| Method | Facility class | Rate |
|---|---|---|
| `utility provider rate` | Commercial | `(T − X) / Cb` for every lease (tenants are taxed on the invoice) |
| `utility provider rate` | Residential | `T / Cb` for every lease (no invoice tax) |
| `utility provider rate` | Mixed Use | base `T / Cb`; taxable lease → `base / (1 + v)`, else `base` |
| `distribute to tenants` | any | base `T / Ct`; taxable lease → `base / (1 + v)`, else `base` |
| `defined` | any | base `T / Cb` seeds an editable per-lease rate; taxable lease → `base / (1 + v)` |

If no lease is taxable (or the utility tax is zero-rated) every branch collapses to the plain
rate with no invoice tax.

The taxable reduction **divides by `(1 + v)`**, it does not multiply by `(1 − v)`. The provider
bill total is tax-inclusive, so the tax-exclusive rate is `T / (1 + v)`; re-adding `v` on the
invoice then rebuilds the gross share exactly. Under `distribute to tenants` this makes
`Σ(amount + tax) = T` to the cent. (Multiplying by `(1 − v)` would yield `(1 − v)(1 + v) = 1 − v²`
of the bill — a silent shortfall of `v²`, 2.56% at `v = 0.16`.)

Note the invoice tax is the **lease's own** tax value, so `bill_tax_amount` from the provider
bill is reproduced only when the provider taxed at that same rate. A bill carrying non-VATable
levies has a lower implied tax rate, so the invoice `tax` will exceed the bill's `X` while
`amount + tax` still reconciles to `T`.

The response returns the resolved **base** rate as `resolved_charge_rate` and the applied
`distribution_method`; each row also carries its own reduced `charge_rate`, plus `is_taxable`
and `tax_id` (for the `defined` method the FE may edit the rates before processing).

Alongside it the response returns **`bill_rate`** — the provider's own per-unit price,
`total_bill_amount / bill_consumption` (**gross**, tax included), read straight off the bill
and independent of the distribution method. Display it beside `resolved_charge_rate` so the
user can see what the tenants are charged versus what the provider charged. It is `null`
when `total_bill_amount` or `bill_consumption` was not supplied (e.g. a flat `charge_rate`
override). Note that for a **commercial** facility `resolved_charge_rate` is derived net of
tax (`(T − X) / Cb`), so it is legitimately lower than `bill_rate`.

### Tenant vs. bill consumption

For the same side-by-side comparison on units, the response returns
**`total_tenant_consumption`** (`Ct`, the units the tenant meters actually recorded) next to
the **`bill_consumption`** that was submitted. `summary.tenant_consumption`,
`summary.bill_consumption` and `summary.indirect_consumption` repeat them in the summary
block. The gap between the two is what the indirect split spreads (or the landlord absorbs).

### Indirect consumption

Under `utility provider rate` the tenant meters rarely add up to the provider's meter. The
difference — common areas, losses — is **indirect consumption**:

```
U   = max(Cb − Ct, 0)                       total indirect units (total_indirect_consumption)
u   = U × (lease space size / total facility space size)   this lease's units
amt = u × the lease's charge_rate
```

It is only computed when the utility has `facility_utilities.bill_indirect_consumption =
true`; otherwise `total_indirect_consumption` is `0` and the landlord absorbs it. The
denominator is **all** space in the property, including vacant and non-leased space, so the
share belonging to unlet space also stays with the landlord. A lease gets **one** row, so it
receives its indirect share exactly once however many meters serve it.

At the process step this becomes a **second invoice line** — see
[Process Utility Billing](#process-utility-billing).

The response includes a **`cache_key`** — an opaque token holding the server-computed
rows. Pass it back to `utility-billing-process` so it bills exactly what was previewed
without trusting client-supplied amounts. Each row also carries `lease_id`,
`previous_utility_meter_reading_id`, and `current_utility_meter_reading_id`.

### One row per lease

Rows are **per lease**, never per meter — `summary.total_rows` counts leases.

- **A lease with several meters** gets a single row: consumptions are summed, and the
  readings are summed too (so `current_reading − previous_reading` still equals the row's
  `consumption`). Reading images become an **array** (`meter.previous_reading.images` /
  `meter.current_reading.images`), and `meters[]` carries the per-meter breakdown for a
  drill-down. `meter.number` is the meter numbers joined with `", "`; `meter.numbers` is the
  same as an array and `meter.count` is how many. The plural
  `previous_utility_meter_reading_ids` / `current_utility_meter_reading_ids` list every
  reading the row was billed from; the singular keys keep the first meter's.
- **Leases sharing a meter** (e.g. Jisaidie and ELOG) are returned **consecutively** and tagged
  with the same `meter_group` (a 1-based integer) plus `shared_with_lease_ids` (the other
  leases in that group), so the split is easy to read without hunting across the table.
  Leases with a meter of their own keep their natural position and get their own
  `meter_group` with an empty `shared_with_lease_ids`.
- **`lease.spaces`** lists only the spaces a meter for the selected utility is attached to —
  other spaces on the lease are omitted. Each entry carries the `meter_number` serving it.
- **Reading images** are trimmed to `{ title, type, source_url }`.

### How readings are resolved

- For each active lease, the meter serving the lease's space(s) for the selected
  utility is used.
- **Previous reading** = the meter's latest reading where `billed_at` is set. If the
  meter has never been billed, the meter's `initial_reading` is used as the baseline.
- **Current reading** = the meter's most recent reading where `billed_at` is null.
- **Consumption** = `current_reading − previous_reading`.
- **Shared meters:** when one meter serves spaces belonging to multiple leases,
  consumption is split proportionally by space `size`.
- **Several meters on one lease:** each meter's consumption is computed on its own and the
  results are summed into the lease's single row (see [One row per lease](#one-row-per-lease)).
- Amount formula: `consumption × charge_rate`; `tax = amount × v` for taxable leases and
  `0` otherwise; `total = amount + tax`. The `charge_rate` is the method-derived (and, for
  taxable leases, tax-reduced) rate — see above.

### Row fields

| Field | Type | Notes |
|---|---|---|
| `lease` | object | `{ id, user: { name }, spaces: [{ id, name, size, meter_number }] }` — **metered spaces only** |
| `meter_group` | integer\|null | Groups leases that share a meter; rows of a group are returned consecutively. `null` on a no-meter row |
| `shared_with_lease_ids` | integer[] | The other leases in this `meter_group` (empty when the lease has its meter to itself) |
| `meter` | object\|null | The lease's meters merged into one block. `null` when the lease has no meter for the utility |
| `meter.number` | string | Meter number(s), joined with `", "` when a lease has several |
| `meter.numbers` / `meter.count` | string[] / integer | The same numbers as an array, and how many meters the row merges |
| `meter.is_faulty` / `meter.fault_reason` | bool / string | True if **any** of the row's meters is faulty; the reason(s), prefixed by meter number |
| `meter.previous_reading` | object | `{ current_reading, is_initial_reading, images[] }` — reading **summed** across the row's meters; `is_initial_reading` is true when a meter baselined off its `initial_reading` |
| `meter.current_reading` | object\|null | `{ current_reading, images[] }`, summed the same way. `null` when no meter has an unbilled reading |
| `meter.*.images[]` | object[] | `{ title, type, source_url }` per reading image; empty when none were captured |
| `meters[]` | object[] | Per-meter breakdown: `{ number, is_faulty, fault_reason, previous_reading, current_reading, consumption, consumption_type, consumption_notes, previous/current_utility_meter_reading_id }`. Single entry for a one-meter lease |
| `previous_utility_meter_reading_ids` / `current_utility_meter_reading_ids` | integer[] | Every reading the row was billed from. The singular keys hold the first meter's (that is the one the invoice line links) |
| `consumption` | number\|null | Units for this lease, summed across its meters |
| `consumption_type` | string\|null | `individual` or `shared` (`shared` if **any** of the row's meters is shared) |
| `consumption_notes` | string\|null | Sharing breakdown, one sentence per shared meter |
| `charge_rate` | number\|null | The per-unit rate applied to this lease (tax-reduced for taxable leases; editable per-lease for `defined`) |
| `is_taxable` | bool | Whether this lease is taxed on its invoice (from `lease.tax_id`) |
| `tax_id` | integer\|null | The tax applied to this lease's invoice line (`null` when not taxable) |
| `amount` / `tax` / `total` | number\|null | Computed **direct** line figures (`tax` is `0` for non-taxable leases) |
| `indirect_consumption` | number | This lease's share of the unmetered units (`0` when indirect is not billed) |
| `indirect_amount` / `indirect_tax` / `indirect_total` | number | The indirect line's figures, priced at the same `charge_rate` |
| `has_warning` / `warning_notes` | bool / string | Warns on zero/negative consumption or a faulty meter (amount is still computed) |
| `has_error` / `error_notes` | bool / string | Errors when the lease has no meter, the reading is not submitted, or no unbilled reading exists |

### Preconditions (422)

- If **no** active lease has a meter for the utility:
  `No leases in this property have a meter for the selected utility.`
- If leases have meters but **none** of the meters has been submitted for the period:
  `The readings for the meters have not been submitted for this period.`

### Example response

Lease 101 below is served by **two** meters, so its row merges them (readings summed,
images collected, `meters[]` holding the detail). Lease 102 has no meter for the utility.

```json
{
  "message": "Utility billing readings fetched.",
  "cache_key": "8Kd2...q9",
  "distribution_method": "utility provider rate",
  "resolved_charge_rate": 25,
  "bill_rate": 29,
  "bill_indirect_consumption": true,
  "total_indirect_consumption": 20,
  "total_tenant_consumption": 50,
  "bill_consumption": 70,
  "rows": [
    {
      "lease_id": 101,
      "meter_group": 1,
      "shared_with_lease_ids": [],
      "previous_utility_meter_reading_id": 900,
      "current_utility_meter_reading_id": 950,
      "previous_utility_meter_reading_ids": [900, 901],
      "current_utility_meter_reading_ids": [950, 951],
      "lease": {
        "id": 101,
        "user": { "name": "Jane Tenant" },
        "spaces": [
          { "id": 12, "name": "Shop A", "size": 100, "meter_number": "WM-0012" },
          { "id": 13, "name": "Store A", "size": 40, "meter_number": "WM-0031" }
        ]
      },
      "meter": {
        "number": "WM-0012, WM-0031",
        "numbers": ["WM-0012", "WM-0031"],
        "count": 2,
        "is_faulty": false,
        "fault_reason": null,
        "previous_reading": {
          "current_reading": 220,
          "is_initial_reading": false,
          "images": [
            { "title": "June reading", "type": "image", "source_url": "https://.../uploads/uuid/preview" }
          ]
        },
        "current_reading": {
          "current_reading": 270,
          "images": [
            { "title": "July reading", "type": "image", "source_url": "https://.../uploads/uuid/preview" }
          ]
        }
      },
      "meters": [
        {
          "number": "WM-0012",
          "is_faulty": false,
          "fault_reason": null,
          "previous_utility_meter_reading_id": 900,
          "current_utility_meter_reading_id": 950,
          "previous_reading": {
            "id": 900,
            "current_reading": 120,
            "is_initial_reading": false,
            "is_faulty": false,
            "fault_reason": null,
            "billed_at": "2026-06-01T00:00:00.000000Z",
            "recorded_at": "2026-06-01T00:00:00.000000Z",
            "image": { "title": "June reading", "type": "image", "source_url": "https://.../uploads/uuid/preview" }
          },
          "current_reading": {
            "id": 950,
            "current_reading": 150,
            "is_initial_reading": false,
            "is_faulty": false,
            "fault_reason": null,
            "billed_at": null,
            "recorded_at": "2026-07-01T00:00:00.000000Z",
            "image": { "title": "July reading", "type": "image", "source_url": "https://.../uploads/uuid/preview" }
          },
          "consumption": 30,
          "consumption_type": "individual",
          "consumption_notes": null
        },
        {
          "number": "WM-0031",
          "is_faulty": false,
          "fault_reason": null,
          "previous_utility_meter_reading_id": 901,
          "current_utility_meter_reading_id": 951,
          "previous_reading": { "id": 901, "current_reading": 100, "is_initial_reading": false, "image": null },
          "current_reading": { "id": 951, "current_reading": 120, "is_initial_reading": false, "image": null },
          "consumption": 20,
          "consumption_type": "individual",
          "consumption_notes": null
        }
      ],
      "consumption": 50,
      "consumption_type": "individual",
      "consumption_notes": null,
      "charge_rate": 25,
      "amount": 1250,
      "tax": 200,
      "total": 1450,
      "indirect_consumption": 5,
      "indirect_amount": 125,
      "indirect_tax": 20,
      "indirect_total": 145,
      "has_warning": false,
      "warning_notes": null,
      "has_error": false,
      "error_notes": null
    },
    {
      "lease_id": 102,
      "lease": { "id": 102, "user": { "name": "Bob Tenant" }, "spaces": [] },
      "meter_group": null,
      "shared_with_lease_ids": [],
      "meter": null,
      "meters": [],
      "previous_utility_meter_reading_id": null,
      "current_utility_meter_reading_id": null,
      "previous_utility_meter_reading_ids": [],
      "current_utility_meter_reading_ids": [],
      "consumption": null,
      "consumption_type": null,
      "consumption_notes": null,
      "charge_rate": null,
      "amount": null,
      "tax": null,
      "total": null,
      "indirect_consumption": 0,
      "indirect_amount": 0,
      "indirect_tax": 0,
      "indirect_total": 0,
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
    "tenant_consumption": 50,
    "bill_consumption": 70,
    "indirect_consumption": 20,
    "amount": 1250,
    "tax": 200,
    "indirect_amount": 125,
    "indirect_tax": 20,
    "indirect_total": 145,
    "total": 1595
  }
}
```

Two leases sharing one meter come back next to each other, cross-referenced:

```json
[
  { "lease_id": 101, "meter_group": 1, "shared_with_lease_ids": [103], "consumption_type": "shared",
    "consumption_notes": "Shared meter WM-0012 across 2 leases; this lease's share 25% (size 100 of 400)." },
  { "lease_id": 103, "meter_group": 1, "shared_with_lease_ids": [101], "consumption_type": "shared",
    "consumption_notes": "Shared meter WM-0012 across 2 leases; this lease's share 75% (size 300 of 400)." }
]
```

## Process Utility Billing

`POST /api/v1/app/{company}/property-management/lease-management/charges/utility-billing-process`

Bills every billable lease for the utility (creates one tenant invoice per lease, flags
its items as utility bills and links the previous/current readings, marks the current
readings billed, and sets the lease's `utility_billed_at`) and raises **one** provider
expense bill against the utility's `FacilityContract`.

Each invoice carries **one or two** lines, both against the lease's utility component:

| Line | When | Notes text | Quantity |
|---|---|---|---|
| Direct | always | `Electricity — Direct Consumption: Prev:522 Curr:566 Cons: 44` | the lease's metered consumption |
| Indirect | `indirect_consumption > 0` | `Electricity — Indirect Consumption: 5 of total indirect bill 50` | the lease's share of the unmetered units |

Both are priced at the same `charge_rate`. Only the direct line links the meter readings
(`previous_utility_meter_reading_id` / `current_utility_meter_reading_id`), so it is the
one that shows the reading images.

A lease served by several meters is billed **once**, on its merged row: one invoice with a
single direct line for the summed consumption, whose notes carry the summed readings
(`Prev:1600 Curr:2100 Cons: 500`). **Every** reading in the row's
`current_utility_meter_reading_ids` is marked billed, not just the one the line links, so no
meter is re-billed next period.

A lease is skipped only when its `utility_billed_at` falls **inside the current period**
(on or after the first of the current month). A lease with no `utility_billed_at` has simply
never been billed and is read as *last billed in the previous period*, so it qualifies — a
lease onboarded today bills on the very first run, and a lease last billed in an earlier
period reopens on its own. The `leases:reset-utility-billing` command is therefore no longer
required monthly; it remains a manual override to force a re-bill within the current period.
Rows with zero/negative consumption or an error are skipped.

Body fields:

| Field | Required | Type | Notes |
|---|---|---|---|
| `facility_id` | Yes | integer | The property |
| `utility_id` | Yes | integer | `lease_components.id` where `is_utility_charge = true` |
| `tax_id` | Yes | integer | The utility's tax / fallback. Per-lease invoice tax is driven by each `lease.tax_id`: taxable leases are taxed, non-taxable leases are billed with a zero-rated tax |
| `notes` | No | string | Prepended into invoice/bill notes |
| `cache_key` | No | string | Token from `utility-billing-fetch`. Bills the cached rows when it matches the same `facility_id`/`utility_id`/`tax_id`/`distribution_method`/`charge_rate`/`total_bill_amount`/`bill_tax_amount`/`bill_consumption`; otherwise the server recomputes. Carries the indirect-consumption split too. Consumed (forgotten) after processing |
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
