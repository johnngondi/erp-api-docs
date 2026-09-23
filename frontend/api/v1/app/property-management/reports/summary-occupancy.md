# Summary Occupancy Report API

Domain: `Property Management > Reports > Tenants > Tenancy Reports`

Base route:

`/api/v1/app/{company}/property-management/reports`

Summary Occupancy answers **how much of each property is let** — occupied against vacant, counted
both by unit and by area. It is **read-only** and computed on the fly.

> **Read [the shared reports contract](./README.md) first.** This page documents what is specific to
> this report. The response envelope, the bucket keys, the field keys, the per-cell `{ value, color }`
> shape and the colour vocabulary live in the README and apply here unchanged.

## Endpoints

- `GET /reports/tenants/tenancy-reports/summary-occupancy` — requires `view-summary-occupancy-report`
- `GET /reports/tenants/tenancy-reports/summary-occupancy/export` — requires
  `export-summary-occupancy-report`. Takes a required `format` (`excel` | `pdf`); any other value, or
  none, returns `422` with a `format` error.
- Listed by [`GET /reports`](./README.md#discovering-reports--get-reports).

The two permissions are **independent**, and neither is shared with
[Tenancy Schedule](./tenancy-schedule.md), its sibling under the same parent.

## The six-status mapping

The source is `facility_spaces` — every space of every property in scope, reached through all four
floor shapes (a floor hung off the facility, off a block, off a wing, or off a block's wing).

A space is occupied or vacant according to its **own `status`**, and all six statuses are mapped
explicitly:

| `facility_spaces.status` | Counts as |
|---|---|
| `available` | vacant |
| `under consideration` | vacant |
| `only rent available` | **vacant** |
| `only service charge available` | **occupied** |
| `on notice` | occupied |
| `unavailable` | occupied |

**The two partial statuses fall on opposite sides, and that is deliberate.** A space carries separate
`rent_occupied_by_lease_id` and `service_charge_occupied_by_lease_id` pointers, so:

- `only rent available` — the **rent side is still to let**, so the space is **vacant**.
- `only service charge available` — the **rent is already let**, so the space is **occupied**.

Any rule phrased as *"everything except `available` is occupied"* gets `only rent available` wrong,
and nothing in the output will show it. Do not re-derive this mapping on the frontend: render the
`occupied_units` / `vacant_units` the backend returns.

## This is a snapshot, not a period

`status` is the space's state **now**, so `period_from` / `period_to` are echoed in `header.period`
and `header.filters` but **change no figure**. Two consequences for the UI:

- Do not present the period as though it scoped the numbers. It is there so the export header and
  the on-screen header agree with every other report.
- **This report will sometimes disagree with [Property Status](./property-status.md)** about the same
  property, by design. Property Status asks whether *a lease covered the space during the period*; a
  space `on notice` whose lease has already ended is **occupied here and vacant there**. Neither is
  wrong — they answer different questions.

## Filters

The six [global filters](./README.md#global-filters) and nothing else. `header.filters` echoes all
six, `null` included.

```
GET …/reports/tenants/tenancy-reports/summary-occupancy?landlord_id=4&facility_type_id=2&properties_status=active
```

## Buckets

One bucket:

| `bucket` | `header.label` | Rows |
|---|---|---|
| `overview` | `Summary Occupancy` | one row per property in scope, ordered by name, then a `Total` `subtotal` row |

**A property with no spaces still gets a row**, all zeros — so a newly onboarded property is visibly
empty rather than missing.

## Columns

| `key` | `label` | `format` | Notes |
|---|---|---|---|
| `property` | `Property` | `string` | |
| `total_units` | `Total Units` | `integer` | Spaces counted |
| `occupied_units` | `Occupied Units` | `integer` | Per the mapping above |
| `vacant_units` | `Vacant Units` | `integer` | `total_units − occupied_units` |
| `total_area` | `Total Area` | `money` | Sum of `facility_spaces.size` |
| `occupied_area` | `Occupied Area` | `money` | |
| `vacant_area` | `Vacant Area` | `money` | `total_area − occupied_area` |
| `occupancy_by_unit` | `Occupancy % (Unit)` | **`string`** | Preformatted, e.g. `"93.55 %"` |
| `occupancy_by_area` | `Occupancy % (Area)` | **`string`** | Preformatted |

> **The three area columns use `format: money`** so they render to two decimals like the other
> numeric columns. They are a **space measure, not a currency amount** — do not prefix them with the
> header's currency code. Their unit is the property's own space unit (SqFt, SqM…).

The two occupancy columns are **strings, not numbers**: already rounded and formatted. They carry no
`color` — this report does not band occupancy. The numeric values are on `summary`.

## Percentages are recomputed, never averaged

**Every percentage — including the `Total` row's — is computed from that row's own totals.**

| Property | Total units | Occupied | By unit |
|---|---|---|---|
| Big House | 10 | 9 | 90.00 % |
| Small House | 1 | 0 | 0.00 % |
| **Total** | **11** | **9** | **81.82 %** |

The mean of `90.00` and `0.00` is `45.00`. The subtotal reports **81.82 %**, because 9 of 11 spaces
are let. The same holds for area, and the two are computed independently — a property can be 25 %
occupied by unit and 10 % by area.

**A property with no spaces reports `0.00 %`**, not an error and not `null`: there is nothing to
divide by, so there is nothing to report.

## Summary

Both bucket-level and report-level `summary`:

| Key | Meaning |
|---|---|
| `property_count` | Properties in scope |
| `total_units` | Spaces counted |
| `occupied_units` | Spaces whose status counts as occupied |
| `occupancy_by_unit` | `occupied_units ÷ total_units` as a **number**, e.g. `81.82` |
| `occupancy_by_area` | `occupied_area ÷ total_area` as a **number** |

## Example response

```jsonc
{
  "data": {
    "header": {
      "property": null,
      "currency": { "code": "KES", "name": "Kenyan Shilling" },
      "period": { "from": "2026-07-01", "to": "2026-07-31" },
      "filters": { "landlord_id": 4, "facility_id": null, "facility_type_id": 2,
                   "properties_status": "active", "period_from": "2026-07-01", "period_to": "2026-07-31" },
      "generated_at": "2026-08-17T09:12:44+00:00"
    },
    "report": [
      {
        "bucket": "overview",
        "header": { "label": "Summary Occupancy" },
        "fields": [ /* the nine columns above */ ],
        "items": [
          { "type": "normal", "property": { "value": "KAHAWA HOUSE" },
            "total_units": { "value": 62 }, "occupied_units": { "value": 58 }, "vacant_units": { "value": 4 },
            "total_area": { "value": 39932.32 }, "occupied_area": { "value": 37350.10 }, "vacant_area": { "value": 2582.22 },
            "occupancy_by_unit": { "value": "93.55 %" }, "occupancy_by_area": { "value": "93.53 %" } },
          { "type": "normal", "property": { "value": "Kahawa House Side Tower" },
            "total_units": { "value": 0 }, "occupied_units": { "value": 0 }, "vacant_units": { "value": 0 },
            "total_area": { "value": 0 }, "occupied_area": { "value": 0 }, "vacant_area": { "value": 0 },
            "occupancy_by_unit": { "value": "0.00 %" }, "occupancy_by_area": { "value": "0.00 %" } },
          { "type": "subtotal", "background_color": "secondary", "property": { "value": "Total" },
            "total_units": { "value": 62 }, "occupied_units": { "value": 58 }, "vacant_units": { "value": 4 },
            "total_area": { "value": 39932.32 }, "occupied_area": { "value": 37350.10 }, "vacant_area": { "value": 2582.22 },
            "occupancy_by_unit": { "value": "93.55 %" }, "occupancy_by_area": { "value": "93.53 %" } }
        ],
        "summary": { "property_count": 2, "total_units": 62, "occupied_units": 58,
                     "occupancy_by_unit": 93.55, "occupancy_by_area": 93.53 }
      }
    ],
    "summary": { "property_count": 2, "total_units": 62, "occupied_units": 58,
                 "occupancy_by_unit": 93.55, "occupancy_by_area": 93.53 }
  }
}
```

## Export

`…/summary-occupancy/export?format=excel|pdf`, taking every filter the view endpoint takes. The
frontend exports exactly what is on screen by replaying the query string with `format` appended.
