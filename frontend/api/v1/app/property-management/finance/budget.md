# Facility Budgets API

Domain: `Property Management > Finance`

Base route:

`/api/v1/app/{company}/property-management/finance/budgets`

## Overview

A facility budget is **system-generated and manually tunable**. It is not freely
created or deleted through the API — instead it is generated automatically and
its line amounts are edited in place.

A budget covers a single period (driven by the `facility_budget_cycle` setting)
and is composed of:

- **Income items** — split into:
  - **Commercial units** — auto-billed lease components at facility level
    (`facility_residential_unit_id = null`), e.g. Rent, Service Charge, Parking.
  - **Residential units** — auto-billed lease components per residential unit
    type (`facility_residential_unit_id` set).
  - **Utilities** — facility-wide utility components (not tied to a unit),
    seeded at `0`.
- **Expense items** — one per facility expense type, defaulting to `0`.

Estimation and lifecycle are documented in
[Budget Settings](../settings.md#budget-settings).

## Endpoints

- `GET /finance/budgets` — list budgets
- `POST /finance/budgets/generate` — queue generation for a facility with no current budget
- `GET /finance/budgets/{budget}` — show a budget with grouped items
- `PATCH /finance/budgets/{budget}/income-items/{incomeItem}` — edit an income line amount
- `PATCH /finance/budgets/{budget}/expense-items/{expenseItem}` — edit an expense line amount

## Authorization

| Action | Permission |
|---|---|
| List / show | `view-facility-budget` |
| Generate | `generate-facility-budget` |
| Update items | `update-facility-budget` |

## List Budgets

`GET /finance/budgets`

Supported query params:

- Filters:
  - `filter[facility_id]`
  - `filter[status]` — `ok`, `at-risk` (from `BudgetStatus`)
  - `filter[period]=YYYY-MM-DD` — budgets whose period contains the given date
- Sort: `sort=id,period_start,period_end,income_budget,expense_budget,created_at`
- Pagination: `per_page`, `page`

List items return budget headers (no nested line items).

## Show Budget

`GET /finance/budgets/{budget}`

Returns the header plus grouped items.

```json
{
  "data": {
    "id": 12,
    "facility_id": 4,
    "income_budget": "1200000.00000",
    "income_actual": "0.00000",
    "income_budget_status": { "value": "ok", "color": "success" },
    "expense_budget": "300000.00000",
    "expense_actual": "0.00000",
    "expense_budget_status": { "value": "ok", "color": "success" },
    "status": { "value": "ok", "color": "success" },
    "period_start": { "raw": "2026-06-15T00:00:00.000000Z", "formatted": "15 Jun, 2026" },
    "period_end": { "raw": "2026-12-31T00:00:00.000000Z", "formatted": "31 Dec, 2026" },
    "currency": { "id": 1, "code": "KES" },
    "income": {
      "commercial": [
        {
          "id": 101,
          "facility_residential_unit_id": null,
          "lease_component_id": 1,
          "budgeted_amount": "900000.00000",
          "actual_amount": "0.00000",
          "budget_status": { "value": "ok", "color": "success" },
          "lease_component": { "id": 1, "name": "Rent" }
        }
      ],
      "residential": [
        {
          "facility_residential_unit_id": 7,
          "title": "2 Bedroom",
          "items": [
            {
              "id": 140,
              "facility_residential_unit_id": 7,
              "lease_component_id": 1,
              "budgeted_amount": "300000.00000",
              "actual_amount": "0.00000",
              "budget_status": { "value": "ok", "color": "success" },
              "lease_component": { "id": 1, "name": "Rent" }
            }
          ]
        }
      ],
      "utilities": [
        {
          "id": 160,
          "facility_residential_unit_id": null,
          "lease_component_id": 6,
          "budgeted_amount": "0.00000",
          "actual_amount": "0.00000",
          "budget_status": { "value": "ok", "color": "success" },
          "lease_component": { "id": 6, "name": "Electricity" }
        }
      ]
    },
    "expenses": [
      {
        "id": 200,
        "expense_type_id": 3,
        "budgeted_amount": "0.00000",
        "actual_amount": "0.00000",
        "budget_status": { "value": "ok", "color": "success" },
        "expense_type": { "id": 3, "name": "Repairs and Maintenance" }
      }
    ]
  }
}
```

## Generate a Budget

`POST /finance/budgets/generate`

Use when a facility has **no current budget** (e.g. a renewal, or a facility
created before budgeting existed). Generation runs in a queued job.

Payload:

| Field | Required | Type | Notes |
|---|---|---|---|
| `facility_id` | Yes | integer | Must exist in `facilities.id` |
| `period_start` | No | date | Defaults to today; the period ends at the close of the cycle containing it |

Behaviour:

- Returns `422` with a `facility_id` message if the facility already has a
  budget covering the period.
- Otherwise returns `{ "message": "Budget generation has been queued for this facility." }`.

> Budgets are also generated automatically when a facility is created, and
> re-synced when its spaces, parkings or residential unit types change.

## Update Income Item

`PATCH /finance/budgets/{budget}/income-items/{incomeItem}`

| Field | Required | Type | Notes |
|---|---|---|---|
| `budgeted_amount` | Yes | numeric (`>= 0`) | New budgeted amount for the line |

Recomputes the budget's `income_budget` header total. Returns the updated item
under `income_item`. A `404` is returned if the item does not belong to the
budget.

## Update Expense Item

`PATCH /finance/budgets/{budget}/expense-items/{expenseItem}`

| Field | Required | Type | Notes |
|---|---|---|---|
| `budgeted_amount` | Yes | numeric (`>= 0`) | New budgeted amount for the line |

Recomputes the budget's `expense_budget` header total. Returns the updated item
under `expense_item`. A `404` is returned if the item does not belong to the
budget.

## Statuses

- `BudgetStatus` (budget header `status`): `ok`, `at-risk`.
- `BudgetItemStatus` (item and header income/expense status): `ok`, `over`, `under`.

A budget is flagged `at-risk` by the `budgets:monitor` job when actual
expenditure exceeds, or actual income falls below, the period-prorated budget by
more than `budget_at_risk_threshold_percent` (default `10`). Actuals are sourced
from lease collections (income) and facility expenses (expense).

## Errors

Use shared behavior in `docs/frontend/app/README.md`:

- Render `4xx` messages and validation details.
- Show generic fallback for `5xx`.
