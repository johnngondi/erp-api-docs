# Collections Summary Report API

Domain: `Property Management > Reports > Tenants > Collections Reports`

Base route:

`/api/v1/app/{company}/property-management/reports`

Collections Summary is [Property Collections](./property-collections.md) **read across time**: the
same three buckets, but each one a matrix with a money column per period of the chosen cycle. It
answers **when** the money came in. It is **read-only** and computed on the fly.

> **Read [the shared reports contract](./README.md) first.** This page documents what is specific to
> this report. The response envelope, the bucket keys, the field keys, the per-cell `{ value, color }`
> shape and the colour vocabulary live in the README and apply here unchanged.

## Endpoints

- `GET /reports/tenants/collections-reports/collections-summary` — requires
  `view-collections-summary-report`
- `GET /reports/tenants/collections-reports/collections-summary/export` — requires
  `export-collections-summary-report`. Takes a required `format` (`excel` | `pdf`); any other value,
  or none, returns `422` with a `format` error.
- Listed by [`GET /reports`](./README.md#discovering-reports--get-reports).

The two permissions are **independent**, and neither is shared with
[Property Collections](./property-collections.md) or
[Legal Fee Collection](./legal-fee-collection.md).

## Collections only

There is **no billed figure anywhere in this report**. Billed against collected is what
[Property Collections](./property-collections.md) is for; a matrix of what came in has no room for it.

Both reports read one shared source, so **`summary.total_collected` here equals Property Collections'
`summary.total_collected`** for the same window — and it does not change when you change
`report_cycle`, because the cycle only decides how the money is spread across columns, never how much
of it there is. That equality is the quickest way to tell the two reports are in sync.

As there, collected means `lease_collections` with status `confirmed`; a `cancelled` collection never
appears.

## Filters

The six [global filters](./README.md#global-filters) plus one:

| Param | Type | Meaning when omitted |
|---|---|---|
| six global filters | see the shared contract | all / current month |
| `report_cycle` | `weekly` \| `monthly` \| `quarterly` \| `annually` | **`monthly`** |

`header.filters` echoes all seven, `null` included — including `report_cycle`, which is `null` when
the default was used.

```
GET …/reports/tenants/collections-reports/collections-summary?period_from=2026-05-01&period_to=2026-07-31&report_cycle=monthly
```

Money is shown as stored, in each property's reporting currency — nothing is converted.

## Buckets

The same three levels as Property Collections, always in this order:

| `bucket` | `header.label` | Rows |
|---|---|---|
| `lease-components` | `By Lease Component` | one row per lease component collected anywhere in scope, ordered by name, then a `Total` `subtotal` row |
| `all-properties` | `All Properties` | one row per property in scope, ordered by landlord then name, then a `Total` `subtotal` row |
| `landlord-{id}` | the landlord's name, or `Unassigned` | that landlord's properties only, then its own `Total` row |

Landlord buckets carry `header.landlord` `{ id, name }`, `null` where the properties have no landlord
(the bucket id is then `landlord-0`). A property that collected nothing still gets a row, all zeros.

## Columns — generated at runtime

**Never hardcode the columns. Read `fields`.** The window is sliced into the cycle's calendar
periods, and each one contributes a money column:

| `key` | `label` | `format` | `togglable` | Notes |
|---|---|---|---|---|
| `component` / `property` | `Component` / `Property` | `string` | `false` | The leading label column |
| `period_0` … `period_n` | e.g. `May 2026` | `money` | **`true`** | One per period, in order; carries `period_from` and `period_to` |
| `total` | `Total` | `money` | `false` | `type: grosstotal` — the row's periods summed |

Every period column is **togglable**, so a long window stays readable; `total` is not.

Each period field carries its own bounds, so you never parse the key:

```jsonc
{ "label": "May 2026", "key": "period_0", "format": "money", "type": "normal",
  "weight": "font-normal", "background_color": "none", "alignment": "right",
  "visible": true, "togglable": true,
  "period_from": "2026-05-01", "period_to": "2026-05-31" }
```

> **Key naming.** Columns are keyed `period_0`, `period_1`, … — positional, not named after the dates
> they cover. This is the same convention the Income & Expenditure and
> [SC vs Expenses Summary](./sc-vs-expenses-summary.md) matrices use. The dates are on
> `period_from` / `period_to`; the `label` is what to print in the header.

## Periods are calendar-anchored

A period column is a whole week / month / quarter / year, **clamped** to the requested window. So a
column at either edge of the window may be a partial period, and it only ever counts money inside
`[period_from, period_to]`.

This means **the column count follows the calendar, not the length of the window**:

| Window | `report_cycle` | Columns |
|---|---|---|
| 2026-05-01 → 2026-07-31 | `monthly` | 3 — `May 2026`, `Jun 2026`, `Jul 2026` |
| 2026-05-01 → 2026-07-31 | `quarterly` | **2** — `Q2 2026` (May–Jun) and `Q3 2026` (Jul), because the window straddles the quarter boundary |
| 2026-04-01 → 2026-06-30 | `quarterly` | 1 — `Q2 2026` |
| 2026-07-01 → 2026-07-31 | `weekly` | 5 — ISO weeks, the first and last partial |

`summary.period_count` always tells you how many there are. Drive the table off `fields`, not off
your own date arithmetic.

## Summary

Both bucket-level and report-level `summary`:

| Key | Meaning |
|---|---|
| `total_collected` | Confirmed collections in the period. Equal to Property Collections' figure, and identical across every `report_cycle` |
| `period_count` | Period columns the cycle produced |

## Example response — the `all-properties` bucket

```jsonc
{
  "bucket": "all-properties",
  "header": { "label": "All Properties" },
  "fields": [
    { "label": "Property", "key": "property", "format": "string", "type": "normal", "weight": "font-normal", "background_color": "none", "alignment": "left", "visible": true, "togglable": false },
    { "label": "May 2026", "key": "period_0", "format": "money", "type": "normal", "weight": "font-normal", "background_color": "none", "alignment": "right", "visible": true, "togglable": true, "period_from": "2026-05-01", "period_to": "2026-05-31" },
    { "label": "Jun 2026", "key": "period_1", "format": "money", "type": "normal", "weight": "font-normal", "background_color": "none", "alignment": "right", "visible": true, "togglable": true, "period_from": "2026-06-01", "period_to": "2026-06-30" },
    { "label": "Jul 2026", "key": "period_2", "format": "money", "type": "normal", "weight": "font-normal", "background_color": "none", "alignment": "right", "visible": true, "togglable": true, "period_from": "2026-07-01", "period_to": "2026-07-31" },
    { "label": "Total", "key": "total", "format": "money", "type": "grosstotal", "weight": "font-bold", "background_color": "none", "alignment": "right", "visible": true, "togglable": false }
  ],
  "items": [
    { "type": "normal", "property": { "value": "KAHAWA HOUSE" },
      "period_0": { "value": 2768333.33 }, "period_1": { "value": 2808333.33 },
      "period_2": { "value": 2768333.34 }, "total": { "value": 8345000.00 } },
    { "type": "subtotal", "background_color": "secondary", "property": { "value": "Total" },
      "period_0": { "value": 2798333.33 }, "period_1": { "value": 3108333.33 },
      "period_2": { "value": 2768333.34 }, "total": { "value": 8675000.00 } }
  ],
  "summary": { "total_collected": 8675000.00, "period_count": 3 }
}
```

No cell in this report carries a `color` — there is no rate to band here. Colour lives in
[Property Collections](./property-collections.md).

## Export

`…/collections-summary/export?format=excel|pdf`, taking every filter the view endpoint takes,
`report_cycle` included — so the export has exactly the columns on screen. The frontend exports the
current view by replaying the query string with `format` appended.
