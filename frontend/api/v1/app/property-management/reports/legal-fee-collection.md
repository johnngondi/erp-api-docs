# Legal Fee Collection Report API

Domain: `Property Management > Reports > Tenants > Collections Reports`

Base route:

`/api/v1/app/{company}/property-management/reports`

Legal Fee Collection lists **legal fees billed, what has been recovered, and what is still owed**, a
row per lease, with how long each debt has been outstanding. It is **read-only** and computed on the
fly.

> **Read [the shared reports contract](./README.md) first.** This page documents what is specific to
> this report. The response envelope, the bucket keys, the field keys, the per-cell `{ value, color }`
> shape and the colour vocabulary live in the README and apply here unchanged.

## Endpoints

- `GET /reports/tenants/collections-reports/legal-fee-collection` — requires
  `view-legal-fee-collection-report`
- `GET /reports/tenants/collections-reports/legal-fee-collection/export` — requires
  `export-legal-fee-collection-report`. Takes a required `format` (`excel` | `pdf`); any other value,
  or none, returns `422` with a `format` error.
- Listed by [`GET /reports`](./README.md#discovering-reports--get-reports).

The two permissions are **independent**, and neither is shared with
[Property Collections](./property-collections.md) or
[Collections Summary](./collections-summary.md).

## The component is selected by flag, not by name

Only billings and collections against a lease component flagged **`is_legal_fees_deposit`** count.
That flag lives on the component itself and is settable through the
[lease component settings endpoints](../settings.md#lease-components).

**The report never looks at the component's name or id.** A `Legal Fees` component ships flagged, but
it can be renamed to anything — `Conveyancing Costs`, `Advocate Fees` — and this report returns
exactly the same rows. The consequences for the frontend:

- Do **not** label anything in this screen by reading a component called "Legal Fees"; there may not
  be one.
- A settings screen that clears the flag on the shipped component and sets it on another silently
  re-points this report — which is the intended way to change which component legal fees are billed
  against. Only one component should carry the flag at a time.

As elsewhere, **collected means `lease_collections` with status `confirmed`**; a `cancelled`
collection never appears. Billings have no status.

## Filters

The six [global filters](./README.md#global-filters) and nothing else. `header.filters` echoes all
six, `null` included.

```
GET …/reports/tenants/collections-reports/legal-fee-collection?facility_id=1&period_from=2026-05-01&period_to=2026-07-31
```

There is deliberately **no component filter**. The report is defined by the flag; pointing it at
another component would make it a different report.

Money is shown as stored, in each property's reporting currency — nothing is converted.

## Buckets

One bucket:

| `bucket` | `header.label` | Rows |
|---|---|---|
| `overview` | `Legal Fee Collection` | one row per lease with legal fees billed or recovered in the window, ordered by property then tenant, then a `subtotal` row |

The bucket is **always present**, even when nothing matched: it then holds only its subtotal row,
reading `0 leases` with zero totals.

## Columns

| `key` | `label` | `format` | Notes |
|---|---|---|---|
| `tenant` | `Tenant` | `string` | The lease's tenant; `—` where the lease has none |
| `property` | `Property` | `string` | |
| `unit` | `Unit` | `string` | **Every** space on the lease, comma-separated; `—` where the lease has no space |
| `legal_fee_billed` | `Legal Fee Billed` | `money` | Billed in the window |
| `recovered` | `Recovered` | `money` | Confirmed collections in the window |
| `outstanding` | `Outstanding` | `money` | `legal_fee_billed − recovered`; tinted `danger` while above zero |
| `days_outstanding` | `Days Outstanding` | `integer` | See below |

### `days_outstanding`

Days from the **earliest legal-fee billing in the window** to **`period_to`** — "how long has this
been owed, as at the report date".

It is **`0`** when:

- nothing is outstanding (a settled lease is not ageing), or
- the recovery in this window sits against a billing raised *outside* it, so there is no billing date
  in scope to measure from.

The `subtotal` row **omits `days_outstanding` entirely** — ageing is a property of a lease, not of a
portfolio. Per the shared contract, a missing field key renders as an empty cell.

### The subtotal row

Its label spans the three text columns:

```jsonc
{ "type": "subtotal", "background_color": "secondary",
  "tenant": { "value": "2 leases", "col_span": 3 },
  "legal_fee_billed": { "value": 150000.00 }, "recovered": { "value": 70000.00 },
  "outstanding": { "value": 80000.00 } }
```

`col_span: 3` means the row **omits the next two field keys** (`property`, `unit`) — walk `fields`
with a skip counter, as the shared contract describes. The label is `1 lease` / `N leases`. The
subtotal's `outstanding` carries **no colour**; the tint marks individual debts, not the total.

## Summary

Both bucket-level and report-level `summary`:

| Key | Meaning |
|---|---|
| `lease_count` | Leases listed |
| `total_billed` | Legal fees billed in the period |
| `total_recovered` | Legal fees recovered in the period |
| `total_outstanding` | `total_billed − total_recovered` |

## Example response

```jsonc
{
  "data": {
    "header": {
      "property": { "id": 1, "name": "KAHAWA HOUSE", "currency": { "code": "KES", "name": "Kenyan Shilling" } },
      "period": { "from": "2026-05-01", "to": "2026-07-31" },
      "filters": { "landlord_id": null, "facility_id": 1, "facility_type_id": null,
                   "properties_status": null, "period_from": "2026-05-01", "period_to": "2026-07-31" },
      "generated_at": "2026-08-17T09:12:44+00:00"
    },
    "report": [
      {
        "bucket": "overview",
        "header": { "label": "Legal Fee Collection" },
        "fields": [ /* the seven columns above */ ],
        "items": [
          { "type": "normal", "tenant": { "value": "Acme Traders Ltd" },
            "property": { "value": "KAHAWA HOUSE" }, "unit": { "value": "3rd Flr — 3B" },
            "legal_fee_billed": { "value": 120000.00 }, "recovered": { "value": 40000.00 },
            "outstanding": { "value": 80000.00, "color": "danger" },
            "days_outstanding": { "value": 91 } },
          { "type": "normal", "tenant": { "value": "Njiru Hardware" },
            "property": { "value": "NJIRU PLAZA" }, "unit": { "value": "Shop 4" },
            "legal_fee_billed": { "value": 30000.00 }, "recovered": { "value": 30000.00 },
            "outstanding": { "value": 0 },
            "days_outstanding": { "value": 0 } },
          { "type": "subtotal", "background_color": "secondary",
            "tenant": { "value": "2 leases", "col_span": 3 },
            "legal_fee_billed": { "value": 150000.00 }, "recovered": { "value": 70000.00 },
            "outstanding": { "value": 80000.00 } }
        ],
        "summary": { "lease_count": 2, "total_billed": 150000.00, "total_recovered": 70000.00, "total_outstanding": 80000.00 }
      }
    ],
    "summary": { "lease_count": 2, "total_billed": 150000.00, "total_recovered": 70000.00, "total_outstanding": 80000.00 }
  }
}
```

Row two is settled: `outstanding` is `0` and carries **no `color` key** (absent means `none`), and
`days_outstanding` is `0`.

## Export

`…/legal-fee-collection/export?format=excel|pdf`, taking every filter the view endpoint takes. The
frontend exports exactly what is on screen by replaying the query string with `format` appended.
