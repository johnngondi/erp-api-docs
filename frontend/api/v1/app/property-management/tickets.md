# Tickets API

Domain: `Property Management`

Base route:

`/api/v1/app/{company}/property-management/tickets`

## Endpoints

- `GET /tickets/categories`
- `GET /tickets`
- `POST /tickets`
- `GET /tickets/{ticket}`
- `PUT/PATCH /tickets/{ticket}`
- `DELETE /tickets/{ticket}`
- `POST /tickets/{ticket}/elevate`

## List Tickets

`GET /api/v1/app/{company}/property-management/tickets`

Supported query params:

- Filters:
  - `filter[id]`
  - `filter[title]`
  - `filter[facility_id]`
  - `filter[asset_id]`
  - `filter[ticket_category_id]`
  - `filter[status]`
  - `filter[priority]`
  - `filter[created_at]`
- Sort:
  - `sort=id,priority,created_at,status`
- Include: not supported
- Select fields: not supported
- Pagination:
  - `per_page`, `page`

Examples:

- `GET /api/v1/app/12/property-management/tickets?filter[status]=open&sort=-created_at`
- `GET /api/v1/app/12/property-management/tickets?filter[priority]=high&per_page=20`

## Ticket Categories

`GET /api/v1/app/{company}/property-management/tickets/categories`

Use this to populate create/update forms.

## Create/Update Ticket

`POST /api/v1/app/{company}/property-management/tickets`  
`PUT/PATCH /api/v1/app/{company}/property-management/tickets/{ticket}`

Request body:

| Field | Required | Type | Allowed Values / Notes |
|---|---|---|---|
| `title` | Yes | string | - |
| `description` | Yes | string | - |
| `ticket_type` | Yes | string | `asset`, `facility`, `other` |
| `ticket_category_id` | Yes | integer | Must exist in `ticket_categories.id` |
| `facility_id` | Yes | integer | Must exist in `facilities.id` |
| `facility_space_id` | Conditional | integer | Required if `ticket_type=facility`; must exist in `facility_spaces.id` |
| `asset_id` | Conditional | integer | Required if `ticket_type=asset`; must exist in `assets.id` |
| `owner_id` | No | integer | Must exist in `users.id`; defaults to current user |
| `user_group_id` | No | integer | Must exist in `user_groups.id`; auto-derived in some cases |

Example create request:

```json
{
  "title": "Water leak at Block A",
  "description": "Persistent leak in corridor near Unit 2A",
  "ticket_type": "facility",
  "ticket_category_id": 3,
  "facility_id": 14,
  "facility_space_id": 223
}
```

Example response:

```json
{
  "message": "Ticket created successfully.",
  "ticket": {
    "id": 901,
    "title": "Water leak at Block A",
    "priority": { "value": "high", "color": "danger" },
    "status": { "value": "open", "color": "primary" }
  }
}
```

## Elevate Ticket to Procurement Request

`POST /api/v1/app/{company}/property-management/tickets/{ticket}/elevate`

Request body/query:

| Field | Required | Type | Allowed Values / Notes |
|---|---|---|---|
| `type` | Yes | string | `purchase`, `work` |

Example request:

```json
{
  "type": "work"
}
```

Example response:

```json
{
  "message": "Ticket elevated to request successfully.",
  "ticket": {
    "id": 901,
    "is_elevated": true
  },
  "request": {
    "id": 320,
    "type": "work",
    "title": "Water leak at Block A"
  }
}
```

## Common enum values in responses

- `status`: `open`, `closed`
- `priority`: `high`, `normal`, `low`

## Errors

Use shared error handling from `docs/frontend/app/README.md`:

- Show `4xx` validation/permission errors to users.
- Show generic message for `5xx`.
