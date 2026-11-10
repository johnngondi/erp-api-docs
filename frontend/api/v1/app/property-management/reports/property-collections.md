# Property Collections Report API

Domain: `Property Management > Reports > Tenants > Collections Reports`

Base route:

`/api/v1/app/{company}/property-management/reports`

Property Collections is a **receipt listing**: every receipt that paid a lease in the period, one row
per receipt per lease, with what it paid against each lease component, the VAT on it and its total.
It answers **who paid, when, under which receipt, and for what**. Its columns follow the legacy
"Property Collections" report: Property | Tenant | Txn Date | Ref. | Rent | S.Charge | Parking | VAT |
Total, plus a Payment Date. It is **read-only** and computed on the fly.

> **Read [the shared reports contract](./README.md) first.** This page documents what is specific to
> this report. The response envelope, the bucket keys, the field keys, the per-cell `{ value, color }`
> shape and the colour vocabulary live in the README and apply here unchanged.

> **Changed 2026-11-02.** This report used to compare billed against collected in three buckets
> (`lease-components`, `all-properties`, `landlord-{id}`) with a collection rate and variance. That
> shape repeated [Collections Summary](./collections-summary.md) and is gone. Billed against
> collected per lease lives in [Billings & Collections](./billings-and-collections.md).

> **Changed 2026-11-05.** `transaction_date` (Txn Date) is now the **collection's** date
> (`lease_collections.transaction_date`), the date that puts a row in the period. The receipt's own
> date (`facility_receipts.transaction_date`), which Txn Date used to show, moved to the new
> `payment_date` (Payment Date) column. A row is now one receipt on one lease **on one collection
> date**, and the Total row's label spans one more column.

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

- **A row is one receipt on one lease on one collection date.** The report reads
  `lease_collections` whose status is **`confirmed`** and whose own `transaction_date` falls in the
  period, and groups them by the receipt they were allocated from
  (`lease_collections.receipt_allocation_id` → `facility_receipt_allocations.receipt_id` →
  `facility_receipts`), by lease and by the collection's date (the day of
  `lease_collections.transaction_date`). A receipt that pays two leases is two rows; a receipt that
  pays Rent and Service Charge on one lease the same day is **one** row with both amounts. A receipt
  whose money reaches a lease on two days (say, a credit applied to a later invoice) is one row per day.
- A `cancelled` collection is not money collected and never appears.
- **Component cells are net** (pre-tax): the sum of `lease_collections.amount` for that component on
  that receipt and lease. A collection reaches its component through its lease item
  (`lease_collections.lease_item_component_id` → `lease_item_components.lease_component_id`).
- **`vat` is the stored tax, summed** (`lease_collections.tax`). It is never recomputed as a
  percentage of the net, so a zero-rated or partly taxed receipt shows exactly what was posted.
- **`total` is the stored total, summed** (`lease_collections.total`), so on every row
  `total = component cells + vat`.
- `transaction_date` is the **collection's** date (`lease_collections.transaction_date`), the date
  the period is applied to. `payment_date` and `reference` are the **receipt's** own
  (`facility_receipts.transaction_date` and `facility_receipts.transaction_number`). The two dates
  are normally the same day; they differ when a receipt is allocated to the lease after it was paid.
- Rows reach their property through the lease (`leases.facility_id`).

[Collections Summary](./collections-summary.md) and
[Legal Fee Collection](./legal-fee-collection.md) read the same source, so they reconcile for the
same window: this report's `summary.total_collected` equals Collections Summary's.

## Filters

The [global filters](./README.md#global-filters) — `landlord_ids`, `facility_ids`, `buckets` and the
rest — plus the [block / wing / floor lists](#blocks-wings-and-floors), and nothing else.
`header.filters` echoes every filter, `null` included.

```
GET …/reports/tenants/collections-reports/property-collections?facility_ids[]=1&period_from=2026-05-01&period_to=2026-07-31
```

Selecting properties (`facility_ids`) narrows the scope; it does **not** change the bucket shape.
With one property selected you still get `overall` and that one `property-{id}` bucket, holding the
same rows.

Money is shown as stored, in each property's reporting currency — nothing is converted.
With several properties in scope, `header.currency` is set when they all share one reporting
currency, else `null`; with exactly one, the currency is on `header.property.currency` as in every
report. Each `property-{id}` bucket carries its own `header.currency` either way.

### Blocks, wings and floors

With exactly one property selected, `facility_block_ids`, `facility_wing_ids` and
`facility_floor_ids` narrow the report to part of it (several nodes are a union). Each lease counts
**whole** under the node of its **primary space** — the space of the item holding Rent, else
Service Charge, else the lowest-id spaced item. A lease with no space is in the property run
and out of every block / wing / floor run, so the block runs plus those leases add up to the property run.
Rules, `422`s and the add-up guarantee:
[Splitting by block / wing / floor](./README.md#splitting-by-block--wing--floor). The pickers
are filled from [`GET …/reports/filter-options/structure`](./README.md#picking-blocks-wings-and-floors).

## Buckets

Always in this order:

| `bucket` | `header.label` | Rows |
|---|---|---|
| `overall` | `Overall` | always first: every receipt row across the scope, then a `Total` `subtotal` row |
| `property-{id}` | the property's name | one per property in scope, ordered by property name: that property's receipt rows, then its own `Total` row |

A `property-{id}` bucket carries two extra header keys:

```jsonc
"header": {
  "label": "Kahawa House",
  "property": { "id": 7, "name": "Kahawa House" },
  "currency": { "code": "KES", "name": "Kenya Shilling" }   // that property's reporting currency, or null
}
```

`overall`'s header is `{ "label": "Overall" }` only. Every property in scope gets a bucket, so a
property that collected nothing in the period shows just its `Total` row of zeros. The
`property-{id}` buckets add back up to `overall`.

Pick buckets with [`buckets`](./README.md#global-filters), e.g. `buckets[]=property-7` for one
property's listing alone, or `buckets[]=overall`.

### Row order

Default row order (`header.sort.default`):

| `bucket` | Default order |
|---|---|
| `overall` | `["property", "transaction_date", "tenant"]` — property name, then collection date (oldest first), then tenant name |
| `property-{id}` | `["transaction_date", "tenant"]` |

Names compare case-insensitively with blanks last; ties keep the receipt order. Re-sort any bucket
with [`sort`](./README.md#sorting-rows--sort), e.g. `sort[overall]=-transaction_date` for the newest
collections first, `sort[overall]=-payment_date` for the newest receipts, or
`sort[property-7]=-total` for the largest; the `Total` row stays last.
[`columns`](./README.md#choosing-columns--columns) picks the columns shown and exported.

## Columns

| `key` | `label` | `format` | Notes |
|---|---|---|---|
| `property` | `Property` | `string` | **`overall` only** — the `property-{id}` buckets omit it |
| `tenant` | `Tenant` | `string` | The lease's tenant (`leases.user_id` → `users.name`) |
| `lease_id` | `Lease` | `id` | Togglable, **hidden by default** (`visible: false`). For drill-down to the lease |
| `transaction_date` | `Txn Date` | `string` | The collection's date (`lease_collections.transaction_date`), `YYYY-MM-DD` |
| `payment_date` | `Payment Date` | `string` | The receipt's date (`facility_receipts.transaction_date`), `YYYY-MM-DD` |
| `reference` | `Ref.` | `string` | The receipt's transaction number |
| `component_{id}` | the component's name | `money` | **Dynamic**, see below |
| `vat` | `VAT` | `money` | Summed stored tax |
| `total` | `Total` | `money` | Summed stored total, `weight: font-medium` |

**Component columns are generated at runtime**, one per lease component that has a collection
anywhere in the report, spliced in **before `vat`** and ordered by component id (Rent, Service
Charge, Parking, … in the seeded catalogue). Each is `togglable: true` and carries `component_id`:

```jsonc
{ "label": "Service Charge", "key": "component_2", "format": "money", "type": "normal",
  "alignment": "right", "togglable": true, "visible": true, "sortable": true, "component_id": 2 }
```

The set is the **same in every bucket** of one response, so a property that collected no Parking
still has a Parking column (all zeros). Every row carries every component key; a component the
receipt did not pay is `0`. Never hardcode the component columns — read them from `fields`.

### The `Total` row

Each bucket closes on a `subtotal` row (`background_color: secondary`) summing every money column.
Its label cell spans the text columns via `col_span`, and the row **omits** the covered keys:

| Bucket | Label cell | `col_span` | Omitted keys |
|---|---|---|---|
| `overall` | `property` = `Total` | 6 | `tenant`, `lease_id`, `transaction_date`, `payment_date`, `reference` |
| `property-{id}` | `tenant` = `Total` | 5 | `lease_id`, `transaction_date`, `payment_date`, `reference` |

The span counts `lease_id` even while it is hidden; per the shared contract, a renderer (and the
exports) reduce the span to the visible columns it covers.

## Summary

| Key | Bucket `summary` | Report `summary` | Meaning |
|---|---|---|---|
| `total_collected` | yes | yes | Sum of the rows' `total` — confirmed collections in the period |
| `receipt_count` | yes | yes | Rows listed (receipt + lease + collection date) |
| `property_count` | — | yes | Properties in scope, i.e. `property-{id}` buckets |

The report-level `total_collected` and `receipt_count` equal the `overall` bucket's.

## Example response

```jsonc
{
  "data": {
    "header": {
      "property": null,
      "properties": [],
      "landlord": null,
      "landlords": [],
      "blocks": [], "wings": [], "floors": [],
      "currency": { "code": "KES", "name": "Kenya Shilling" },
      "period": { "from": "2026-07-01", "to": "2026-07-31" },
      "filters": { "landlord_id": null, "facility_id": null, "facility_type_id": null,
                   "properties_status": null, "period_from": "2026-07-01", "period_to": "2026-07-31",
                   "landlord_ids": [], "facility_ids": [] /* , the block / wing / floor lists */ },
      "generated_at": "2026-11-02T09:12:44+00:00"
    },
    "report": [
      {
        "bucket": "overall",
        "header": {
          "label": "Overall",
          "sort": { "applied": ["property", "transaction_date", "tenant"], "default": ["property", "transaction_date", "tenant"], "source": "default" }
        },
        "fields": [
          { "label": "Property", "key": "property", "format": "string", "alignment": "left" /* , … */ },
          { "label": "Tenant", "key": "tenant", "format": "string", "alignment": "left" },
          { "label": "Lease", "key": "lease_id", "format": "id", "alignment": "left", "togglable": true, "visible": false },
          { "label": "Txn Date", "key": "transaction_date", "format": "string", "alignment": "left" },
          { "label": "Payment Date", "key": "payment_date", "format": "string", "alignment": "left" },
          { "label": "Ref.", "key": "reference", "format": "string", "alignment": "left" },
          { "label": "Rent", "key": "component_1", "format": "money", "alignment": "right", "togglable": true, "component_id": 1 },
          { "label": "Service Charge", "key": "component_2", "format": "money", "alignment": "right", "togglable": true, "component_id": 2 },
          { "label": "Parking", "key": "component_3", "format": "money", "alignment": "right", "togglable": true, "component_id": 3 },
          { "label": "VAT", "key": "vat", "format": "money", "alignment": "right" },
          { "label": "Total", "key": "total", "format": "money", "alignment": "right", "weight": "font-medium" }
        ],
        "items": [
          { "type": "normal", "property": { "value": "Kahawa House" }, "tenant": { "value": "Acme Traders Ltd" },
            "lease_id": { "value": 31 }, "transaction_date": { "value": "2026-07-05" }, "payment_date": { "value": "2026-07-05" }, "reference": { "value": "RCT-1001" },
            "component_1": { "value": 100000.00 }, "component_2": { "value": 25000.00 }, "component_3": { "value": 0.0 },
            "vat": { "value": 20000.00 }, "total": { "value": 145000.00 } },
          { "type": "normal", "property": { "value": "Kahawa House" }, "tenant": { "value": "Beta Stores" },
            "lease_id": { "value": 34 }, "transaction_date": { "value": "2026-07-09" }, "payment_date": { "value": "2026-06-28" }, "reference": { "value": "RCT-1007" },
            "component_1": { "value": 0.0 }, "component_2": { "value": 0.0 }, "component_3": { "value": 5000.00 },
            "vat": { "value": 800.00 }, "total": { "value": 5800.00 } },
          { "type": "normal", "property": { "value": "Mvule Court" }, "tenant": { "value": "Jane Wanjiru" },
            "lease_id": { "value": 52 }, "transaction_date": { "value": "2026-07-02" }, "payment_date": { "value": "2026-07-02" }, "reference": { "value": "MPESA-QH72K" },
            "component_1": { "value": 60000.00 }, "component_2": { "value": 0.0 }, "component_3": { "value": 0.0 },
            "vat": { "value": 0.0 }, "total": { "value": 60000.00 } },
          { "type": "subtotal", "background_color": "secondary",
            "property": { "value": "Total", "col_span": 6 },
            "component_1": { "value": 160000.00 }, "component_2": { "value": 25000.00 }, "component_3": { "value": 5000.00 },
            "vat": { "value": 20800.00 }, "total": { "value": 210800.00 } }
        ],
        "summary": { "total_collected": 210800.00, "receipt_count": 3 }
      },
      {
        "bucket": "property-7",
        "header": { "label": "Kahawa House", "property": { "id": 7, "name": "Kahawa House" },
                    "currency": { "code": "KES", "name": "Kenya Shilling" }, "sort": { /* … */ } },
        "fields": [ /* as overall, without `property` */ ],
        "items": [
          /* the two Kahawa House rows, without `property` */
          { "type": "subtotal", "background_color": "secondary",
            "tenant": { "value": "Total", "col_span": 5 },
            "component_1": { "value": 100000.00 }, "component_2": { "value": 25000.00 }, "component_3": { "value": 5000.00 },
            "vat": { "value": 20800.00 }, "total": { "value": 150800.00 } }
        ],
        "summary": { "total_collected": 150800.00, "receipt_count": 2 }
      },
      {
        "bucket": "property-9",
        "header": { "label": "Mvule Court", "property": { "id": 9, "name": "Mvule Court" },
                    "currency": { "code": "KES", "name": "Kenya Shilling" }, "sort": { /* … */ } },
        "fields": [ /* … */ ],
        "items": [ /* one row + Total */ ],
        "summary": { "total_collected": 60000.00, "receipt_count": 1 }
      }
    ],
    "summary": { "total_collected": 210800.00, "receipt_count": 3, "property_count": 2 }
  }
}
```

The first row is one receipt that paid Rent 100,000 and Service Charge 25,000 net plus VAT 20,000,
so it is one row totalling 145,000. The second row's receipt was paid on 28 June and allocated to the
lease on 9 July, so it lists under July with its June `payment_date`. No cell in this report carries a `color`.

## Export

`…/property-collections/export?format=excel|pdf`, taking every filter the view endpoint takes
(`buckets`, `sort` and `columns` included). The frontend exports exactly what is on screen by
replaying the query string with `format` appended. The export contains the same buckets, rows and
cells as the view response, printing only the visible columns (so `lease_id` is left out unless
`columns[...]` lists it).
