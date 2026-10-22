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
- `GET /charges/utility-billings` — the utility billing history: past runs, with a totals row (row 156)
- `GET /charges/utility-billings/{utilityBilling}` — one run, per tenant, with its meters (row 156)

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
| `tax_id` | Yes | integer | The tax for this run. It is applied to **every** lease's utility lines, whatever tax the lease itself carries; pick a 0% tax for an untaxed utility such as water. Must exist in `taxes.id` |
| `total_bill_amount` | Cond. | number | **Total monthly bill charged** (tax-inclusive) — the sum of the current period's charges, NOT the amount payable/balance due after credits or arrears. Required unless `charge_rate` is supplied; drives the derived rate |
| `bill_consumption` | Cond. | number | Total metered consumption on the provider bill. Required for the `utility provider rate` and `defined` methods |
| `bill_tax_amount` | Cond. | number | Tax portion of the provider bill. When the selected tax is above 0% the rate is priced on `total_bill_amount − bill_tax_amount`; when it is omitted the net falls back to `total_bill_amount / (1 + tax value)`. Ignored for the rate under a 0% tax. Carried to the provider bill. Recommended whenever tax applies |
| `charge_rate` | No | number | **Optional flat override.** When supplied, it is used as the per-unit rate for every lease, the distribution method is ignored (no tax reduction; the rows are still taxed at the selected tax) and no indirect consumption is computed |

### How the tenant rate is derived

The per-unit `charge_rate` is **computed server-side** from the utility's
`facility_utilities.utility_bill_distribution_method` and the **selected tax** (`tax_id`),
unless a flat `charge_rate` override is supplied. The selected tax applies to **every lease
alike**: the lease's own `lease.tax_id` and the property's type (commercial, residential,
mixed use) play no part in utility billing.

- **16% (any tax above 0%)** taxes every tenant. The rate is priced on the bill **net of tax**
  and the invoice tax adds the tax back, so tenant charges rebuild the gross bill.
- **0%** taxes nobody. The rate is priced on the **gross** bill and no invoice tax is added.

The two main methods differ in **what the bill is divided by**:

- **`utility provider rate`** — tenants pay their own metered units at the provider's own
  per-unit price, so the denominator is the **bill's** consumption. Units the provider
  billed that no tenant meter accounts for become *indirect consumption* (see below).
- **`distribute to tenants`** — the **whole** bill is spread over what tenants actually
  used, so the denominator is **tenant** consumption. Nothing is left over.

With `T = total_bill_amount`, `X = bill_tax_amount`, `Cb = bill_consumption`,
`Ct = Σ(billable tenant consumption)`, `v = the selected tax's value` (a fraction, `0.16` = 16%):

- `taxed = v > 0`
- `net = T − X` when `bill_tax_amount` is supplied, else `T / (1 + v)`

| Method | Rate (every lease) |
|---|---|
| `utility provider rate` | `(taxed ? net : T) / Cb` |
| `distribute to tenants` | `(taxed ? net : T) / Ct` |
| `defined` | `(taxed ? net : T) / Cb` seeds an editable per-lease rate |
| flat `charge_rate` override | the override, flat (no reduction) |

Every priced row then carries `tax = round(amount × v, 2)`, `tax_id` = the selected tax and
`is_taxable = taxed`; the indirect figures are taxed the same way.

The fallback net **divides by `(1 + v)`**, it does not multiply by `(1 − v)`. The provider
bill total is tax-inclusive, so the tax-exclusive figure is `T / (1 + v)`; re-adding `v` on the
invoice then rebuilds the gross exactly. Under `distribute to tenants` this makes
`Σ(amount + tax) = T` to the cent. (Multiplying by `(1 − v)` would yield `(1 − v)(1 + v) = 1 − v²`
of the bill — a silent shortfall of `v²`, 2.56% at `v = 0.16`.)

A typed-in `bill_tax_amount` wins over the fallback because provider bills carry levies and
rounding that a percentage does not reproduce. The invoice tax is still `amount × v`, so when
the bill's `X` is not exactly `v` of its net, `amount + tax` differs from `T` by that gap.

The response returns the resolved rate as `resolved_charge_rate` (the same rate every row
carries as `charge_rate`; for the `defined` method the FE may edit each row's rate before
processing), the applied `distribution_method`, and the selected tax as top-level **`tax_id`**
and **`tax_rate`** (the tax's value as a fraction). Use `tax_rate` to re-derive a row's tax
after a local rate edit (`tax = amount × tax_rate`) instead of the row's `tax / amount` ratio,
which is undefined on a zero amount.

Alongside it the response returns **`bill_rate`** — the provider's own per-unit price,
`total_bill_amount / bill_consumption` (**gross**, tax included), read straight off the bill
and independent of the distribution method. Display it beside `resolved_charge_rate` so the
user can see what the tenants are charged versus what the provider charged. It is `null`
when `total_bill_amount` or `bill_consumption` was not supplied (e.g. a flat `charge_rate`
override). When the selected tax is above 0%, `resolved_charge_rate` is derived net of tax
(e.g. `(T − X) / Cb`), so it is legitimately lower than `bill_rate`.

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
- **`lease.spaces`** lists the spaces the lease holds (from its lease items), and
  **`lease.meters`** the lease's meters for the selected utility. Meters are attached to
  leases, not spaces.
- **Reading images** are trimmed to `{ title, type, source_url }`.

### How readings are resolved

- For each active lease, the meters attached to the lease for the selected utility are used.
- **Previous reading** = the meter's latest reading where `billed_at` is set. If the
  meter has never been billed, the meter's `initial_reading` is used as the baseline.
- **Current reading** = the meter's most recent reading where `billed_at` is null.
- **Consumption** = `current_reading − previous_reading`.
- **Shared meters:** when one meter is attached to several active leases, consumption is
  split proportionally by each lease's leased area (the summed `size` of the spaces on its
  lease items; a space with no size counts as 1).
- **Several meters on one lease:** each meter's consumption is computed on its own and the
  results are summed into the lease's single row (see [One row per lease](#one-row-per-lease)).
- Amount formula: `consumption × charge_rate`; `tax = amount × v` with `v` the selected
  tax's value (`0` under a 0% tax); `total = amount + tax`. The `charge_rate` is the
  method-derived rate, net of tax when the selected tax is above 0% — see above.

### Row fields

| Field | Type | Notes |
|---|---|---|
| `lease` | object | `{ id, user: { name }, spaces: [{ id, name, size }], meters: [{ id, name, meter_number }] }` — `spaces` is every space on the lease, `meters` the lease's meters for the utility |
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
| `charge_rate` | number\|null | The per-unit rate applied to this lease: the same for every lease (net of tax when the selected tax is above 0%; editable per-lease for `defined`) |
| `is_taxable` | bool | Whether the selected tax is above 0%. The same for every priced row; the lease's own tax is not read |
| `tax_id` | integer\|null | The selected tax (the request's `tax_id`) on every priced row; `null` only on rows that were not priced (errors, zero consumption) |
| `amount` / `tax` / `total` | number\|null | Computed **direct** line figures (`tax = amount × tax_rate`, `0` under a 0% tax) |
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
  "tax_id": 3,
  "tax_rate": 0.16,
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
          { "id": 12, "name": "Shop A", "size": 100 },
          { "id": 13, "name": "Store A", "size": 40 }
        ],
        "meters": [
          { "id": 7, "name": "Shop A", "meter_number": "WM-0012" },
          { "id": 9, "name": "Store A", "meter_number": "WM-0031" }
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
      "tax_id": 3,
      "is_taxable": true,
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
      "lease": { "id": 102, "user": { "name": "Bob Tenant" }, "spaces": [{ "id": 14, "name": "Shop B", "size": 80 }], "meters": [] },
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

Each invoice carries **one direct line per meter** that serves the lease, plus at most one
indirect line, all against the lease's utility component:

| Line | When | Notes text | Quantity |
|---|---|---|---|
| Direct (one per meter) | the meter's `consumption > 0` | `Electricity — Meter WM-001: Prev: 1600 Curr: 2100 Cons: 500` | that meter's consumption (for a shared meter, this lease's share) |
| Indirect | `indirect_consumption > 0` | `Electricity — Indirect Consumption: 5 of total indirect bill 50` | the lease's share of the unmetered units |

Direct-line notes:

- On a meter's first bill (no billed reading yet, so the previous side is the meter's
  `initial_reading`), the previous label reads `Prev (initial):`, e.g.
  `Electricity — Meter WM-001: Prev (initial): 0 Curr: 120 Cons: 120`.
- For a shared meter the row's share wording is appended:
  `Electricity — Meter WM-0012: Prev: 1000 Curr: 1400 Cons: 100 Shared meter WM-0012 across 2 leases; this lease's share 25% (size 100 of 400).`
- Notes are capped at 191 characters (longer text is cut and ends with `…`).

Every line is priced at the row's `charge_rate` and carries the request's `tax_id`. Each
direct line links **its own meter's** readings (`previous_utility_meter_reading_id` /
`current_utility_meter_reading_id`), so the invoice shows one group of reading images per
meter. The indirect line links none.

A lease served by several meters is still billed **once** (one invoice, from its single
preview row): the invoice gets a direct line per meter, built from the row's `meters[]`.
A meter whose consumption is zero or negative gets no line (and a warning); a lease where
no meter has a positive consumption is skipped. **Every** reading in the row's
`current_utility_meter_reading_ids` is marked billed, so no meter is re-billed next period.
A row without `meters[]` (a preview cached before this change) falls back to a single direct
line, `Electricity — Direct Consumption: Prev:522 Curr:566 Cons: 44`, linked to the row's
first meter.

A lease is skipped only when it was already billed **for this utility** inside the current period
(on or after the first of the current month): its `utility_billed_at` falls in the period **and**
one of its invoices raised in the period carries a utility line for this utility. Each utility is
billed once a period on its own, so billing water does not stop the same tenants' electricity. A
cancelled or rejected invoice does not count, so cancelling a wrong utility bill reopens that
utility for the period. The skip reads `Lease 93 skipped: Water already billed for this period.`
A lease with no `utility_billed_at` has simply
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
| `tax_id` | Yes | integer | The tax put on **every** invoice line of every lease, whatever tax the lease carries. A 0% tax yields no invoice tax |
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
  "utility_billing_id": 31,
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

### Splitting invoices billed before per-meter lines

Invoices raised before lines were written per meter hold **one** summed line for a lease on
several meters, linked to the first meter's readings only. The command below splits them in
place; the invoice's amount, tax, total, paid and balance do not move, only its lines change.

```
php artisan utility-billing:split-merged-lines --dry-run        # list what would be split
php artisan utility-billing:split-merged-lines                  # split
php artisan utility-billing:split-merged-lines --company=3      # one company
php artisan utility-billing:split-merged-lines --invoice=9001 --invoice=9002
php artisan utility-billing:split-merged-lines --include-signed # also invoices with a CU number
```

- A candidate is a utility line whose linked reading is billed and whose lease has other
  meters for the same utility with readings stamped `billed_at` in the same run and linked to
  no line. Cancelled and rejected invoices are skipped; invoices carrying a CU (eTIMS) invoice
  number are skipped unless `--include-signed` is given.
- The original line keeps its meter and is renamed to the per-meter format; each other meter's
  reading becomes a new line on the same component, cost and tax, linked to its own previous
  (or initial) and current readings.
- Quantities are the meters' own consumption when they add up to the line (an
  individual-meter lease); otherwise they are shared in proportion (a shared meter billed the
  lease only its share) and the row is marked `proportional`. Money follows the quantities
  with the last line absorbing rounding, and the run is rolled back if any sum would change.
- Running it again is a no-op: readings already linked to a line are never split again.

### Billing readings a run left behind (lease not active at the time)

The fetch step only lists **active** leases, so a meter read while its lease was still pending
(or suspended) keeps an unbilled reading after the run. Once the lease is active, the command
below bills those readings without waiting for the next period's run.

```
php artisan utility-billing:bill-unbilled-readings --dry-run             # list what would be billed
php artisan utility-billing:bill-unbilled-readings                       # bill everything found
php artisan utility-billing:bill-unbilled-readings --facility=12         # one property
php artisan utility-billing:bill-unbilled-readings --utility=4           # one utility (lease component id)
php artisan utility-billing:bill-unbilled-readings --company=3           # one company
php artisan utility-billing:bill-unbilled-readings --lease=901 --lease=902
php artisan utility-billing:bill-unbilled-readings --rate=25 --tax=2     # price with this rate and tax id
php artisan utility-billing:bill-unbilled-readings --notes="Catch-up"    # prefixed to the invoice notes
```

- A candidate is a meter on at least one **active** lease whose newest reading is unbilled.
  Candidates are grouped per property + utility and each group is billed like a normal run of
  that utility: shared meters split by leased area, one invoice line per meter linked to its own
  readings, the first bill priced from the meter's initial reading. The submission check is
  off, since `utility-meters:reset-submissions` may already have cleared `submitted_at`.
- The per-unit rate and tax are taken from the **latest utility invoice line** of the same
  property + utility (cancelled and rejected invoices ignored), i.e. the run the reading missed
  or the closest one to it. `--rate` and `--tax` replace either or both; a property with no
  earlier run is skipped unless both are given.
- No provider bill is raised: the provider was billed by the run the reading missed. No
  indirect-consumption line is written either.
- A lease already billed for this utility inside the current month is skipped (same rule as the
  processor), and
  a meter whose unbilled reading is **older** than a billed one is left alone: that consumption
  was already charged from the initial (or an earlier billed) reading up to the later one.
- Each billed reading is stamped `billed_at` and the lease `utility_billed_at`, so a second
  run finds nothing and the next regular run starts from the reading just billed.

## Utility billing history (row 156)

Every billing processed from the billing screen is saved as a **run**: one property, one utility,
the provider bill it was billed against, and what it billed, as it stood. `utility-billing-process`
returns the run's `utility_billing_id`, and each invoice it raised carries it.

- **History starts with the first billing after this release.** Earlier billings were never saved
  and are not rebuilt.
- Readings billed by the `utility-billing:bill-unbilled-readings` command (outside the screen)
  belong to no run, so they are not in the history.
- A billing that billed no lease and raised no provider bill is not kept.

**Who:** `view-utility-billing` or `create-utility-billing`, on the properties the user is
allocated to; otherwise `403`. Processing a billing still needs `create-utility-billing`, so show
**New Utility Billing** only to `create` holders. Each row also carries the provider bill's
`bill_date`, `bill_number` and `bill_tax_invoice_number` (each `null` when not entered on the
billing screen) - useful to tell two months of the same property apart, with `created_at` as the
fallback.

### List the runs

`GET /charges/utility-billings` — newest first, paginated (`per_page`).

| Filter | Kind |
| --- | --- |
| `filter[id]`, `filter[facility_id]`, `filter[utility_id]`, `filter[created_by_id]` | exact (comma list for several) |
| `filter[created_between]=2026-10-01,2026-10-31`, `filter[bill_date_between]=…` | date range |
| `filter[created_at]`, `filter[bill_date]` | partial (e.g. `2026-10`) |

Sorts: `id`, `created_at`, `bill_date`, `consumed`, `amount`, `tax`, `total` (`-` for descending).

```json
{
  "data": [
    {
      "id": 31,
      "facility": { "id": 61, "name": "Kahawa Side Tower" },
      "utility": { "id": 4, "name": "Water" },
      "distribution_method": "utility provider rate",
      "charge_rate": 25,
      "notes": "October water",
      "bill_id": 555,
      "bill_date": "2026-10-15",
      "bill_number": "NWC-88213",
      "total_bill_amount": 14500,
      "billed_leases": 2,
      "skipped_leases": 0,
      "total_previous": 2100,
      "total_current": 2600,
      "consumed": 500,
      "indirect_consumed": 0,
      "amount": 12500,
      "tax_amount": 2000,
      "total": 14500,
      "created_by": { "id": 1, "name": "Jane Admin" },
      "created_at": { "raw": "2026-10-20T08:15:00.000000Z", "formatted": "20 Oct, 2026", "diff": "2 hours ago" }
    }
  ],
  "filters": {
    "utilities": [{ "id": 6, "name": "Electricity" }, { "id": 4, "name": "Water" }],
    "facilities": [{ "id": 61, "name": "Kahawa Side Tower" }]
  },
  "totals": {
    "runs": 7, "total_previous": 14700, "total_current": 18200, "consumed": 3500,
    "indirect_consumed": 0, "amount": 87500, "tax_amount": 14000, "total": 101500
  },
  "links": { "...": "pagination" },
  "meta": { "...": "pagination" }
}
```

- One row per run: Property (`facility.name`), Utility, Total previous, Total current, Consumed,
  Amount, Tax (`tax_amount`), Total.
- `totals` is the **totals row**: every run the filters match, not just this page.
- `filters` are the options for the Utility and Property filters: the utilities and properties the
  caller's runs cover, whatever filter is applied, so a view-only user (who cannot read the
  lease-component settings) still gets them.
- `consumed` is what was billed off the meters; `indirect_consumed` is the indirect share billed on
  top (common areas, losses), when the utility bills it.
- **New Utility Billing** opens the existing billing screen. After processing, open the run with the
  returned `utility_billing_id`.

### Open a run

`GET /charges/utility-billings/{utilityBilling}`

```json
{
  "data": {
    "utility_billing": { "id": 31, "...": "the run, as in the list, plus tax and bill_upload" },
    "leases": [
      {
        "lease_id": 93,
        "label": "Lease #93 - Carol Wanjiru",
        "tenant": { "id": 224, "name": "Carol Wanjiru" },
        "invoice_id": 9001,
        "invoice_status": { "value": "unpaid", "color": "warning" },
        "previous": 1300,
        "current": 1650,
        "consumed": 350,
        "indirect_consumed": 0,
        "amount": 8750,
        "tax": 1400,
        "total": 10150,
        "meters": [
          {
            "meter_id": 12,
            "meter_number": "WM-A1",
            "meter_name": "WM-A1",
            "previous_reading": { "id": 70, "value": 1200, "read_at": "2026-09-20 09:00:00", "image": { "id": 501, "source_url": "…", "thumbnail_url": "…" } },
            "current_reading": { "id": 71, "value": 1500, "read_at": "2026-10-20 09:00:00", "image": { "id": 502, "source_url": "…", "thumbnail_url": "…" } },
            "consumed": 300,
            "amount": 7500,
            "tax": 1200,
            "total": 8700
          }
        ]
      }
    ],
    "totals": { "leases": 2, "previous": 2100, "current": 2600, "consumed": 500, "indirect_consumed": 0, "amount": 12500, "tax": 2000, "total": 14500 }
  }
}
```

- One row per tenant: `label` is "Lease# - Name". **View Invoice** opens `invoice_id`.
- `meters` are the expandable rows, one per meter the tenant was billed on. Show them expandable
  when there is more than one.
- **Reading photo:** each reading carries its `image` (`null` when none was taken). Both the previous
  and the current reading are sent; clicking a reading shows its photo.
- `totals` is the totals row for the run's tenants.
- A property the caller is not allocated to answers `403` "You are not allocated to this property."

## Frontend Error Handling

Apply shared rules in `docs/frontend/app/README.md`.
