# Billings & Collections Report API

Domain: `Property Management > Reports > Tenants`

Base route:

`/api/v1/app/{company}/property-management/reports`

The Billings & Collections report aggregates, per lease, everything a tenant was **billed** and
everything they **paid** for a period, with the opening (brought-forward) and closing
(carried-forward) balances. It is **read-only** and computed on the fly.

> **Read [the shared reports contract](./README.md) first.** This page only documents what is
> specific to this report — its extra filters, its four buckets, and its balance conventions. The
> response envelope (`header` + `report.<bucket>` of `fields`/`items` + `summary`), the field keys
> (`format`, `type`, `weight`, `background_color`, …), the per-cell `{ value, color }` shape, the
> row `type` values, and the colour/format vocabulary all live in the README and apply here
> unchanged.

## Endpoints

- `GET /reports/tenants/billings-and-collections`

## Get the report

`GET /api/v1/app/{company}/property-management/reports/tenants/billings-and-collections`

Returns a `header`, a `report` object holding four buckets, and a report-level `summary`.

### Filters (query params)

All filters are **optional** and flat (not nested under `filter[...]`). The six
[global filters](./README.md#global-filters) apply; this report adds three more:

| Param | Type | Meaning when omitted |
|---|---|---|
| `landlord_id` | `users.id` | All landlords (global) |
| `facility_id` | `facilities.id` | All properties (global) |
| `facility_type_id` | `facility_types.id` | All property types (global) |
| `properties_status` | enum `App\Enums\Status` | Any property status (global) |
| `period_from` | date `Y-m-d` | Start of the current month (global) |
| `period_to` | date `Y-m-d` | End of the current month (global) |
| `leases_status` | enum `App\Enums\LeaseStatus` (`active`, `suspended`, `terminated`) | Any lease status |
| `expense_category_id` | `expense_categories.id` — the **Account** | All accounts |
| `lease_component_id` | `lease_components.id` — the **Component** | All components |

`landlord_id`, `facility_type_id` and `properties_status` are resolved through the lease's property
(a lease has no landlord of its own). `expense_category_id` / `lease_component_id` narrow which
billing/collection lines are counted across **all** buckets.

Example:

```
GET …/reports/tenants/billings-and-collections?facility_id=1&period_from=2026-07-01&period_to=2026-07-31
```

There is no pagination, sort, or include — a report is returned whole for the resolved period.

---

## The four buckets

| Bucket | One row per… | Columns |
|---|---|---|
| `summary` | lease | `lease_id`, `name`, `balance_bf`, `total_billings`, `total_collections`, `balance_cf` |
| `summary_by_account` | lease | summary columns **plus** per-account `account_{id}_billings` (before `total_billings`) and `account_{id}_collections` (before `total_collections`) |
| `billings` | lease | `lease_id`, `name`, `balance_bf`, per-component `component_{id}` (before `total_amount`), `total_amount`, `total_tax`, `gross_total`, `balance_cf` |
| `collections` | lease | same shape as `billings`, for money received |

**Accounts vs components:** an *account* is an expense category; a *component* is a lease component
(each component belongs to one account). `summary_by_account` rolls up to accounts; `billings` /
`collections` break down to individual components. The per-account and per-component columns are the
[dynamic columns](./README.md#reportbucketfields--column-definitions) described in the contract —
`togglable: true`, each carrying its `account_id` / `component_id`.

### Balance convention

- `balance_bf` (brought forward) — activity strictly **before** `period_from`.
  - `summary` / `summary_by_account`: prior billings **−** prior collections.
  - `billings` bucket: prior **billings** only. `collections` bucket: prior **collections** only.
- `balance_cf` (carried forward):
  - `summary` / `summary_by_account`: `balance_bf + total_billings − total_collections`.
  - `billings` / `collections`: `balance_bf + gross_total`.
- In the billings/collections buckets, the per-component cells sum to `gross_total`, and
  `total_amount + total_tax = gross_total`.
- Only **confirmed** collections are counted; cancelled collections are excluded.

### This report's presentation choices

Following the shared vocabulary, this report sets:

- `total_billings` / `total_collections` / `total_amount` / `total_tax` columns → `type: subtotal`,
  `weight: font-medium`.
- `gross_total` / `balance_cf` columns → `type: grosstotal`, `weight: font-bold`.
- Each bucket's **Total** row → `type: subtotal`, `background_color: secondary`.
- A negative `balance_bf` / `balance_cf` **cell** → `color: danger` (arrears); otherwise `none`.

---

## Sample response

Trimmed to the `header`, one full bucket, and the `summary`. The other buckets follow the same
`{ fields, items }` shape (see the bucket table above for their columns) and the same cell shape.

```json
{
  "data": {
    "header": {
      "property": {
        "id": 1,
        "name": "Riverside Mixed-Use Plaza",
        "currency": { "code": "KES", "name": "Kenya Shilling" }
      },
      "period": { "from": "2026-07-01", "to": "2026-07-31" },
      "filters": {
        "landlord_id": null,
        "facility_id": 1,
        "facility_type_id": null,
        "properties_status": null,
        "leases_status": null,
        "expense_category_id": null,
        "lease_component_id": null
      },
      "generated_at": "2026-07-15T14:52:57+03:00"
    },
    "report": {
      "summary": {
        "fields": [
          { "label": "Lease", "key": "lease_id", "format": "integer", "type": "normal", "weight": "font-normal", "background_color": "none", "alignment": "left", "visible": true, "togglable": false },
          { "label": "Tenant", "key": "name", "format": "string", "type": "normal", "weight": "font-normal", "background_color": "none", "alignment": "left", "visible": true, "togglable": false },
          { "label": "Balance B/F", "key": "balance_bf", "format": "money", "type": "normal", "weight": "font-normal", "background_color": "none", "alignment": "right", "visible": true, "togglable": true },
          { "label": "Billings", "key": "total_billings", "format": "money", "type": "subtotal", "weight": "font-medium", "background_color": "none", "alignment": "right", "visible": true, "togglable": false },
          { "label": "Collections", "key": "total_collections", "format": "money", "type": "subtotal", "weight": "font-medium", "background_color": "none", "alignment": "right", "visible": true, "togglable": false },
          { "label": "Balance C/F", "key": "balance_cf", "format": "money", "type": "grosstotal", "weight": "font-bold", "background_color": "none", "alignment": "right", "visible": true, "togglable": false }
        ],
        "items": [
          {
            "lease_id": { "value": 1 },
            "name": { "value": "Super Admin" },
            "balance_bf": { "value": 0 },
            "total_billings": { "value": 34800 },
            "total_collections": { "value": 30000 },
            "balance_cf": { "value": 4800 },
            "type": "normal"
          },
          {
            "lease_id": { "value": null },
            "name": { "value": "Total" },
            "balance_bf": { "value": 0 },
            "total_billings": { "value": 34800 },
            "total_collections": { "value": 30000 },
            "balance_cf": { "value": 4800 },
            "type": "subtotal",
            "background_color": "secondary"
          }
        ]
      },
      "summary_by_account": { "fields": [ "…" ], "items": [ "…" ] },
      "billings":           { "fields": [ "…" ], "items": [ "…" ] },
      "collections":        { "fields": [ "…" ], "items": [ "…" ] }
    },
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
// A per-account column in report.summary_by_account.fields — carries account_id
{ "label": "Rent Billings", "key": "account_1_billings", "format": "money", "type": "normal",
  "weight": "font-normal", "background_color": "none", "alignment": "right", "togglable": true,
  "visible": true, "account_id": 1 }

// A per-component column in report.billings.fields — carries component_id
{ "label": "Rent", "key": "component_1", "format": "money", "type": "normal",
  "weight": "font-normal", "background_color": "none", "alignment": "right", "togglable": true,
  "visible": true, "component_id": 1 }
```

Column order (keys) for the dynamic buckets, for reference:

```jsonc
// report.summary_by_account.fields
[ "lease_id", "name", "balance_bf",
  "account_{id}_billings",   // ← per-account, injected before total_billings
  "total_billings",
  "account_{id}_collections",// ← per-account, injected before total_collections
  "total_collections", "balance_cf" ]

// report.billings.fields / report.collections.fields
[ "lease_id", "name", "balance_bf",
  "component_{id}",          // ← per-component, injected before total_amount
  "total_amount", "total_tax", "gross_total", "balance_cf" ]
```

For how to render any of this generically (columns from `fields`, cell `value`/`color`, row and
column tints, subtotal/grosstotal emphasis), see the
[render guide in the contract](./README.md#how-to-render-report-template).
