# Occupancy Trend Report API

Domain: `Property Management > Reports > Tenants`

Base route:

`/api/v1/app/{company}/property-management/reports`

Occupancy Trend answers **how full each property was over time**. Each row is one period of the
chosen cycle, measured **at the end of that period**: total spaces, occupied, vacant, the occupancy
rate and the occupied area. It is **read-only** and computed on the fly from the leases, which carry
their history back to 2010.

> **Read [the shared reports contract](./README.md) first.** This page documents what is specific to
> this report. The response envelope, the bucket keys, the field keys, the per-cell `{ value, color }`
> shape and the colour vocabulary live in the README and apply here unchanged.

## Endpoints

- `GET /reports/tenants/occupancy-trend` — requires `view-occupancy-trend-report`
- `GET /reports/tenants/occupancy-trend/export` — requires `export-occupancy-trend-report`. Takes a
  required `format` (`excel` | `pdf`); any other value, or none, returns `422` with a `format` error.
- Listed by [`GET /reports`](./README.md#discovering-reports--get-reports).

The two permissions are **independent**, and neither is shared with any other report.

## What is counted

- **Total spaces** are the property's spaces **as they are today** (not deleted), reached through any
  of the four floor shapes (facility, block, wing, wing of a block), parking bays and signage
  positions included. Spaces are not versioned, so a space added in 2025 also counts in 2019.
- A space is **occupied at a moment** when a lease item (`lease_items.facility_space_id`) ties it to a
  lease that had started by then and was not yet terminated:
  `start_at <= moment` and (`terminated_at` is null or `terminated_at > moment`). A lease runs until
  it is terminated, never until `end_at`, as in Tenancy Schedule and Property Status.
- Leases in status **`pending` or `rejected` never occupy a space**: a pending lease has not been
  approved yet and a rejected one never will be. Every other status counts.
- The moment is the **end of each period** (its last second), clamped to `period_to`.
- **Occupied area** is the summed `size` of the occupied spaces, **parking bays and signage
  positions excluded**, since those are let by the unit and not by area (the basis Management Fee and
  Tenancy Schedule use for area).

This is a period-end measure, so it can differ from [Summary Occupancy](./summary-occupancy.md),
which reads each space's current `status`, and from Property Status, which counts a space occupied
if a lease covered it at any point during the window.

## Filters

The [global filters](./README.md#global-filters) plus these:

| Param | Type | Meaning when omitted |
|---|---|---|
| [global filters](./README.md#global-filters) — `landlord_ids`, `facility_ids`, `facility_type_id`, `properties_status`, period | see the shared contract | all / current month |
| `buckets` | list of this report's bucket ids (`all-properties`, `property-{id}`) | Every bucket — see [Selecting buckets](./README.md#selecting-buckets--buckets) |
| `report_cycle` | `monthly` \| `quarterly` \| `annually` | **`monthly`** |
| `facility_block_ids` / `facility_wing_ids` / `facility_floor_ids` | lists of `facility_blocks.id` / `facility_wings.id` / `facility_floors.id` | The whole property — only with exactly one property selected; see [below](#blocks-wings-and-floors) |

The deprecated `landlord_id` / `facility_id` are still accepted and merged into the lists (see
[the shared contract](./README.md#the-deprecated-landlord_id--facility_id)).

`weekly` returns `422`. `header.filters` echoes every filter, `null` included, the lists as lists
(`[]` when empty).

```
GET …/reports/tenants/occupancy-trend?period_from=2015-01-01&period_to=2026-12-31&report_cycle=annually&landlord_ids[]=4&landlord_ids[]=7
```

### Blocks, wings and floors

With exactly one property selected, `facility_block_ids`, `facility_wing_ids` and
`facility_floor_ids` narrow the report to part of it (several nodes are a union; a node inside a
selected node refines it). Like [Summary Occupancy](./summary-occupancy.md#blocks-wings-and-floors),
this report counts **spaces, not leases**: it keeps the spaces whose floor lies under a selected node
— exact by construction, with no primary-space rule. A kept space is occupied by any lease that holds
it, so a lease spanning two blocks occupies its space in each. Every space sits on a floor, so the
block runs plus the floors under no block add up to the property, period by period.
Rules, `422`s and the add-up guarantee:
[Splitting by block / wing / floor](./README.md#splitting-by-block--wing--floor). The pickers
are filled from [`GET …/reports/filter-options/structure`](./README.md#picking-blocks-wings-and-floors).

## Buckets

Always in this order:

| `bucket` | `header` | Rows |
|---|---|---|
| `all-properties` | `label: "All Properties"` | one row per period, every property in scope added together |
| `property-{id}` | `label` (the property name), `property { id, name }` | one row per period for that property |

Property buckets follow ordered by property name. `facility_ids` narrows the scope but does **not**
change the bucket set. `buckets` shrinks only `report`; `header` and `summary` stay those of the whole
scope. The top-level `header` carries the scope as every report does: `property` (with exactly one
property selected), `properties`, `landlord`, `landlords`, and `blocks`, `wings`, `floors` — see
[`header`](./README.md#header). Every period gets a row, and a property with no spaces still gets its bucket,
all zeros. There is no total row: occupancy at different moments does not add up.

Each row also carries `period_from`, `period_to` and `as_at` (the moment measured), as row metadata.

## Columns

| `key` | `label` | `format` | Notes |
|---|---|---|---|
| `period` | `Period` | `string` | e.g. `Jan 2019`, `Q1 2019`, `2019` |
| `total_spaces` | `Total Spaces` | `integer` | Today's space count |
| `occupied_spaces` | `Occupied` | `integer` | |
| `vacant_spaces` | `Vacant` | `integer` | `total_spaces − occupied_spaces` |
| `occupancy_rate` | `Occupancy %` | `string` | `occupied ÷ total`, preformatted, coloured with the shared bands |
| `occupied_area` | `Occupied Area` | `money` | A space measure rendered to two decimals, not a currency amount |

`occupancy_rate` is coloured above 80% `success`, 50–80% `warning`, below 50% `danger`, none where the
property has no spaces. The `all-properties` rate is recomputed from the summed counts, never averaged.

## Summary

Both bucket-level and report-level (`summary` = the `all-properties` bucket's). All figures are **at
the end of the last period**:

| Key | Meaning |
|---|---|
| `total_spaces` | Spaces in scope |
| `occupied_spaces` | Occupied at the end of the last period |
| `vacant_spaces` | Vacant at the end of the last period |
| `occupancy_rate` | `occupied_spaces ÷ total_spaces` as a number |
| `occupied_area` | Occupied area at the end of the last period |
| `period_count` | Period rows the cycle produced |

## Example response — one property bucket

```jsonc
{
  "bucket": "property-12",
  "header": { "label": "Kahawa House", "property": { "id": 12, "name": "Kahawa House" } },
  "fields": [
    { "label": "Period", "key": "period", "format": "string", "type": "normal", "weight": "font-normal", "background_color": "none", "alignment": "left", "visible": true, "togglable": false },
    { "label": "Total Spaces", "key": "total_spaces", "format": "integer", "type": "normal", "weight": "font-normal", "background_color": "none", "alignment": "right", "visible": true, "togglable": false },
    { "label": "Occupied", "key": "occupied_spaces", "format": "integer", "type": "normal", "weight": "font-normal", "background_color": "none", "alignment": "right", "visible": true, "togglable": false },
    { "label": "Vacant", "key": "vacant_spaces", "format": "integer", "type": "normal", "weight": "font-normal", "background_color": "none", "alignment": "right", "visible": true, "togglable": false },
    { "label": "Occupancy %", "key": "occupancy_rate", "format": "string", "type": "normal", "weight": "font-medium", "background_color": "none", "alignment": "right", "visible": true, "togglable": false },
    { "label": "Occupied Area", "key": "occupied_area", "format": "money", "type": "normal", "weight": "font-normal", "background_color": "none", "alignment": "right", "visible": true, "togglable": true }
  ],
  "items": [
    { "type": "normal", "period_from": "2025-01-01", "period_to": "2025-12-31", "as_at": "2025-12-31 23:59:59",
      "period": { "value": "2025" }, "total_spaces": { "value": 40 }, "occupied_spaces": { "value": 30 },
      "vacant_spaces": { "value": 10 }, "occupancy_rate": { "value": "75.00 %", "color": "warning" },
      "occupied_area": { "value": 18250.00 } },
    { "type": "normal", "period_from": "2026-01-01", "period_to": "2026-10-23", "as_at": "2026-10-23 23:59:59",
      "period": { "value": "2026" }, "total_spaces": { "value": 40 }, "occupied_spaces": { "value": 35 },
      "vacant_spaces": { "value": 5 }, "occupancy_rate": { "value": "87.50 %", "color": "success" },
      "occupied_area": { "value": 21000.00 } }
  ],
  "summary": { "total_spaces": 40, "occupied_spaces": 35, "vacant_spaces": 5, "occupancy_rate": 87.5, "occupied_area": 21000.00, "period_count": 2 }
}
```

## Export

`…/occupancy-trend/export?format=excel|pdf`, taking every filter the view endpoint takes. The frontend
exports the current view by replaying the query string with `format` appended.
