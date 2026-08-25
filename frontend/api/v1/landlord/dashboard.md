# Landlord Dashboard API

Base route:

`/api/v1/landlord`

## Endpoint

- `GET /api/v1/landlord`

## Query Parameters

| Param | Type | Required | Description |12
|---|---|---|---|
| `period[from]` | date (`Y-m-d`) | No | Start date for all dashboard widgets. |
| `period[to]` | date (`Y-m-d`) | No | End date for all dashboard widgets. |
| `facility_ids[]` | integer[] | No | Facility filter. Only facilities owned by current landlord are considered. |

Default behavior:

- If `period` is not provided, backend uses the current quarter (`startOfQuarter` to `endOfQuarter`).
- If `facility_ids` is not provided, backend uses all facilities owned by current landlord.

## Response

Returns dashboard cards, chart series, top performers, and current dashboard permissions.

Example response:

```json
{
  "stats": {
    "total_income": 125000.5,
    "total_expenses": 35000,
    "operating_income": 90000.5
  },
  "revenue": {
    "period": {
      "from": "2026-04-01",
      "to": "2026-06-30"
    },
    "granularity": "monthly",
    "data": [
      { "label": "Apr 2026", "value": 40000 },
      { "label": "May 2026", "value": 50000 },
      { "label": "Jun 2026", "value": 35000.5 }
    ]
  },
  "expenses": {
    "period": {
      "from": "2026-04-01",
      "to": "2026-06-30"
    },
    "granularity": "monthly",
    "data": [
      { "label": "Apr 2026", "value": 12000 },
      { "label": "May 2026", "value": 10000 },
      { "label": "Jun 2026", "value": 13000 }
    ]
  },
  "top_five_earners": [
    { "id": 10, "label": "Westland Towers", "value": 65000 },
    { "id": 12, "label": "Riverfront Suites", "value": 45000 }
  ],
  "top_five_spenders": [
    { "id": 3, "label": "Repairs and Maintenance", "value": 17000 },
    { "id": 6, "label": "Security", "value": 9000 }
  ],
  "permissions": [
    "view-facility",
    "create-remittance"
  ]
}
```

## Granularity Rules

- If selected period is less than one month: `daily`
- If selected period is one month up to less than one quarter: `weekly`
- If selected period is one quarter or more: `monthly`
