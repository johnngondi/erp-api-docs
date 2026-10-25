# Billings & Collections Report API

Domain: `Property Management > Reports > Tenants`

Base route:

`/api/v1/app/{company}/property-management/reports`

The Billings & Collections report aggregates, per lease, everything a tenant was **billed** and
everything they **paid** for a period, with the opening (brought-forward) and closing
(carried-forward) balances. It is **read-only** and computed on the fly.

> **Read [the shared reports contract](./README.md) first.** This page only documents what is
> specific to this report — its extra filters, its buckets, and its balance conventions. The
> response envelope (`header` + a `report` array of buckets + `summary`), the bucket keys
> (`bucket`/`header`/`fields`/`items`/`summary`), the field keys (`format`, `type`, `weight`,
> `background_color`, …), the per-cell `{ value, color }` shape, the row `type` values, and the
> colour/format vocabulary all live in the README and apply here unchanged.

## Endpoints

- `GET /reports/tenants/billings-and-collections` — requires `view-billings-and-collections-report`
- `GET /reports/tenants/billings-and-collections/export` — requires
  `export-billings-and-collections-report`. Takes a required `format` (`excel` | `pdf`) plus the same
  filters as the report (below) and downloads the generated file — see
  [the shared contract](./README.md#permissions--export).
- Listed by [`GET /reports`](./README.md#discovering-reports--get-reports) — the discovery
  endpoint describes the report URL, permissions, filter descriptors, buckets and columns,
  summary keys and export formats, so none of it needs hardcoding.

## Get the report

`GET /api/v1/app/{company}/property-management/reports/tenants/billings-and-collections`

Returns a `header`, a `report` array of four or five buckets, and a report-level `summary`.

### Filters (query params)

All filters are **optional** and flat (not nested under `filter[...]`). The
[global filters](./README.md#global-filters) and the tenant reports' block / wing / floor lists
apply; this report adds three more:

| Param | Type | Meaning when omitted |
|---|---|---|
| `landlord_ids` | list of `users.id` | All landlords (global) |
| `facility_ids` | list of `facilities.id` | All properties (global) |
| `facility_type_id` | `facility_types.id` | All property types (global) |
| `properties_status` | enum `App\Enums\Status` | Any property status (global) |
| `period_from` | date `Y-m-d` | Start of the current month (global) |
| `period_to` | date `Y-m-d` | End of the current month (global) |
| `buckets` | list of this report's bucket ids | Every bucket (global) — see [Selecting buckets](./README.md#selecting-buckets--buckets) |
| `landlord_id` / `facility_id` | single id — **deprecated** aliases, merged into the lists | (global) |
| `leases_status` | enum `App\Enums\LeaseStatus` (`active`, `suspended`, `terminated`) | Any lease status |
| `expense_category_id` | `expense_categories.id` — the **Account** | All accounts |
| `lease_component_id` | `lease_components.id` — the **Component** | All components |
| `facility_block_ids` | list of `facility_blocks.id` | The whole property — only with exactly one property selected; see [below](#blocks-wings-and-floors) |
| `facility_wing_ids` | list of `facility_wings.id` | The whole property — likewise |
| `facility_floor_ids` | list of `facility_floors.id` | The whole property — likewise |

`landlord_ids`, `facility_type_id` and `properties_status` are resolved through the lease's property
(a lease has no landlord of its own). `expense_category_id` / `lease_component_id` narrow which
billing/collection lines are counted across **all** buckets.

**Lease rows are limited to the properties the caller is allocated to**, in every bucket. Before
this, only `summary_by_property` was: the lease buckets — and so `overview`'s Total row and
`data.summary` — could include leases of properties outside the caller's allocation. A user
allocated to only some properties may therefore see lower totals than before; those are the
correct ones.

Example:

```
GET …/reports/tenants/billings-and-collections?facility_ids[]=1&period_from=2026-07-01&period_to=2026-07-31
GET …/reports/tenants/billings-and-collections?facility_ids[]=1&facility_block_ids[]=11&buckets[]=overview
```

There is no pagination, sort, or include — a report is returned whole for the resolved period.

### Blocks, wings and floors

With exactly one property selected, `facility_block_ids`, `facility_wing_ids` and
`facility_floor_ids` narrow the report to part of it (several nodes are a union). Each lease counts
**whole** under the node of its **primary space** — the space of the item holding Rent, else
Service Charge, else the lowest-id spaced item. A lease with no space is in the property run
and out of every block / wing / floor run, so the block runs plus those leases add up to the property run.
Billings, collections and the opening balance all follow the lease, so a lease's
row is never split between nodes. `summary_by_property` is absent, as on any single-property run.
Rules, `422`s and the add-up guarantee:
[Splitting by block / wing / floor](./README.md#splitting-by-block--wing--floor). The pickers
are filled from [`GET …/reports/filter-options/structure`](./README.md#picking-blocks-wings-and-floors).

---

## The buckets

They always come back in this order. Five of them are fixed; `summary_by_property` is the one
bucket that varies with the filters — it leads the list **unless exactly one property is selected**, and
is dropped when exactly one property is selected (it would just restate the `overview` Total row).
Read the order from the `bucket` keys rather than assuming an index.

| `bucket` | `header.label` | One row per… | Columns |
|---|---|---|---|
| `summary_by_property` | Summary by Property | property — **unless exactly one property is selected** | `facility_id`, `name`, `balance_bf`, per-component `component_{id}_billings` (before `total_billings`), `total_billings`, `total_expected`, per-component `component_{id}_collections` (before `total_collections`), `total_collections`, `balance_cf` |
| `overview` | Summary | lease | `lease_id`, `landlord_identifier`, `name`, `balance_bf`, `total_billings`, `total_collections`, `balance_cf` |
| `overall_b_c_by_tenant` | Billings and Collections by Tenant | lease | `lease_id`, `landlord_identifier`, `name`, `balance_bf`, per-component `component_{id}_billings` (before `billings_tax`), `billings_tax`, `total_billings`, `total_expected`, per-component `component_{id}_collections` (before `collections_tax`), `collections_tax`, `total_collections`, `balance_cf` |
| `summary_by_account` | Summary by Account | lease | overview columns **plus** per-account `account_{id}_billings` (before `total_billings`) and `account_{id}_collections` (before `total_collections`) |
| `billings` | Billings | lease | `lease_id`, `landlord_identifier`, `name`, `balance_bf`, per-component `component_{id}` (before `total_amount`), `total_amount`, `total_tax`, `gross_total`, `balance_cf` |
| `collections` | Collections | lease | same shape as `billings`, for money received |

Every lease bucket carries **`landlord_identifier`** (label `Landlord ID`, `format: string`)
immediately after `lease_id`: what the landlord calls the tenant in their own system
(`leases.landlord_identifier`). It is `null` when the lease has none, and `null` on every Total row,
which names no lease.

Each bucket's `header` carries only the standard `label` — this report's buckets are sections of one
scope, not separate entities. Each bucket's `summary` holds that bucket's column totals (the same
numbers as its Total row).

**`summary_by_property`** lists **every property the filters match**, so a property with no leases —
or no activity in the period — still comes back as a zero row. Its rows are ordered by property
name. Its `Total` row reconciles with the `overview` bucket's `Total` row on `balance_bf`,
`total_billings`, `total_collections` and `balance_cf`. Its per-component columns are the first in
this API to ship `visible: false` alongside `togglable: true` — the breakdown is there for whoever
opens the column picker, not for the default view. Because both sides share one row, those keys
carry a side suffix (`component_{id}_billings` / `component_{id}_collections`) rather than the bare
`component_{id}` the `billings` / `collections` buckets use.

**`overall_b_c_by_tenant`** puts each lease's billings and collections side by side. Its
per-component cells hold the amount **net of tax**, and each side's tax gets its own column
(`billings_tax` / `collections_tax`, labelled VAT), so on each side the component cells plus the
VAT cell add up to the side's total (`total_billings` = Billed, `total_collections` = Paid).
`total_expected` (Total Due) is `balance_bf + total_billings`. Its component columns are visible by
default and, like `summary_by_property`, carry the side suffix.

**Accounts vs components:** an *account* is an expense category; a *component* is a lease component
(each component belongs to one account). `summary_by_account` rolls up to accounts; `billings` /
`collections` break down to individual components. The per-account and per-component columns are the
[dynamic columns](./README.md#fields--column-definitions) described in the contract —
`togglable: true`, each carrying its `account_id` / `component_id`.

### Balance convention

- `balance_bf` (brought forward) — activity strictly **before** `period_from`.
  - `summary` / `summary_by_account` / `summary_by_property`: prior billings **−** prior collections.
  - `billings` bucket: prior **billings** only. `collections` bucket: prior **collections** only.
- `total_expected` (`summary_by_property` and `overall_b_c_by_tenant`) — `balance_bf + total_billings`:
  everything owed in the period, arrears included.
- `balance_cf` (carried forward):
  - `summary` / `summary_by_account`: `balance_bf + total_billings − total_collections`.
  - `summary_by_property` / `overall_b_c_by_tenant`: `total_expected − total_collections` (the
    same arithmetic).
  - `billings` / `collections`: `balance_bf + gross_total`.
- In the billings/collections buckets, the per-component cells sum to `gross_total`, and
  `total_amount + total_tax = gross_total`.
- Only **confirmed** collections are counted; cancelled collections are excluded.

### This report's presentation choices

Following the shared vocabulary, this report sets:

- `total_billings` / `total_collections` / `total_amount` / `total_tax` columns → `type: subtotal`,
  `weight: font-medium`.
- `gross_total` / `balance_cf` columns → `type: grosstotal`, `weight: font-bold`.
- Column tints (`background_color`), so billings and collections read as two blocks:
  - Billing columns (per-account `account_{id}_billings`, per-component `component_{id}_billings`,
    `billings_tax`, and the `billings` bucket's `component_{id}` / `total_amount` / `total_tax`) →
    `info`.
  - Collection columns (per-account `account_{id}_collections`, per-component
    `component_{id}_collections`, `collections_tax`, and the `collections` bucket's `component_{id}` / `total_amount` /
    `total_tax`) → `success`.
  - Totals (`total_billings`, `total_collections`, `gross_total`) → `danger`.
  - `balance_bf`, `total_expected` and `balance_cf` → `none`.
- Each bucket's **Total** row → `type: subtotal`, `background_color: secondary`.
- A negative `balance_bf` / `balance_cf` **cell** → `color: danger` (arrears); otherwise `none`.

---

## Sample response

Trimmed to the `header`, one full bucket, and the `summary`. The other buckets follow the same
five-key shape (see the bucket table above for their columns) and the same cell shape. This call
passed `facility_ids[]=1`, so `summary_by_property` is absent and `overview` leads the array.

```json
{
  "data": {
    "header": {
      "property": {
        "id": 1,
        "name": "Riverside Mixed-Use Plaza",
        "currency": { "code": "KES", "name": "Kenya Shilling" }
      },
      "properties": [{ "id": 1, "name": "Riverside Mixed-Use Plaza" }],
      "landlord": null,
      "landlords": [],
      "blocks": [],
      "wings": [],
      "floors": [],
      "period": { "from": "2026-07-01", "to": "2026-07-31" },
      "filters": {
        "landlord_id": null,
        "facility_id": null,
        "facility_type_id": null,
        "properties_status": null,
        "leases_status": null,
        "expense_category_id": null,
        "lease_component_id": null,
        "landlord_ids": [],
        "facility_ids": [1],
        "facility_block_ids": [],
        "facility_wing_ids": [],
        "facility_floor_ids": [],
        "buckets": []
      },
      "generated_at": "2026-07-15T14:52:57+03:00"
    },
    "report": [
      {
        "bucket": "overview",
        "header": { "label": "Summary" },
        "fields": [
          { "label": "Lease", "key": "lease_id", "format": "id", "type": "normal", "weight": "font-normal", "background_color": "none", "alignment": "left", "visible": true, "togglable": false },
          { "label": "Landlord ID", "key": "landlord_identifier", "format": "string", "type": "normal", "weight": "font-normal", "background_color": "none", "alignment": "left", "visible": true, "togglable": false },
          { "label": "Tenant", "key": "name", "format": "string", "type": "normal", "weight": "font-normal", "background_color": "none", "alignment": "left", "visible": true, "togglable": false },
          { "label": "Balance B/F", "key": "balance_bf", "format": "money", "type": "normal", "weight": "font-normal", "background_color": "none", "alignment": "right", "visible": true, "togglable": true },
          { "label": "Billings", "key": "total_billings", "format": "money", "type": "subtotal", "weight": "font-medium", "background_color": "danger", "alignment": "right", "visible": true, "togglable": false },
          { "label": "Collections", "key": "total_collections", "format": "money", "type": "subtotal", "weight": "font-medium", "background_color": "danger", "alignment": "right", "visible": true, "togglable": false },
          { "label": "Balance C/F", "key": "balance_cf", "format": "money", "type": "grosstotal", "weight": "font-bold", "background_color": "none", "alignment": "right", "visible": true, "togglable": false }
        ],
        "items": [
          {
            "lease_id": { "value": 1 },
            "landlord_identifier": { "value": "LL-0042" },
            "name": { "value": "Super Admin" },
            "balance_bf": { "value": 0 },
            "total_billings": { "value": 34800 },
            "total_collections": { "value": 30000 },
            "balance_cf": { "value": 4800 },
            "type": "normal"
          },
          {
            "lease_id": { "value": null },
            "landlord_identifier": { "value": null },
            "name": { "value": "Total" },
            "balance_bf": { "value": 0 },
            "total_billings": { "value": 34800 },
            "total_collections": { "value": 30000 },
            "balance_cf": { "value": 4800 },
            "type": "subtotal",
            "background_color": "secondary"
          }
        ],
        "summary": {
          "balance_bf": 0,
          "total_billings": 34800,
          "total_collections": 30000,
          "balance_cf": 4800
        }
      },
      { "bucket": "summary_by_account", "header": { "label": "Summary by Account" }, "fields": [ "…" ], "items": [ "…" ], "summary": { "…": "…" } },
      { "bucket": "billings",           "header": { "label": "Billings" },           "fields": [ "…" ], "items": [ "…" ], "summary": { "…": "…" } },
      { "bucket": "collections",        "header": { "label": "Collections" },        "fields": [ "…" ], "items": [ "…" ], "summary": { "…": "…" } }
    ],
    "summary": {
      "lease_count": 1,
      "total_balance_bf": 0,
      "total_billings": 34800,
      "total_collections": 30000,
      "total_balance_cf": 4800
    }
  }
}
```

Example of the dynamic-column buckets (from the same call) — note the injected `account_2_billings` /
`component_2` columns come back `togglable: true` and carry their entity ref:

```jsonc
// A per-account column in the `summary_by_account` bucket's fields — carries account_id
{ "label": "Rent Billings", "key": "account_1_billings", "format": "money", "type": "normal",
  "weight": "font-normal", "background_color": "info", "alignment": "right", "togglable": true,
  "visible": true, "account_id": 1 }

// A per-component column in the `billings` bucket's fields — carries component_id
{ "label": "Rent", "key": "component_1", "format": "money", "type": "normal",
  "weight": "font-normal", "background_color": "info", "alignment": "right", "togglable": true,
  "visible": true, "component_id": 1 }

// The same idea in `summary_by_property`, but hidden by default — don't render it until the
// user turns it on in the column picker
{ "label": "Rent Billings", "key": "component_1_billings", "format": "money", "type": "normal",
  "weight": "font-normal", "background_color": "info", "alignment": "right", "togglable": true,
  "visible": false, "component_id": 1 }
```

Column order (keys) for the dynamic buckets, for reference:

```jsonc
// the `summary_by_property` bucket's fields (omitted entirely when exactly one property is selected)
[ "facility_id", "name", "balance_bf",
  "component_{id}_billings",    // ← per-component, injected before total_billings, visible: false
  "total_billings", "total_expected",
  "component_{id}_collections", // ← per-component, injected before total_collections, visible: false
  "total_collections", "balance_cf" ]

// the `overall_b_c_by_tenant` bucket's fields
[ "lease_id", "landlord_identifier", "name", "balance_bf",
  "component_{id}_billings",    // ← per-component (net of tax), injected before billings_tax
  "billings_tax", "total_billings", "total_expected",
  "component_{id}_collections", // ← per-component (net of tax), injected before collections_tax
  "collections_tax", "total_collections", "balance_cf" ]

// the `summary_by_account` bucket's fields
[ "lease_id", "landlord_identifier", "name", "balance_bf",
  "account_{id}_billings",   // ← per-account, injected before total_billings
  "total_billings",
  "account_{id}_collections",// ← per-account, injected before total_collections
  "total_collections", "balance_cf" ]

// the `billings` / `collections` buckets' fields
[ "lease_id", "landlord_identifier", "name", "balance_bf",
  "component_{id}",          // ← per-component, injected before total_amount
  "total_amount", "total_tax", "gross_total", "balance_cf" ]
```

For how to render any of this generically (columns from `fields`, cell `value`/`color`, row and
column tints, subtotal/grosstotal emphasis), see the
[render guide in the contract](./README.md#how-to-render-report-template).
