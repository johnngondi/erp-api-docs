# Supplier Contracts Report API

Domain: `Property Management > Reports > Suppliers`

Base route:

`/api/v1/app/{company}/property-management/reports`

The Supplier Contracts report is a **register of supplier contracts** (`facility_contracts`). It totals
them by expense type, lists them one per contract, and flags the ones about to lapse. It is
**read-only** and computed on the fly.

> **Read [the shared reports contract](./README.md) first.** This page documents what is specific to
> this report. The response envelope, the bucket keys, the field keys, the per-cell `{ value, color }`
> shape and the colour vocabulary live in the README and apply here unchanged.

## Endpoints

- `GET /reports/suppliers/supplier-contracts` — requires `view-supplier-contracts-report`
- `GET /reports/suppliers/supplier-contracts/export` — requires `export-supplier-contracts-report`.
  Takes a required `format` (`excel` | `pdf`) plus the same filters as the report; any other format,
  or none, returns `422` with a `format` error.
- Listed by [`GET /reports`](./README.md#discovering-reports--get-reports) as a top-level supplier
  report (it is not in a parent group).

The view and export permissions are **independent**.

## As at today

The register answers "which contracts do we have, and which need renewing now?", so it is measured
**from today**:

- The period filters (`period_from`/`period_to`) are **echoed in the header but do not narrow the
  register**. An expired contract is exactly the row someone needs to see, and it would fall outside
  any current period. `header.as_at` is today's date.
- **Status as shown.** Nothing moves a contract to `expired` when its end date passes, so an
  **`active` contract past its end date is shown, and filtered, as `expired`**. Any other stored status
  (`pending`, `suspended`, `expired`, `inactive`) is shown as stored.
- An open-ended contract (no `end_at`) never expires and is never tinted.

## Filters

The six [global filters](./README.md#global-filters) plus:

| Param | Type | Meaning when omitted |
|---|---|---|
| six global filters | see the shared contract | all properties (the period is echoed only) |
| `contract_status` | `pending` \| `active` \| `suspended` \| `expired` \| `inactive` | Any status |
| `expiring_within_days` | integer ≥ 0 | No expiry warning (expired contracts are still tinted) |
| `supplier_id` | `users.id` (the contract's `vendor_id`) | All suppliers |

- **`contract_status`** matches the status **as shown**: `expired` includes active contracts past
  their end date, and `active` excludes them.
- **`expiring_within_days`** only changes the tint. It never hides a contract.
- All filters narrow **both** buckets, so they always reconcile.

`header.filters` echoes all nine.

```
GET …/reports/suppliers/supplier-contracts?expiring_within_days=30
GET …/reports/suppliers/supplier-contracts?supplier_id=42&contract_status=active
```

## Header

- `property` — `{ id, name }` when a `facility_id` is given, otherwise `null`.
- `currency` — `{ code, name }` when every listed contract is in one currency; `null` when they differ.
- `as_at` — today's date (`Y-m-d`), the date expiry is measured from.
- `period` — the echoed period (not used to narrow).

## Buckets

Always both, in this order:

| `bucket` | `header.label` | One row per… |
|---|---|---|
| `overall` | `By Expense Type` | expense type, then a `Total` subtotal row |
| `overview` | `Supplier Contracts` | contract, then a subtotal row |

**Both buckets are built from the same contracts**, so the `overall` `Total` row's `contract_count`
and `total_value` equal the number of `overview` rows and the sum of their `contract_value`. The two
bucket `summary` objects are identical.

### `overall` columns

| Column | `key` | `format` |
|---|---|---|
| Expense Type | `expense_type` | string (contracts with no expense type are grouped as `Untyped`) |
| Contracts | `contract_count` | integer |
| Total Value | `total_value` | money, `grosstotal` |

Rows are ordered by `total_value` descending, then by name. The closing row is `Total`, with
`type: subtotal` and `background_color: secondary`.

### `overview` columns

| Column | `key` | `format` | Source |
|---|---|---|---|
| Supplier | `supplier` | string | the contract's vendor name |
| Property | `property` | string | the contract's property name |
| Expense Type | `expense_type` | string | the contract's expense type, or `Untyped` |
| Contract Value | `contract_value` | money | `facility_contracts.amount` as stored, in the contract's own currency |
| Start Date | `start_date` | string `Y-m-d` | `start_at` |
| End Date | `end_date` | string `Y-m-d` | `end_at`; `null` when open-ended |
| Status | `status` | string | the status as shown (see above); the cell carries its colour (`pending` warning, `active` success, `suspended` secondary, `expired` danger, `inactive` none) |
| Withholding Taxes | `withholding_taxes` | string | every tax in `facility_contracts.withholding_tax_ids`, by name, **comma-separated in one cell** (e.g. `WHT 5%, VAT WH 2%`); `null` when none |

Rows are ordered by supplier, then property, then start date. The closing row is a label
(`Total (N contracts)`) with `col_span: 3`, then `contract_value`; the other keys are omitted. It has
`type: subtotal` and `background_color: secondary`.

## Tinting

`overview` rows carry a `background_color`:

| When | `background_color` |
|---|---|
| The end date has passed (`end_at` before today) | `danger` |
| The contract ends within `expiring_within_days` of today, today included | `warning` |
| Otherwise, open-ended, or no `expiring_within_days` given | none (key omitted) |

`danger` wins over `warning`. Without `expiring_within_days`, only ended contracts are tinted.
`overall` rows are never tinted.

## Summary

The same object on both buckets and on `data.summary`:

| Key | Meaning |
|---|---|
| `contract_count` | Contracts listed |
| `total_value` | Sum of `contract_value` |
| `expiring_soon` | Contracts tinted `warning` |
| `expired` | Contracts tinted `danger` |

## Example response

`expiring_within_days=30`, as at 30 Jun 2026:

```jsonc
{
  "data": {
    "header": {
      "property": null,
      "currency": { "code": "KES", "name": "Kenya Shilling" },
      "as_at": "2026-06-30",
      "period": { "from": "2026-06-01", "to": "2026-06-30" },
      "filters": { "landlord_id": null, "facility_id": null, "facility_type_id": null, "properties_status": null,
                   "period_from": null, "period_to": null,
                   "contract_status": null, "expiring_within_days": 30, "supplier_id": null },
      "generated_at": "2026-06-30T12:00:00+03:00"
    },
    "report": [
      {
        "bucket": "overall",
        "header": { "label": "By Expense Type" },
        "fields": [ /* expense_type, contract_count, total_value */ ],
        "items": [
          { "expense_type": { "value": "Security" }, "contract_count": { "value": 2 }, "total_value": { "value": 8000 }, "type": "normal" },
          { "expense_type": { "value": "Cleaning" }, "contract_count": { "value": 1 }, "total_value": { "value": 2000 }, "type": "normal" },
          { "expense_type": { "value": "Total" }, "contract_count": { "value": 3 }, "total_value": { "value": 10000 }, "type": "subtotal", "background_color": "secondary" }
        ],
        "summary": { "contract_count": 3, "total_value": 10000, "expiring_soon": 1, "expired": 1 }
      },
      {
        "bucket": "overview",
        "header": { "label": "Supplier Contracts" },
        "fields": [ /* the eight columns above, in order */ ],
        "items": [
          { "supplier": { "value": "Guardforce" }, "property": { "value": "ACK Gardens" }, "expense_type": { "value": "Security" },
            "contract_value": { "value": 5000 }, "start_date": { "value": "2025-07-01" }, "end_date": { "value": "2026-06-15" },
            "status": { "value": "expired", "color": "danger" }, "withholding_taxes": { "value": "WHT 5%, VAT WH 2%" },
            "type": "normal", "background_color": "danger" },
          { "supplier": { "value": "Guardforce" }, "property": { "value": "Beta House" }, "expense_type": { "value": "Security" },
            "contract_value": { "value": 3000 }, "start_date": { "value": "2025-08-01" }, "end_date": { "value": "2026-07-20" },
            "status": { "value": "active", "color": "success" }, "withholding_taxes": { "value": "WHT 5%" },
            "type": "normal", "background_color": "warning" },
          { "supplier": { "value": "Sparkle Cleaners" }, "property": { "value": "ACK Gardens" }, "expense_type": { "value": "Cleaning" },
            "contract_value": { "value": 2000 }, "start_date": { "value": "2026-01-01" }, "end_date": { "value": null },
            "status": { "value": "active", "color": "success" }, "withholding_taxes": { "value": null },
            "type": "normal" },
          { "supplier": { "value": "Total (3 contracts)", "col_span": 3 }, "contract_value": { "value": 10000 },
            "type": "subtotal", "background_color": "secondary" }
        ],
        "summary": { "contract_count": 3, "total_value": 10000, "expiring_soon": 1, "expired": 1 }
      }
    ],
    "summary": { "contract_count": 3, "total_value": 10000, "expiring_soon": 1, "expired": 1 }
  }
}
```
