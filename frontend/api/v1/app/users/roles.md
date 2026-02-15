# Roles API

Domain: `Users`

Base prefix:

`/api/v1/app/{company}/users/roles`

## Endpoints

Currently implemented:

- `GET /roles`
- `POST /roles`

Routes exist but are not implemented in controller:

- `GET /roles/{role}`
- `PUT/PATCH /roles/{role}`
- `DELETE /roles/{role}`

## List Roles

`GET /api/v1/app/{company}/users/roles`

Supported query params:

- Filters:
  - `filter[name]`
  - `filter[created_at]`
  - `filter[description]`
  - `filter[user_group_id]`
- Sort:
  - Not configured
- Include:
  - `include=permissions`
- Select fields:
  - Not supported
- Pagination:
  - `per_page`, `page`

Example:

- `GET /api/v1/app/12/users/roles?filter[user_group_id]=4&include=permissions`

## Create Role

`POST /api/v1/app/{company}/users/roles`

Request fields:

| Field | Required | Type | Allowed Values / Notes |
|---|---|---|---|
| `name` | Yes | string | Role name |
| `user_group_id` | Yes | integer | Must exist in `user_groups.id` |
| `description` | No | string | Optional |
| `guard_name` | No | string | Default `web` |
| `enforce_on_facility` | No | boolean | Default `false` |
| `permissions` | No | array | Permission names or IDs accepted by backend sync |

Important backend validation:

- Role `name` must be unique per `user_group_id`. Duplicate returns `422`.

Example request:

```json
{
  "name": "facility-finance-reviewer",
  "user_group_id": 2,
  "description": "Can review and approve finance records at facility level.",
  "enforce_on_facility": true,
  "permissions": [
    "view-bill",
    "approve-bill"
  ]
}
```

Example response:

```json
{
  "message": "Role created successfully.",
  "role": {
    "id": 88,
    "name": "Facility Finance Reviewer",
    "description": "Can review and approve finance records at facility level.",
    "enforce_on_facility": true,
    "userGroup": {
      "id": 2,
      "title": "Landlord",
      "description": "Landlord users"
    }
  }
}
```

## Errors

Use shared behavior in `docs/frontend/app/README.md`:

- Show backend `4xx` messages and validation errors to users.
- Show generic fallback for `5xx` responses.
