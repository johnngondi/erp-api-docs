# Management Fee Report API

Domain: `Property Management > Reports > Landlords`

Base route:

`/api/v1/app/{company}/property-management/reports`

The Management Fee report shows what the agent **earned managing each property** over a period,
calculated against the terms of each management contract. It is what the agent reports back to each
owner, so it is **grouped by landlord**, with one row per property. It is **read-only** and computed
on the fly.

> **Read [the shared reports contract](./README.md) first.** This page only documents what is specific
> to this report — its extra filter, its landlord buckets, its dynamic role columns, and how the fee is
> calculated. The response envelope (`header` + a `report` array of buckets + `summary`), the bucket
> keys, the field keys, the per-cell `{ value, color }` shape, and the colour/format vocabulary all live
> in the README and apply here unchanged.

## Endpoints

- `GET /reports/landlords/management-fee` — requires `view-management-fee-report`
- `GET /reports/landlords/management-fee/export` — requires `export-management-fee-report`. Takes a
  required `format` (`excel` | `pdf`) plus the same filters as the report (below) and downloads the
  generated file — see [the shared contract](./README.md#permissions--export). Any other `format`, or
  none, returns `422` with a `format` error.
- Listed by [`GET /reports`](./README.md#discovering-reports--get-reports) — the discovery
  endpoint describes the report URL, permissions, filter descriptors, buckets and columns,
  summary keys and export formats, so none of it needs hardcoding.

The two permissions are **independent**. Being able to view the report does not grant exporting it,
and being able to export it does not grant viewing it.

## Get the report

`GET /api/v1/app/{company}/property-management/reports/landlords/management-fee`

### Filters (query params)

All filters are **optional** and flat (not nested under `filter[...]`). The six
[global filters](./README.md#global-filters) apply; this report adds one:

| Param | Type | Meaning when omitted |
|---|---|---|
| `landlord_id` | `users.id` | All landlords (global) |
| `facility_id` | `facilities.id` | All properties (global) |
| `facility_type_id` | `facility_types.id` | All property types (global) |
| `properties_status` | enum `App\Enums\Status` | Any property status (global) |
| `period_from` | date `Y-m-d` | Start of the current month (global) |
| `period_to` | date `Y-m-d` | End of the current month (global) |
| `management_contract_id` | `facility_management_contracts.id` | All contracts — limits the report to the properties that one contract covers |

`header.filters` echoes **every** filter above, `null` included. `header.period` echoes the resolved
window, so an omitted period comes back as the current month there while `header.filters.period_from`
and `period_to` stay `null`.

Example:

```
GET …/reports/landlords/management-fee?landlord_id=4&period_from=2026-07-01&period_to=2026-07-31
GET …/reports/landlords/management-fee/export?landlord_id=4&period_from=2026-07-01&period_to=2026-07-31&format=pdf
```

### Currency

This report takes **no currency filter** and converts nothing: every figure is in its property's own
reporting currency. For a single property (`facility_id` set) the currency sits on
`header.property.currency`. Otherwise `header.currency` names the currency the properties in scope
share, and is `null` when they report in different currencies.

## Buckets

| `bucket` | When present | `header.label` | Rows |
|---|---|---|---|
| `overall` | always, first | `Portfolio Total` | one `subtotal` row, `All landlords` |
| `landlord-{id}` | one per landlord in scope, **sorted by landlord name** | the landlord's name | one row per property, then a `subtotal` row |

- **Landlord is the grouping and property is the row.** A landlord's bucket holds every one of their
  properties in scope and nothing else.
- **Parking properties are rows of their own.** There is no parking property type; a parking
  property is simply another property, and it is never merged into the building it serves.
- Each `landlord-{id}` bucket `header` adds `landlord` — `{ id, name }`. A property with no landlord
  falls under `landlord-0`, labelled `Unassigned`, with `landlord: null`.
- Rows within a bucket are sorted by property name.

## Columns

Every bucket shares one column set, in this order:

| Column | `key` | `format` | Meaning |
|---|---|---|---|
| Property | `property` | string | The property's name |
| Type | `type_` | string | The property's type |
| SqFt/Units | `sqft_units` | money | The lettable measure — see [occupancy](#occupancy) |
| Occupancy Rate % | `occupancy_rate` | string | Occupied ÷ lettable, rendered `0.00 %` |
| Balance B/F | `balance_bf` | money | Tenant balance outstanding at the start of the period |
| Billing | `billing` | money | Billed to tenants within the period |
| Total Expected | `total_expected` | money | `balance_bf + billing` |
| Collections (for the Period) | `collections` | money | Confirmed collections within the period |
| Balance C/F | `balance_cf` | money | `total_expected − collections`; `danger` when positive |
| % Collection | `collection_percent` | string | `collections ÷ total_expected`, rendered `0.00 %` |
| Management fee % | `management_fee_percent` | string | The contract rate, e.g. `2.5%`, or `—` for a flat fee |
| Management Fees | `management_fees` | money | The fee earned, **gross of VAT** — see [the fee](#the-fee) |
| *(dynamic)* | `role_{roleId}` | string | One column per property role — see [role columns](#role-columns) |

> **The key is `type_`, not `type`.** Every row carries its own `type` key (`normal` / `subtotal`), its
> visual role under the shared contract. A cell called `type` would collide with it, so the property
> type column is keyed `type_`. Read the column from `fields` as always and nothing special is needed.

Money values are rounded to two decimal places, so a stored `39932.31999999999` is returned as
`39932.32`.

### The fee

**The fee is charged on collections, not billings.** An agent earns on money actually received.

For a percentage contract:

```
Management Fees = fee base × fee % × (1 + VAT rate)
```

- **It is gross of VAT.** There is no separate VAT column: a stated `2.5%` is an effective `2.9%` at
  the standard rate. The figure in `management_fees` already includes the VAT.
- **The VAT rate is the company's default tax**, never a fixed 16%. A company that is **not VAT
  registered** earns the bare `fee base × fee %`, with no uplift.
- **The remittance charges the same fee.** Computing a remittance and the management fee bill it
  raises apply this exact rule, including no VAT for a company that is not VAT registered, so the
  fee on this report matches what is billed for the same property and period.
- **The fee base is the period's collections** unless the contract narrows it. A contract can limit
  the fee to certain lease components, and can charge on collections net of their own tax. Where it
  does, `management_fees` is calculated on that narrower base while `collections` still shows
  everything received, so the two will not multiply out exactly.
- A base that nets negative earns no fee rather than a negative one.

A **flat-fee** contract stays in the report. Its `management_fee_percent` is `—`, a rendered value
rather than an empty cell. Its `management_fees` is the flat amount, VAT-uplifted on the same rule, and
it counts toward every total. A property with no contract, or a contract whose fee type is `none`,
shows `—` and earns `0`.

### Occupancy

`sqft_units` is whichever measure the property is actually let by:

- A property that declares a **lettable area** is measured by **area**. `sqft_units` is that area and
  occupancy is the area of its taken spaces over it, capped at `100.00 %`. Parking bays and signage
  positions never count toward area.
- A property with **no lettable area** — a parking property above all — is measured by **space
  count**. `sqft_units` is its number of spaces and occupancy is taken spaces over that count.

A space counts as taken when **a lease covered it at any point in the period**. A lease runs from its
start until it is terminated, so a lease terminated partway through the period still counts, and one
terminated before the period began does not.

`occupancy_rate` and `collection_percent` **always resolve**: they read `0.00 %` where there is nothing
to divide by, never an empty cell or a missing key.

### Totals

Each bucket closes on a `subtotal` row with `background_color: secondary`:

- `property` spans the four identifying columns (`col_span: 4`), reading `Total` in a landlord bucket
  and `All landlords` in `overall`. The covered keys are absent.
- **Every money column totals.**
- **Percentages are recomputed from the bucket's own totals, never averaged across its rows.**
  `collection_percent` on the subtotal is `Σ collections ÷ Σ total_expected`. Averaging would let a
  small property distort the figure: a large property collecting 90% and a tiny one collecting 10%
  total close to 90%, not 50%.
- **`management_fee_percent` has no subtotal.** Flat-fee and percentage properties mix in that column,
  so no single rate describes the group and the key is omitted. `occupancy_rate` has none either, and
  the role columns are absent from subtotal rows.

### Role columns

After `management_fees` comes one column per **property role** — Property Manager, Agency and so on:

- `key` is `role_{roleId}` and the field carries an explicit `role_id` reference, so there is no need
  to parse the key.
- `label` is the role's name, `format` is `string`, and **`togglable` is `true`**.
- The value is the name of whoever holds that role on the property. Where several people hold the
  same role on one property, their names are **joined into one cell**, comma-separated and sorted.
- The column set is the **union of roles across the whole report**, ordered by role name, and every
  bucket declares the same set, `overall` included. A property with no holder for a role another
  property has simply omits that key, which renders as an empty cell.

## Summary

Each bucket's `summary`:

| Key | Meaning |
|---|---|
| `total_collections` | Collections in the bucket |
| `total_management_fees` | Management fees in the bucket, gross of VAT |
| `property_count` | Properties in the bucket |

The report-level `data.summary` carries `total_collections` and `total_management_fees` for the whole
scope, plus `landlord_count`.

## Example response

```jsonc
{
  "data": {
    "header": {
      "property": null,
      "period": { "from": "2026-07-01", "to": "2026-07-31" },
      "filters": {
        "landlord_id": 4, "facility_id": null, "facility_type_id": null,
        "properties_status": null, "period_from": "2026-07-01", "period_to": "2026-07-31",
        "management_contract_id": null
      },
      "generated_at": "2026-08-17T09:12:44+00:00",
      "currency": { "code": "KES", "name": "Kenyan Shilling" }
    },
    "report": [
      {
        "bucket": "overall",
        "header": { "label": "Portfolio Total" },
        "fields": [ /* the columns above, then the role columns */ ],
        "items": [
          {
            "property": { "value": "All landlords", "col_span": 4 },
            "balance_bf": { "value": 3067265.00 },
            "billing": { "value": 11535472.85 },
            "total_expected": { "value": 14602737.85 },
            "collections": { "value": 5571489.44 },
            "balance_cf": { "value": 9031248.41 },
            "collection_percent": { "value": "38.15 %" },
            "management_fees": { "value": 161573.19 },
            "type": "subtotal",
            "background_color": "secondary"
          }
        ],
        "summary": { "total_collections": 5571489.44, "total_management_fees": 161573.19, "property_count": 2 }
      },
      {
        "bucket": "landlord-4",
        "header": { "label": "Clementine Ndibo Holdings", "landlord": { "id": 4, "name": "Clementine Ndibo Holdings" } },
        "fields": [
          { "label": "Property", "key": "property", "format": "string", "type": "normal", "weight": "font-normal", "background_color": "none", "alignment": "left", "visible": true, "togglable": false },
          { "label": "Type", "key": "type_", "format": "string", "type": "normal", "weight": "font-normal", "background_color": "none", "alignment": "left", "visible": true, "togglable": false },
          /* …sqft_units through management_fee_percent… */
          { "label": "Management Fees", "key": "management_fees", "format": "money", "type": "subtotal", "weight": "font-medium", "background_color": "none", "alignment": "right", "visible": true, "togglable": false },
          { "label": "Property Manager", "key": "role_3", "format": "string", "type": "normal", "weight": "font-normal", "background_color": "none", "alignment": "left", "visible": true, "togglable": true, "role_id": 3 }
        ],
        "items": [
          {
            "property": { "value": "KAHAWA HOUSE" },
            "type_": { "value": "Commercial" },
            "sqft_units": { "value": 39932 },
            "occupancy_rate": { "value": "93.55 %" },
            "balance_bf": { "value": 3067265.00 },
            "billing": { "value": 11100472.85 },
            "total_expected": { "value": 14167737.85 },
            "collections": { "value": 5426489.44 },
            "balance_cf": { "value": 8741248.41, "color": "danger" },
            "collection_percent": { "value": "38.30 %" },
            "management_fee_percent": { "value": "2.5%" },
            "management_fees": { "value": 157368.19 },
            "role_3": { "value": "Clementine Ndibo" },
            "type": "normal"
          },
          {
            "property": { "value": "Kahawa House - Parking" },
            "type_": { "value": "Commercial" },
            "sqft_units": { "value": 0 },
            "occupancy_rate": { "value": "0.00 %" },
            "balance_bf": { "value": 0 },
            "billing": { "value": 435000.00 },
            "total_expected": { "value": 435000.00 },
            "collections": { "value": 145000.00 },
            "balance_cf": { "value": 290000.00, "color": "danger" },
            "collection_percent": { "value": "33.33 %" },
            "management_fee_percent": { "value": "—" },
            "management_fees": { "value": 4205.00 },
            "role_3": { "value": "Clementine Ndibo" },
            "type": "normal"
          },
          {
            "property": { "value": "Total", "col_span": 4 },
            "balance_bf": { "value": 3067265.00 },
            "billing": { "value": 11535472.85 },
            "total_expected": { "value": 14602737.85 },
            "collections": { "value": 5571489.44 },
            "balance_cf": { "value": 9031248.41 },
            "collection_percent": { "value": "38.15 %" },
            "management_fees": { "value": 161573.19 },
            "type": "subtotal",
            "background_color": "secondary"
          }
        ],
        "summary": { "total_collections": 5571489.44, "total_management_fees": 161573.19, "property_count": 2 }
      }
    ],
    "summary": { "total_collections": 5571489.44, "total_management_fees": 161573.19, "landlord_count": 1 }
  }
}
```

Reading the example: `management_fees` on the first row is `5426489.44 × 2.5% × 1.16`. The parking row
is a flat fee, so it shows `—` for its rate yet still adds its `4205.00` to the total. The subtotal's
`collection_percent` is `5571489.44 ÷ 14602737.85`, not the average of `38.30` and `33.33`.
