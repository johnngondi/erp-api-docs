# Property Budget Settings API

Domain: `Property Management > Properties`

Base route:

`/api/v1/app/{company}/property-management/facilities/{facility}/budget-settings`

## Overview

Budget settings exist at **two levels**:

1. **Company level** — the seeded `settings` rows in group `budget`, managed through
   the shared general settings endpoints. See
   [Budget Settings](../settings.md#budget-settings). These are the defaults.
2. **Property level** — the same eight keys, optionally overridden per property and
   stored as columns on `facilities`. That is what this endpoint manages.

When a budget is generated (or monitored) the engine reads each setting from the
property first; if the property has no value for that key it falls back to the
company setting. A property therefore starts fully **inherited**, and the UI only
needs to persist the fields a user actually changes.

> Note the column names match the setting keys with one exception:
> `facility_budget_cycle` is stored on the property as **`budget_cycle`**. Always
> send the **column** name in the payload — each entry in the response carries both
> (`key` and `column`) so you never have to hardcode the mapping.

## Endpoints

- `GET /facilities/{facility}/budget-settings` — read the property's budget settings
- `PUT /facilities/{facility}/budget-settings` — save overrides (partial payload allowed)

## Authorization

| Action | Permission |
|---|---|
| Show | `view-facility` |
| Update | `update-facility` |

## Show

`GET /facilities/{facility}/budget-settings`

Returns one entry per overridable key, in a fixed display order, each carrying both
the property's own value and the value the budget engine will actually use.

```json
{
  "facility_id": 4,
  "settings": [
    {
      "key": "facility_budget_cycle",
      "column": "budget_cycle",
      "name": "Property Budget Cycle",
      "description": "Determines how often a property budget is prepared…",
      "type": "select",
      "options": {
        "monthly": "Monthly",
        "quarterly": "Quarterly",
        "semi_annually": "Semi-Annually",
        "annually": "Annually"
      },
      "dependency_key": null,
      "dependency_value": null,
      "value": null,
      "default_value": "annually",
      "effective_value": "annually",
      "inherited": true
    },
    {
      "key": "expenditure_budget_derivative_operation",
      "column": "expenditure_budget_derivative_operation",
      "name": "Expenditure Adjustment Operation",
      "type": "select",
      "options": { "add": "Add", "less": "Less" },
      "dependency_key": "expenditure_budget_derivative",
      "dependency_value": "add_or_less",
      "value": "less",
      "default_value": "add",
      "effective_value": "less",
      "inherited": false
    }
  ]
}
```

Field meanings:

| Field | Meaning |
|---|---|
| `key` | The company setting key this entry overrides |
| `column` | The payload field name to send on `PUT` |
| `name`, `description`, `type`, `options`, `dependency_key`, `dependency_value` | Presentation metadata, read from the company's own `settings` row so the two never drift |
| `value` | The property's override. `null` when inherited |
| `default_value` | The company setting's current value |
| `effective_value` | What the budget engine will use — `value` when set, otherwise `default_value` |
| `inherited` | `true` when `value` is `null` |

### Rendering

Reuse the company Budget tab layout — the `type`, `options` and dependency rules are
identical (see the [UI mapping](../settings.md#budget-settings)). The only additions:

- Render `default_value` as the input's **placeholder** while `inherited` is `true`,
  so the user can see what the property is currently falling back to.
- Offer a per-field "reset to company default" control that sends `null` for that
  column.
- Apply the same dependency behaviour: only show `expenditure_budget_derivative_operation`
  and `..._operation_amount` when the *effective* `expenditure_budget_derivative` is
  `add_or_less`, and `..._average_periods` when it is `average`.

## Update

`PUT /facilities/{facility}/budget-settings`

A **partial** payload. Fields absent from the body are left untouched; sending an
explicit `null` clears the override so the property inherits the company setting
again. There is no separate reset endpoint.

| Field | Type | Rules |
|---|---|---|
| `budget_cycle` | string / null | `monthly`, `quarterly`, `semi_annually`, `annually` |
| `residential_units_income_estimate` | numeric / null | `0`–`100` |
| `commercial_units_income_estimate` | numeric / null | `0`–`100` |
| `expenditure_budget_derivative` | string / null | `same_as_last_cycle`, `add_or_less`, `average` |
| `expenditure_budget_derivative_operation` | string / null | `add`, `less` |
| `expenditure_budget_derivative_operation_amount` | numeric / null | `0`–`100` |
| `expenditure_budget_derivative_average_periods` | integer / null | `>= 1` |
| `budget_at_risk_threshold_percent` | numeric / null | `0`–`100` |

Example — override the cycle and the at-risk band, leave everything else inherited:

```json
{
  "budget_cycle": "quarterly",
  "budget_at_risk_threshold_percent": 25
}
```

Example — reset the cycle back to the company default without touching anything else:

```json
{ "budget_cycle": null }
```

Response:

```json
{
  "message": "Property budget settings updated successfully.",
  "budget_settings": { "facility_id": 4, "settings": [ … ] }
}
```

Invalid values return `422` with per-field validation errors keyed by the column name.

> Changing these settings does **not** retroactively rewrite an existing budget.
> They take effect the next time a budget is generated for the property, and
> `budget_at_risk_threshold_percent` takes effect on the next monitoring run.

## Errors

Use shared behavior in `docs/frontend/app/README.md`:

- Render `4xx` messages and validation details.
- Show generic fallback for `5xx`.
