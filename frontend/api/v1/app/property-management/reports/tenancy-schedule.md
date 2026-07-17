# Tenancy Schedule Report API

Domain: `Property Management > Reports > Tenants`

Base route:

`/api/v1/app/{company}/property-management/reports`

The Tenancy Schedule report is a **point-in-time occupancy schedule**. For each property it lays out
the physical hierarchy **Block › Wing › Floor › Space** and, for every space, shows its name,
category, type, size, status, the tenant (when leased), the tenant's lease start/end and billing
cycle, and the per-month billable amounts (Rent, Service Charge, Parking, Signage — any autobilled
component). It is **read-only** and computed on the fly.

Occupancy is reconstructed **as of the selected period** (`period_from` … `period_to`, defaulting to
the current month): the report shows who held each space *then*, not just now — see
[Historical occupancy](#historical-occupancy).

> **Read [the shared reports contract](./README.md) first.** This page documents only what is specific
> to this report. The response envelope (`header` + a `report` array of buckets + `summary`), the
> bucket keys, the field keys, the per-cell `{ value, color }` shape, and the colour vocabulary all
> live in the README.

## The one place this report departs from the contract

Every other report's `items` is a **flat** list of rows. **This report's `items` is a nested tree.**
Group nodes (block / wing / floor) carry a `label` and a `children` array; only the **Space leaves**
carry the field-keyed cells that the `fields` describe. Render it with a tree-aware variant of the
report template (indent by `depth`); the envelope, bucket keys and cell shape are otherwise unchanged.

## Endpoints

- `GET /reports/tenants/tenancy-schedule`

## Filters (query params)

All filters are **optional** and flat (not nested under `filter[...]`). The six
[global filters](./README.md#global-filters) apply; this report adds two:

| Param | Type | Meaning when omitted |
|---|---|---|
| `landlord_id` | `users.id` | All landlords (global) |
| `facility_id` | `facilities.id` | All properties (global) |
| `facility_type_id` | `facility_types.id` | All property types (global) |
| `properties_status` | enum `App\Enums\Status` | Any property status (global) |
| `period_from` / `period_to` | date `Y-m-d` | Current month (global) — a schedule is a current snapshot, so the period is echoed but does not change occupancy |
| `facility_block_id` | `facility_blocks.id` | The whole property tree |
| `facility_wing_id` | `facility_wings.id` | The whole property tree |

- `facility_block_id` / `facility_wing_id` **require** a `facility_id`, and must belong to it — a `422`
  is returned otherwise.
- When a block (or wing) is selected, its **own** row is omitted and the tree **starts at its
  children** (its wings/floors, or floors), which become the top-level `items` at `depth: 0`.

## Buckets

- **With `facility_id`** → exactly **one** `property_{id}` bucket (optionally rooted at the selected
  block/wing).
- **Without `facility_id`** → **one `property_{id}` bucket per property** in scope, ordered by name.
  **There is no `overview` bucket** — every bucket is a property.

Each bucket's `header` carries `label` (the property name), `property { id, name }`, and `currency`
(the property's reporting currency, for reference — see the currency note below).

## Columns (`fields`)

The leaf/space columns, in order: `name` (Space), `category`, `type`, `size`, `status`, `tenant`,
`lease_start`, `lease_end`, `billing_cycle`, then one **money column per autobilled component present
in the property** (`component_{id}`, e.g. `Rent/M`, `Service Charge/M`, `Parking/M`, `Signage/M` —
each carries `component_id`, `togglable:true`), then `total` (Total/M, `type: grosstotal`).

All columns use `format: string`. The money columns are strings because their **values are
currency-prefixed, preformatted** (e.g. `"KES 120,000.00"`) rather than raw numbers — so the renderer
displays them as-is and right-aligns. A space that does not carry one of the property's component
columns shows a dash `"-"`.

### Derivations

- **category** — the space's *effective facility type* title: `FacilityType(space.type_id).title`, or,
  when the space has no `type_id`, the parent **property's** facility type title.
- **type** — Parking (`is_parking`, prefixed with the parking type, e.g. `Closed Parking`), Signage
  (`is_signage`), the residential-unit title when the space is a dwelling (`Studio` for a 0-bedroom
  unit), else `Leasable Space` / `Common Area` / `Landlord Space` by `space_type`.
- **size** — `facility_spaces.size` with the space unit, e.g. `"40 SqFt"`.
- **status** — **period-derived** (not the live stored status): computed from the leases that held the
  space during the selected period. Parking/signage are single-slot (`Unavailable` when held, else
  `Available`); other spaces follow the rent/SC split (`Unavailable` / `Only Service Charge Available`
  / `Only Rent Available` / `Available`), with `Under Consideration` for a free space that has an open
  application. The cell carries the status `color`.
- **tenant**, **lease_start** / **lease_end** (`Y-m-d`), **billing_cycle**, and **amounts** — from the
  space's **primary holding lease for the period** (rent holder → SC holder → parking → signage → the
  latest overlapping lease). Monthly amount per component is `SUM(lease_item_components.cost_per_month)`
  over autobilled components. Parking and signage occupancy is resolved through the lease's
  `is_parking_fee` / `is_signage_fee` components (there is no dedicated occupancy column for them).

### Currency

Amounts are shown in each **lease's own currency**, prefixed as a string (`"KES 120,000.00"`, 2dp).
The bucket header's `currency` is the property's reporting currency and is informational only; a
mixed-currency property is still rendered per its leases' currencies on each row.

## Historical occupancy

The report answers "who occupied this space during `period_from … period_to`?" — pick a past month and
you see the tenant, lease dates, amounts, and status as they were **then**.

A lease holds a space while its **occupancy window overlaps the selected period**. The window is
`[start_at … terminated_at]`:

- A lease **ends only when it is terminated** (completing a tenant exit notice terminates the lease,
  which stamps `terminated_at`). It does **not** end at `end_at` — `end_at` is merely the legal expiry;
  a lease past its `end_at` but not yet terminated is still active and still holding its spaces. `end_at`
  is shown only as the `lease_end` column.
- A never-terminated lease is **open-ended** and overlaps every period from its `start_at` onward.
- All lease statuses are considered when reconstructing a period (a lease terminated since then must
  still appear for a period it covered).

This applies uniformly to rent, service charge, parking, and signage — each resolved via its lease
component flag — so a parking bay or signage panel correctly shows its historical tenant, not a blank.

## Item node shapes

```jsonc
// Group node (block / wing / floor)
{ "row_kind": "block", "depth": 0, "label": "Block A", "children": [ /* … */ ] }

// Space leaf
{
  "row_kind": "space", "depth": 3,
  "name":        { "value": "Shop G1" },
  "category":    { "value": "Commercial" },
  "type":        { "value": "Leasable Space" },
  "size":        { "value": "40 SqFt" },
  "status":      { "value": "Unavailable", "color": "danger" },
  "tenant":      { "value": "Java House Ltd" },
  "lease_start": { "value": "2026-01-01" },
  "lease_end":   { "value": "2028-12-31" },
  "billing_cycle": { "value": "monthly" },
  "component_1": { "value": "KES 120,000.00" },
  "component_2": { "value": "KES 30,000.00" },
  "component_8": { "value": "-" },
  "total":       { "value": "KES 150,000.00" }
}
```

`depth` is always relative to the bucket root, so the frontend indents purely by `depth` and never has
to know which levels a given property uses. Shallow properties simply produce shallower trees:

- only floors → `floor` (depth 0) → `space` (depth 1)
- only blocks → `block` (0) → `floor` (1) → `space` (2)
- only wings → `wing` (0) → `floor` (1) → `space` (2)

## Summary

Both the bucket `summary` and the report-level `summary` are **segmented by space class**:

```jsonc
"summary": {
  "commercial":  { "total_space": "500 SqFt", "occupied": "400 SqFt", "available": "100 SqFt", "rate": 0.80 },
  "parking":     { "total_spaces": 20, "occupied": 14, "available": 6, "rate": 0.70 },
  "signage":     { "total_spaces": 4,  "occupied": 4,  "available": 0, "rate": 1.00 },
  "residential": {
    "total_units": 30, "occupied": 22, "available": 8, "rate": 0.733,
    "by_type": [
      { "type": "Studio",    "total_units": 18, "occupied": 15, "available": 3, "rate": 0.833 },
      { "type": "2 Bedroom", "total_units": 12, "occupied": 7,  "available": 5, "rate": 0.583 }
    ]
  }
}
```

- **Commercial** is measured by **leasable area**, not space count: `total_space` is the property's
  `leasable_space` (or the selected block/wing's, when the tree is rooted there) and `occupied` is the
  summed **size** of its leased spaces. `available` = `total_space − occupied`; `rate` =
  occupied / total_space. The three area figures are preformatted `"<n> SqFt"` strings; `rate` stays a
  number.
- **Parking / Signage** are counted **by space**: `occupied` = the space is leased in the period;
  `available` = not; `rate` = occupied / total.
- **Residential** is rolled up **per residential-unit type** using each unit's `quantity` /
  `allocated` (`available` = `quantity − allocated`), so units shared by several spaces are not
  double-counted; `by_type` breaks it down per unit title.

There is no pagination, sort, or include — a report is returned whole for the resolved period.
