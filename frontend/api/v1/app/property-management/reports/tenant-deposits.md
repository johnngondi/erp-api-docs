# Tenant Deposits Report API

Domain: `Property Management > Reports > Tenants`

Base route:

`/api/v1/app/{company}/property-management/reports`

Tenant Deposits is the register of **what deposit is held against each lease** — broken down by the
component each deposit was taken against, with what has been refunded and what is still held. It is
**read-only** and computed on the fly.

> **Read [the shared reports contract](./README.md) first.** This page documents what is specific to
> this report. The response envelope, the bucket keys, the field keys, the per-cell `{ value, color }`
> shape and the colour vocabulary live in the README and apply here unchanged.

## Endpoints

- `GET /reports/tenants/tenant-deposits` — requires `view-tenant-deposits-report`
- `GET /reports/tenants/tenant-deposits/export` — requires `export-tenant-deposits-report`. Takes a
  required `format` (`excel` | `pdf`); any other value, or none, returns `422` with a `format` error.
- Listed by [`GET /reports`](./README.md#discovering-reports--get-reports).

The two permissions are **independent**.

## Refunds are all-or-nothing

**A deposit is refunded in full or not at all. There is no partial refund, and there never can be.**

`lease_deposits.refunded_at` is a **timestamp, not an amount** — the schema has nowhere to put a
part-refund — and the only code that sets it (`ProcessExitNoticeAction`) stamps *every* deposit on
the lease in a single update when the exit notice completes. So the rule is simply:

| `refunded_at` | `refunded` | `currently_held` |
|---|---|---|
| not null | the **whole** `amount` | `0` |
| null | `0` | the **whole** `amount` |

`refunded + currently_held` always equals `total`, on every row.

> **Do not build a partial-refund UI for this.** No fixture, and no state the application can reach,
> produces a fraction. A row *can* show both figures non-zero — but only when the lease holds several
> deposits and some of them are refunded, never by splitting one.

`refunded_by_exit_notice_id` traces a refunded deposit back to the exit notice that released it. It
ships **`visible: false, togglable: true`** — off by default, there for reconciliation. A lease with
nothing refunded omits the key.

## Deposit-type columns are generated at runtime

**Never hardcode the deposit columns. Read `fields`.**

A deposit's type is the lease component it was taken against, resolved
`lease_deposits.lease_item_component_id` → `lease_item_components.lease_component_id` →
`lease_components`. Each distinct component becomes one column:

```jsonc
{ "label": "Service Charge", "key": "deposit_component_2", "format": "money", "type": "normal",
  "weight": "font-normal", "background_color": "none", "alignment": "right",
  "visible": true, "togglable": true, "component_id": 2 }
```

- Keyed `deposit_component_{component id}`, labelled with the component's name, carrying
  `component_id` so you can group or deep-link without parsing the key.
- **Togglable**, ordered by component name.
- The set is the **union of the types present in this response** — so it changes with the filters. Ask
  for `leases_status=active` and a type only terminated leases hold **drops out of `fields`
  entirely**.

### An omitted key is not a zero

**A lease holding no deposit of a given type omits that key from its row.** Per the shared contract a
missing field key renders as an **empty cell** — which is the point: "no rent deposit was ever taken"
and "a rent deposit of nil was taken" are different statements, and the register should not turn the
first into the second.

```jsonc
// Acme holds Rent + Service Charge. No `deposit_component_7` key at all.
{ "type": "normal", "tenant": { "value": "Acme Traders Ltd" },
  "deposit_component_1": { "value": 900000.00 }, "deposit_component_2": { "value": 240000.00 },
  "total": { "value": 1140000.00 } }

// Njiru holds Rent + Water. No `deposit_component_2` key at all.
{ "type": "normal", "tenant": { "value": "Njiru Hardware" },
  "deposit_component_1": { "value": 450000.00 }, "deposit_component_7": { "value": 30000.00 },
  "total": { "value": 480000.00 } }
```

## A register, not a period

`period_from` / `period_to` are echoed in `header.period` and `header.filters` but **narrow nothing**
— a deposit held is a position, not a movement. Same treatment as
[Management Contracts](./management-contracts.md). Don't present the period as though it scoped the
figures.

## Filters

The six [global filters](./README.md#global-filters) plus one:

| Param | Type | Meaning when omitted |
|---|---|---|
| six global filters | see the shared contract | all / current month (echoed only) |
| `leases_status` | `active` \| `suspended` \| `terminated` | Any lease status |

`header.filters` echoes all seven, `null` included.

```
GET …/reports/tenants/tenant-deposits?facility_id=1&leases_status=active
```

Money is shown as stored, in each property's reporting currency — nothing is converted.

## Buckets

One bucket:

| `bucket` | `header.label` | Rows |
|---|---|---|
| `overview` | `Tenant Deposits` | one row per lease holding a deposit, ordered by property then tenant, then a `subtotal` row |

A lease with no deposits does not appear at all — this is a register of deposits, not of leases.

## Columns

| `key` | `label` | `format` | Notes |
|---|---|---|---|
| `tenant` | `Tenant` | `string` | `—` where the lease has no tenant |
| `property` | `Property` | `string` | |
| `lease_id` | `Lease` | `integer` | The lease's id |
| `start_date` | `Start` | `string` | `01 Jan, 2026` |
| `end_date` | `End` | `string` | `31 Dec, 2028`, or `—` |
| `status` | `Status` | `string` | The lease's status |
| `deposit_component_{id}` | the component's name | `money` | **Dynamic** — see above. `togglable: true` |
| `total` | `Total` | `money` | Every deposit on the lease |
| `refunded` | `Refunded` | `money` | All-or-nothing |
| `currently_held` | `Currently Held` | `money` | `type: subtotal` |
| `refunded_by_exit_notice_id` | `Refund Exit Notice` | `string` | **`visible: false, togglable: true`** |

### The subtotal row

Its label spans the six text columns:

```jsonc
{ "type": "subtotal", "background_color": "secondary",
  "tenant": { "value": "2 leases", "col_span": 6 },
  "deposit_component_1": { "value": 1350000.00 }, "deposit_component_2": { "value": 240000.00 },
  "deposit_component_7": { "value": 30000.00 },
  "total": { "value": 1620000.00 }, "refunded": { "value": 480000.00 }, "currently_held": { "value": 1140000.00 } }
```

`col_span: 6` means the row **omits the next five field keys** (`property` … `status`) — walk `fields`
with a skip counter, as the shared contract describes. The label reads `1 lease` / `N leases`.

## Summary

Both bucket-level and report-level `summary`:

| Key | Meaning |
|---|---|
| `lease_count` | Leases listed — the number of normal rows |
| `deposit_count` | Deposit-type entries across those leases |
| `total_deposits` | Sum of every deposit amount |
| `total_refunded` | Deposits whose `refunded_at` is set, at full amount |
| `total_held` | Deposits whose `refunded_at` is null, at full amount |

`total_refunded + total_held == total_deposits`, always.

## Example response

```jsonc
{
  "data": {
    "header": {
      "property": { "id": 1, "name": "KAHAWA HOUSE", "currency": { "code": "KES", "name": "Kenyan Shilling" } },
      "landlord": null,
      "period": { "from": "2026-07-01", "to": "2026-07-31" },
      "filters": { "landlord_id": null, "facility_id": 1, "facility_type_id": null, "properties_status": null,
                   "period_from": "2026-07-01", "period_to": "2026-07-31", "leases_status": "active" },
      "generated_at": "2026-08-17T09:12:44+00:00"
    },
    "report": [
      {
        "bucket": "overview",
        "header": { "label": "Tenant Deposits" },
        "fields": [ /* the static columns, with deposit_component_* spliced in before `total` */ ],
        "items": [
          { "type": "normal", "tenant": { "value": "Acme Traders Ltd" }, "property": { "value": "KAHAWA HOUSE" },
            "lease_id": { "value": 412 }, "start_date": { "value": "01 Jan, 2026" }, "end_date": { "value": "31 Dec, 2028" },
            "status": { "value": "active" },
            "deposit_component_1": { "value": 900000.00 }, "deposit_component_2": { "value": 240000.00 },
            "total": { "value": 1140000.00 }, "refunded": { "value": 0 }, "currently_held": { "value": 1140000.00 } },
          { "type": "normal", "tenant": { "value": "Njiru Hardware" }, "property": { "value": "KAHAWA HOUSE" },
            "lease_id": { "value": 388 }, "start_date": { "value": "01 Mar, 2024" }, "end_date": { "value": "28 Feb, 2026" },
            "status": { "value": "terminated" },
            "deposit_component_1": { "value": 450000.00 }, "deposit_component_7": { "value": 30000.00 },
            "total": { "value": 480000.00 }, "refunded": { "value": 480000.00 }, "currently_held": { "value": 0 },
            "refunded_by_exit_notice_id": { "value": 91 } },
          { "type": "subtotal", "background_color": "secondary",
            "tenant": { "value": "2 leases", "col_span": 6 },
            "deposit_component_1": { "value": 1350000.00 }, "deposit_component_2": { "value": 240000.00 },
            "deposit_component_7": { "value": 30000.00 },
            "total": { "value": 1620000.00 }, "refunded": { "value": 480000.00 }, "currently_held": { "value": 1140000.00 } }
        ],
        "summary": { "lease_count": 2, "deposit_count": 4, "total_deposits": 1620000.00,
                     "total_refunded": 480000.00, "total_held": 1140000.00 }
      }
    ],
    "summary": { "lease_count": 2, "deposit_count": 4, "total_deposits": 1620000.00,
                 "total_refunded": 480000.00, "total_held": 1140000.00 }
  }
}
```

Row 1 has `refunded_at` null, so `refunded` is `0` and `currently_held` is the whole total, and the
row omits `refunded_by_exit_notice_id`. Row 2 has it set, so the figures are the other way round.
**Neither row is split** — that is the shape of every row in this report.

## Export

`…/tenant-deposits/export?format=excel|pdf`, taking every filter the view endpoint takes,
`leases_status` included — so the export has exactly the deposit-type columns on screen. The frontend
exports the current view by replaying the query string with `format` appended.
