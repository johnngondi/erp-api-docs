# Management Contracts Report API

Domain: `Property Management > Reports > Landlords`

Base route:

`/api/v1/app/{company}/property-management/reports`

The Management Contracts report is the **register of management agreements with landlords**. It lists
every contract with its fee basis, rate, term and how many days it has left, and flags the contracts
that have ended or are about to. It is **read-only** and computed on the fly.

> **Read [the shared reports contract](./README.md) first.** This page only documents what is specific
> to this report — its two extra filters, its single bucket, and its expiry tinting. The response
> envelope, the bucket keys, the field keys, the per-cell `{ value, color }` shape, and the
> colour/format vocabulary all live in the README and apply here unchanged.

## Endpoints

- `GET /reports/landlords/management-contracts` — requires `view-management-contracts-report`
- `GET /reports/landlords/management-contracts/export` — requires
  `export-management-contracts-report`. Takes a required `format` (`excel` | `pdf`) plus the same
  filters as the report and downloads the generated file — see
  [the shared contract](./README.md#permissions--export). Any other `format`, or none, returns `422`
  with a `format` error.
- Listed by [`GET /reports`](./README.md#discovering-reports--get-reports).

The two permissions are **independent**: neither grants the other.

## Get the report

`GET /api/v1/app/{company}/property-management/reports/landlords/management-contracts`

### Filters (query params)

All filters are **optional** and flat. The six [global filters](./README.md#global-filters) apply; this
report adds two:

| Param | Type | Meaning when omitted |
|---|---|---|
| `landlord_id` | `users.id` | All landlords (global) |
| `facility_id` | `facilities.id` | All properties (global) |
| `facility_type_id` | `facility_types.id` | All property types (global) |
| `properties_status` | enum `App\Enums\Status` | Any property status (global) |
| `period_from` | date `Y-m-d` | Start of the current month (global) — echoed only, see below |
| `period_to` | date `Y-m-d` | End of the current month (global) — echoed only, see below |
| `contract_status` | enum `App\Enums\Status` (`active`, `expired`, `suspended`, …) | Any status |
| `expiring_within_days` | integer, `0` or more | No expiry warning is applied |

- **`contract_status`** matches the status the report **shows**, which treats an active contract past
  its end date as `expired` (see [status](#status)). An unknown value returns `422`.
- **`expiring_within_days`** is the warning window, counted from today, today included. A negative
  value returns `422`.
- **The period does not narrow the register.** The register is as at **today**, so a contract that
  ended before the period is still listed — it is exactly the row an agent needs to act on. The period
  is still resolved and echoed in `header.period`, like every report.

`header.filters` echoes every filter above, `null` included.

Example:

```
GET …/reports/landlords/management-contracts?landlord_id=4&contract_status=active&expiring_within_days=60
```

## Buckets

One bucket, always present:

| `bucket` | `header.label` | Rows |
|---|---|---|
| `overview` | `Management Contracts` | one row per contract, then a `subtotal` row |

Rows are sorted by landlord name, then property name, then contract start date. A property with more
than one contract has a row for each.

The closing `subtotal` row counts the contracts: a single `landlord` cell reading e.g. `3 contracts`
(or `1 contract`) with `col_span: 8`, and `background_color: secondary`. Its other keys are absent.

## Columns

| Column | `key` | `format` | Value |
|---|---|---|---|
| Landlord | `landlord` | string | The property's landlord |
| Property | `property` | string | The property the contract covers |
| Fee Basis | `fee_basis` | string | `percentage of collections`, `custom percentage of collections`, `flat fee` or `none` |
| Rate | `rate` | string | The percentage, e.g. `2.5%`; **`—`** for a flat fee or no fee |
| Start Date | `start_date` | string | e.g. `01 Jan, 2025` |
| End Date | `end_date` | string | e.g. `31 Dec, 2027`; `null` for an open-ended contract |
| Days to Expiry | `days_to_expiry` | integer | Whole days from today to the end date — **negative once ended**; `null` when open-ended |
| Status | `status` | string | The contract status — see [status](#status) |

## Tinting — computed server-side

The row's `background_color` says where a contract stands. **Render the returned tint; do not
recompute it from `days_to_expiry`.**

| Condition | `background_color` |
|---|---|
| The contract has ended (`days_to_expiry` below `0`) | `danger` |
| It ends within `expiring_within_days`, today included | `warning` |
| Otherwise, or no `expiring_within_days` was sent | *absent* (no tint) |

- **An ended contract is always `danger`**, whether or not `expiring_within_days` is sent.
- **Without `expiring_within_days` nothing is tinted `warning`.**
- The window is inclusive: with `expiring_within_days=44`, a contract 44 days out is `warning`; with
  `43` it is not. A contract ending today has `days_to_expiry: 0` and counts as expiring, not ended.
- An open-ended contract is never tinted.
- An absent `background_color` means `none`, per the shared contract — it is not a missing value.

## Status

Nothing in the system moves a contract to `expired` when its end date passes. So the report derives
it: **an `active` contract whose end date has passed is shown — and filtered — as `expired`**. Every
other stored status is shown as stored.

## Summary

The bucket's `summary` and the report-level `data.summary` carry the same keys:

| Key | Meaning |
|---|---|
| `contract_count` | Contracts listed |
| `expiring_soon` | Contracts tinted `warning` — `0` when no `expiring_within_days` is sent |
| `expired` | Contracts tinted `danger` |

## Example response

```jsonc
{
  "data": {
    "header": {
      "property": null,
      "period": { "from": "2026-07-01", "to": "2026-07-31" },
      "filters": {
        "landlord_id": null, "facility_id": null, "facility_type_id": null, "properties_status": null,
        "period_from": "2026-07-01", "period_to": "2026-07-31",
        "contract_status": null, "expiring_within_days": 60
      },
      "generated_at": "2026-08-17T09:12:44+00:00",
      "currency": { "code": "KES", "name": "Kenyan Shilling" }
    },
    "report": [
      {
        "bucket": "overview",
        "header": { "label": "Management Contracts" },
        "fields": [
          { "label": "Landlord", "key": "landlord", "format": "string", "type": "normal", "weight": "font-normal", "background_color": "none", "alignment": "left", "visible": true, "togglable": false },
          /* …property, fee_basis, rate, start_date, end_date… */
          { "label": "Days to Expiry", "key": "days_to_expiry", "format": "integer", "type": "normal", "weight": "font-normal", "background_color": "none", "alignment": "right", "visible": true, "togglable": false },
          { "label": "Status", "key": "status", "format": "string", "type": "normal", "weight": "font-normal", "background_color": "none", "alignment": "left", "visible": true, "togglable": false }
        ],
        "items": [
          { "landlord": { "value": "Clementine Ndibo Holdings" }, "property": { "value": "KAHAWA HOUSE" },
            "fee_basis": { "value": "percentage of collections" }, "rate": { "value": "2.5%" },
            "start_date": { "value": "01 Jan, 2025" }, "end_date": { "value": "31 Dec, 2027" },
            "days_to_expiry": { "value": 501 }, "status": { "value": "active" }, "type": "normal" },
          { "landlord": { "value": "Rift Valley Estates" }, "property": { "value": "COFFEE PLAZA" },
            "fee_basis": { "value": "flat fee" }, "rate": { "value": "—" },
            "start_date": { "value": "01 Sep, 2024" }, "end_date": { "value": "30 Sep, 2026" },
            "days_to_expiry": { "value": 44 }, "status": { "value": "active" },
            "type": "normal", "background_color": "warning" },
          { "landlord": { "value": "Westlands Trust" }, "property": { "value": "Kahawa House Side Tower" },
            "fee_basis": { "value": "percentage of collections" }, "rate": { "value": "3%" },
            "start_date": { "value": "01 Jan, 2023" }, "end_date": { "value": "30 Jun, 2026" },
            "days_to_expiry": { "value": -48 }, "status": { "value": "expired" },
            "type": "normal", "background_color": "danger" },
          { "landlord": { "value": "3 contracts", "col_span": 8 }, "type": "subtotal", "background_color": "secondary" }
        ],
        "summary": { "contract_count": 3, "expiring_soon": 1, "expired": 1 }
      }
    ],
    "summary": { "contract_count": 3, "expiring_soon": 1, "expired": 1 }
  }
}
```

Reading the example, generated on 17 Aug 2026: COFFEE PLAZA ends in 44 days, inside the 60-day window,
so it is `warning`; re-sent with `expiring_within_days=30` it would carry no tint. Side Tower ended 48
days ago, so it is `danger`, shows `-48`, and reads `expired` although it was stored as active.
