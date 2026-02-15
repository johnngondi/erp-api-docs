# Property Management Dashboard API

Base route:

`/api/v1/app/{company}/property-management`

## Endpoint

- `GET /api/v1/app/{company}/property-management`

## Request

- No request body.
- No list filters/sorts/includes/fields.

## Response

Returns summary cards, charts, recent ticket items, and current user permissions.

Example response:

```json
{
  "month_total_expenses": "KES 2,085,365.02",
  "pending_jobs": "25",
  "month_power_usage": "2,500 kWh",
  "graphs": {
    "expenses": {
      "data": [
        { "label": "Jan 2024", "value": "250" },
        { "label": "Feb 2024", "value": "203" }
      ]
    },
    "power_usage": {
      "data": [
        { "label": "Jan 2024", "value": "3200" },
        { "label": "Feb 2024", "value": "2030" }
      ]
    }
  },
  "recent_tickets": [
    {
      "request": { "primary": "Leaking Sink repair request", "secondary": "2 days ago" },
      "facility": { "primary": "Jumuia Place 2", "secondary": "1st Floor - Unit 203" },
      "assigned_user": { "primary": "Jon Doe", "secondary": "Plumber" },
      "allowed_actions": ["elevate_to_request", "close_ticket"]
    }
  ],
  "permissions": [
    "view-facility",
    "create-ticket"
  ]
}
```

## Frontend Notes

- Treat this payload as read-only dashboard data.
- If any widget field is missing, hide that widget gracefully instead of failing the whole page.

## Errors

Use shared error handling behavior from `docs/frontend/app/README.md`:

- Show `4xx` messages to users.
- Show generic message for `5xx`.
