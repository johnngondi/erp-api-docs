# Billing & Collection Trend Report API

Domain: `Property Management > Reports > Tenants`

Base route:

`/api/v1/app/{company}/property-management/reports`

Billing & Collection Trend answers **how billing and collection moved over time**, property by
property, from the first legacy (EPMAS) invoice in December 2018 to today. Each row is one period of
the chosen cycle: what was brought forward, billed, collected, the collection rate and what was
carried forward. It is **read-only** and computed on the fly.

> **Read [the shared reports contract](./README.md) first.** This page documents what is specific to
> this report. The response envelope, the bucket keys, the field keys, the per-cell `{ value, color }`
> shape and the colour vocabulary live in the README and apply here unchanged.

## Endpoints

- `GET /reports/tenants/billing-collection-trend` — requires `view-billing-collection-trend-report`
- `GET /reports/tenants/billing-collection-trend/export` — requires
  `export-billing-collection-trend-report`. Takes a required `format` (`excel` | `pdf`); any other
  value, or none, returns `422` with a `format` error.
- Listed by [`GET /reports`](./README.md#discovering-reports--get-reports).

The two permissions are **independent**, and neither is shared with any other report.

## Where the figures come from

The ERP went live on **1 October 2026**. Each lease came across from EPMAS with one opening balance
dated 30 September 2026, so the live ledger holds nothing earlier. The legacy ledger was imported
separately into two **reference-only** tables, `lease_history_billings` and
`lease_history_collections`, which no other report, balance or statement reads. This report joins the
two worlds into one series:

| Column | Legacy history (to 30 Sep 2026) | Live ERP ledger |
|---|---|---|
| **Billed** | `lease_history_billings` rows of type `invoice` and `credit_note` (credit notes are negative) | `lease_billings`, **excluding opening-balance documents** (any billing whose `invoice_id` or `credit_note_id` is on a non-deleted `lease_opening_balances` row) |
| **Collected** | `lease_history_collections` with status `confirmed` | `lease_collections` with status `confirmed` |
| **Cutover Reconciliation** | `lease_history_billings` rows of type `cutover_reconciliation` | — |

- A `cancelled` collection, legacy or live, is never counted.
- Opening-balance documents are left out of **Billed** because the legacy history already carries the
  billing they summarise. Counting both would bill the same debt twice.
- Every other live billing counts at its own date, including one back-dated before October 2026.
- Everything reaches its property through the lease (`leases.facility_id`).

### Cutover reconciliation

The legacy ledger and the opening balances do not agree to the shilling: rounding, VAT proration and
legacy adjustments leave a small gap per lease and component. The importer writes one
`cutover_reconciliation` row per lease and component, dated **30 Sep 2026 23:59:59**, equal to:

`opening balance − (legacy billed − legacy confirmed collections)`

It is shown in its **own column** and is never folded into Billed, so the trend of what was really
billed stays clean and the gap stays visible. With it, the series closes exactly on the live ledger:
**at any date on or after 30 September 2026, `balance_cf` equals the live tenant balance** (live
billed less live confirmed collections) that every other report computes. A non-zero reconciliation
in September 2026 is therefore expected, not an error.

## Filters

The six [global filters](./README.md#global-filters) plus two:

| Param | Type | Meaning when omitted |
|---|---|---|
| six global filters | see the shared contract | all / current month |
| `report_cycle` | `monthly` \| `quarterly` \| `annually` | **`monthly`** |
| `lease_component_id` | integer, `lease_components.id` | all components |

`report_cycle` has no `weekly` option (eight years of weeks is not a trend anyone reads); `weekly`
returns `422`.

With `lease_component_id`, every column is narrowed to that component. **Legacy collections that
were never allocated to a component** (`lease_component_id` null) count as **Rent**, the component
the importer charges them to when it computes the cutover reconciliation, so a Rent-filtered
`balance_cf` still equals the live Rent balance. Under any other component they drop out; without
the filter they count in full.

`header.filters` echoes all eight, `null` included.

```
GET …/reports/tenants/billing-collection-trend?period_from=2019-01-01&period_to=2026-12-31&report_cycle=annually
```

A useful trend needs a wide window; the default is still the current month, as for every report.

## Currency

Nothing is ever summed across currencies.

- Each **property bucket** is in that property's reporting currency
  (`facilities.reporting_currency_id`). A legacy row is converted from its own `currency_id`, a live
  collection from its `currency_id`, and a live billing from its lease's `currency_id`, at the
  currency's current rate. A row already in the reporting currency, or with no currency, is not
  converted.
- The **`all-properties` bucket** is in the properties' shared reporting currency when they all have
  the same one, otherwise in the company's default currency.
- Every bucket's `header.currency` is `{ code, name }` for the currency its money is in, and the
  top-level `header.currency` is the `all-properties` bucket's.

## Buckets

Always in this order:

| `bucket` | `header` | Rows |
|---|---|---|
| `all-properties` | `label: "All Properties"`, `currency` | one row per period, every property in scope added together, then a `Total` `grosstotal` row |
| `property-{id}` | `label` (the property name), `property { id, name }`, `currency` | one row per period for that property, then a `Total` `subtotal` row |

Property buckets follow the `all-properties` bucket ordered by property name. `facility_id` narrows
the scope but does **not** change the bucket set: `all-properties` then holds that one property.

**Every period gets a row, even an empty one**, so a trend line has no gaps. A property with no
activity at all still gets its bucket, all zeros.

Each period row also carries `period_from` and `period_to` (row metadata, not cells), so you never
parse the label.

## Columns

| `key` | `label` | `format` | Notes |
|---|---|---|---|
| `period` | `Period` | `string` | e.g. `Jan 2019`, `Q1 2019`, `2019`; `Total` on the closing row |
| `balance_bf` | `Balance B/F` | `money` | Billed + reconciliation − collected, everything before the period |
| `billed` | `Billed` | `money` | |
| `cutover_reconciliation` | `Cutover Reconciliation` | `money` | Non-zero only in the period holding 30 Sep 2026 |
| `collected` | `Collected` | `money` | Confirmed only |
| `collection_rate` | `Collection Rate %` | `string` | `collected ÷ billed`, a preformatted percentage carrying its colour |
| `balance_cf` | `Balance C/F` | `money` | `balance_bf + billed + cutover_reconciliation − collected` |

`collection_rate` is coloured with the shared bands (see
[Colouring occupancy, collection and budget health](./README.md#colouring-occupancy-collection-and-budget-health)):
above 80% `success`, 50–80% `warning`, below 50% `danger`, none where nothing was billed. A period
can exceed 100% when tenants paid arrears.

The `Total` row's `balance_bf` is the first period's, its `balance_cf` the last period's, and its rate
is recomputed from the summed billed and collected — never averaged across the rows.

Periods are calendar-anchored and clamped to the window, as in
[Collections Summary](./collections-summary.md#periods-are-calendar-anchored).

## Summary

Both bucket-level and report-level (`summary` = the `all-properties` bucket's):

| Key | Meaning |
|---|---|
| `balance_bf` | Balance brought forward at `period_from` |
| `total_billed` | Billed in the window |
| `total_cutover_reconciliation` | Cutover reconciliation in the window |
| `total_collected` | Confirmed collections in the window |
| `collection_rate` | `total_collected ÷ total_billed` as a number, recomputed from the totals |
| `balance_cf` | Balance carried forward at `period_to` |
| `period_count` | Period rows the cycle produced |

## Example response — one property bucket

```jsonc
{
  "bucket": "property-12",
  "header": { "label": "Kahawa House", "property": { "id": 12, "name": "Kahawa House" }, "currency": { "code": "KES", "name": "Kenya Shilling" } },
  "fields": [
    { "label": "Period", "key": "period", "format": "string", "type": "normal", "weight": "font-normal", "background_color": "none", "alignment": "left", "visible": true, "togglable": false },
    { "label": "Balance B/F", "key": "balance_bf", "format": "money", "type": "normal", "weight": "font-normal", "background_color": "none", "alignment": "right", "visible": true, "togglable": false },
    { "label": "Billed", "key": "billed", "format": "money", "type": "normal", "weight": "font-normal", "background_color": "none", "alignment": "right", "visible": true, "togglable": false },
    { "label": "Cutover Reconciliation", "key": "cutover_reconciliation", "format": "money", "type": "normal", "weight": "font-normal", "background_color": "none", "alignment": "right", "visible": true, "togglable": true },
    { "label": "Collected", "key": "collected", "format": "money", "type": "normal", "weight": "font-normal", "background_color": "none", "alignment": "right", "visible": true, "togglable": false },
    { "label": "Collection Rate %", "key": "collection_rate", "format": "string", "type": "normal", "weight": "font-normal", "background_color": "none", "alignment": "right", "visible": true, "togglable": false },
    { "label": "Balance C/F", "key": "balance_cf", "format": "money", "type": "normal", "weight": "font-medium", "background_color": "none", "alignment": "right", "visible": true, "togglable": false }
  ],
  "items": [
    { "type": "normal", "period_from": "2026-08-01", "period_to": "2026-09-30",
      "period": { "value": "Q3 2026" }, "balance_bf": { "value": 1200000.00 }, "billed": { "value": 900000.00 },
      "cutover_reconciliation": { "value": -350.00 }, "collected": { "value": 850000.00 },
      "collection_rate": { "value": "94.44 %", "color": "success" }, "balance_cf": { "value": 1249650.00 } },
    { "type": "normal", "period_from": "2026-10-01", "period_to": "2026-10-31",
      "period": { "value": "Q4 2026" }, "balance_bf": { "value": 1249650.00 }, "billed": { "value": 300000.00 },
      "cutover_reconciliation": { "value": 0.00 }, "collected": { "value": 120000.00 },
      "collection_rate": { "value": "40.00 %", "color": "danger" }, "balance_cf": { "value": 1429650.00 } },
    { "type": "subtotal", "background_color": "secondary",
      "period": { "value": "Total" }, "balance_bf": { "value": 1200000.00 }, "billed": { "value": 1200000.00 },
      "cutover_reconciliation": { "value": -350.00 }, "collected": { "value": 970000.00 },
      "collection_rate": { "value": "80.83 %", "color": "success" }, "balance_cf": { "value": 1429650.00 } }
  ],
  "summary": { "balance_bf": 1200000.00, "total_billed": 1200000.00, "total_cutover_reconciliation": -350.00,
               "total_collected": 970000.00, "collection_rate": 80.83, "balance_cf": 1429650.00, "period_count": 2 }
}
```

## Export

`…/billing-collection-trend/export?format=excel|pdf`, taking every filter the view endpoint takes, so
the export has exactly the rows on screen. The frontend exports the current view by replaying the
query string with `format` appended.
