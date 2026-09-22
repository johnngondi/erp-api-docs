# Property Collections Report API

Domain: `Property Management > Reports > Tenants > Collections Reports`

Base route:

`/api/v1/app/{company}/property-management/reports`

Property Collections answers **how much of what we billed did we actually collect**, for the period,
cut three ways: by lease component, by property, and once per landlord. It is **read-only** and
computed on the fly.

> **Read [the shared reports contract](./README.md) first.** This page documents what is specific to
> this report. The response envelope, the bucket keys, the field keys, the per-cell `{ value, color }`
> shape and the colour vocabulary live in the README and apply here unchanged.

## Endpoints

- `GET /reports/tenants/collections-reports/property-collections` — requires
  `view-property-collections-report`
- `GET /reports/tenants/collections-reports/property-collections/export` — requires
  `export-property-collections-report`. Takes a required `format` (`excel` | `pdf`); any other value,
  or none, returns `422` with a `format` error.
- Listed by [`GET /reports`](./README.md#discovering-reports--get-reports).

The two permissions are **independent**, and neither is shared with
[Collections Summary](./collections-summary.md) or
[Legal Fee Collection](./legal-fee-collection.md).

## What is counted

- **Billed** is `lease_billings` in the period — every component, no status filter (a billing has no
  status).
- **Collected** is `lease_collections` in the period **whose status is `confirmed`**. A `cancelled`
  collection is not money collected and never appears.
- Both reach their property through the lease (`leases.facility_id`).

[Collections Summary](./collections-summary.md) and
[Legal Fee Collection](./legal-fee-collection.md) read the same source, so the three reconcile for
the same window: this report's `summary.total_collected` equals Collections Summary's.

## Filters

The six [global filters](./README.md#global-filters) and nothing else. `header.filters` echoes all
six, `null` included.

```
GET …/reports/tenants/collections-reports/property-collections?facility_id=1&period_from=2026-05-01&period_to=2026-07-31
```

`facility_id` narrows the scope but does **not** change the bucket set: `all-properties` and the
landlord bucket then hold that one property.

Money is shown as stored, in each property's reporting currency — nothing is converted.

## Buckets

Three levels, always in this order:

| `bucket` | `header.label` | Rows |
|---|---|---|
| `lease-components` | `By Lease Component` | one row per lease component billed or collected anywhere in scope, ordered by name, then a `Total` `subtotal` row |
| `all-properties` | `All Properties` | one row per property in scope, ordered by landlord then name, then a `Total` `subtotal` row |
| `landlord-{id}` | the landlord's name, or `Unassigned` | that landlord's properties only, then its own `Total` row |

The landlord buckets carry an extra header key:

```jsonc
"header": { "label": "Clementine Ndibo Holdings", "landlord": { "id": 4, "name": "Clementine Ndibo Holdings" } }
```

`landlord` is `null` where the properties have no landlord, and the bucket id is then `landlord-0`.

`lease-components` and `all-properties` are **the same money sliced differently**, so their totals are
identical by construction. The landlord buckets add back up to `all-properties`.

**A property with nothing billed and nothing collected still gets a row**, all zeros, rather than
being dropped — so the reader sees that it collected nothing instead of wondering where it went.

## Columns

The same five in every bucket; only the leading label column differs (`component` in the first
bucket, `property` in the others).

| `key` | `label` | `format` | Notes |
|---|---|---|---|
| `component` / `property` | `Component` / `Property` | `string` | |
| `billed` | `Billed` | `money` | |
| `collected` | `Collected` | `money` | Confirmed only |
| `collection_rate` | `Collection Rate %` | **`string`** | A preformatted percentage, e.g. `"86.48 %"`, carrying its own `color` |
| `variance` | `Variance` | `money` | **`collected − billed`**, so a shortfall is negative |

`collection_rate` is a **string, not a number**. The backend has already rounded and formatted it, and
attached the colour — render the value as given. The numeric rate is available on the bucket's
`summary.collection_rate` if you need to compute with it.

## The rule that matters: rates are recomputed, never averaged

**Every bucket's rate — including each `Total` row — is computed from that bucket's own summed
`billed` and `collected`.** It is never the mean of the row percentages above it.

Worked example, two properties under one landlord:

| Property | Billed | Collected | Row rate |
|---|---|---|---|
| Kahawa House | 11,475,900 | 8,345,000 | 72.72 % |
| Kahawa Side Tower | 400,000 | 0 | 0.00 % |
| **Total** | **11,875,900** | **8,345,000** | **70.27 %** |

The mean of `72.72` and `0.00` is `36.36`. The bucket reports **70.27 %**, because
`8,345,000 ÷ 11,875,900` is what the landlord actually collected. **If you ever re-derive a rate on
the frontend by averaging the rows, you will disagree with the backend.** Render the returned value.

## Colours

`collection_rate` uses the shared performance bands, the same ones
[Property Status](./property-status.md) uses for occupancy:

| Rate | `color` |
|---|---|
| above 80 % | `success` |
| 50 % to 80 %, both ends included | `warning` |
| below 50 % | `danger` |

80.00 % is `warning` (the success band is strictly above 80); 50.00 % is `warning`; 49.90 % is
`danger`. A bucket with nothing billed has no rate to judge, so the cell reads `0.00 %` and carries
**no `color` key at all** — absent means `none`, per the shared contract.

## Summary

Both bucket-level and report-level `summary` carry the same three scalars:

| Key | Meaning |
|---|---|
| `total_billed` | Billed in the period |
| `total_collected` | Confirmed collections in the period |
| `collection_rate` | `total_collected ÷ total_billed` as a **number** (e.g. `70.27`), recomputed from the totals |

The report-level `summary` equals the `all-properties` bucket's.

## Example response

```jsonc
{
  "data": {
    "header": {
      "property": null,
      "currency": { "code": "KES", "name": "Kenyan Shilling" },
      "period": { "from": "2026-05-01", "to": "2026-07-31" },
      "filters": { "landlord_id": null, "facility_id": null, "facility_type_id": null,
                   "properties_status": null, "period_from": "2026-05-01", "period_to": "2026-07-31" },
      "generated_at": "2026-08-17T09:12:44+00:00"
    },
    "report": [
      {
        "bucket": "lease-components",
        "header": { "label": "By Lease Component" },
        "fields": [ /* the five columns above, with `component` leading */ ],
        "items": [
          { "type": "normal", "component": { "value": "Rent" },
            "billed": { "value": 8210300.00 }, "collected": { "value": 7100200.00 },
            "collection_rate": { "value": "86.48 %", "color": "success" },
            "variance": { "value": -1110100.00 } },
          { "type": "normal", "component": { "value": "Service Charge" },
            "billed": { "value": 3145600.00 }, "collected": { "value": 1204800.00 },
            "collection_rate": { "value": "38.30 %", "color": "danger" },
            "variance": { "value": -1940800.00 } },
          { "type": "subtotal", "background_color": "secondary", "component": { "value": "Total" },
            "billed": { "value": 11355900.00 }, "collected": { "value": 8305000.00 },
            "collection_rate": { "value": "73.13 %", "color": "warning" },
            "variance": { "value": -3050900.00 } }
        ],
        "summary": { "total_billed": 11355900.00, "total_collected": 8305000.00, "collection_rate": 73.13 }
      },
      { "bucket": "all-properties", "header": { "label": "All Properties" }, "fields": [ /* … */ ],
        "items": [ /* one row per property + Total */ ],
        "summary": { "total_billed": 11355900.00, "total_collected": 8305000.00, "collection_rate": 73.13 } },
      { "bucket": "landlord-4",
        "header": { "label": "Clementine Ndibo Holdings", "landlord": { "id": 4, "name": "Clementine Ndibo Holdings" } },
        "fields": [ /* … */ ], "items": [ /* that landlord's properties + Total */ ],
        "summary": { "total_billed": 8210300.00, "total_collected": 6100200.00, "collection_rate": 74.30 } }
    ],
    "summary": { "total_billed": 11355900.00, "total_collected": 8305000.00, "collection_rate": 73.13 }
  }
}
```

Note the subtotal in the first bucket: `8305000 ÷ 11355900 = 73.13 %`, **not** the `62.39 %` mean of
`86.48` and `38.30`.

## Export

`…/property-collections/export?format=excel|pdf`, taking every filter the view endpoint takes. The
frontend exports exactly what is on screen by replaying the query string with `format` appended. The
export contains the same buckets, rows and cells as the view response.
