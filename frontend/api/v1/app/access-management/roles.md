# Access Management: Roles

Base prefix:

`/api/v1/app/{company}/access-management`

## Endpoints

Implemented:

- `GET /roles`
- `POST /roles`
- `GET /roles/{role}`
- `PUT/PATCH /roles/{role}`
- `DELETE /roles/{role}`

## Role Scope Rules

- `app` user-group roles are company scoped.
- For `app` roles, the backend only exposes roles where `roles.company_id == {company}`.
- Non-`app` roles are treated as global (non-company-scoped).

## List Roles

`GET /api/v1/app/{company}/access-management/roles`

Supported query params:

- Filters:
  - `filter[search]` (Scout search; supports CSV role ids e.g. `12,15,20`)
  - `filter[user_group_id]`
  - `filter[company_id]`
  - `filter[created_at]` (date: `YYYY-MM-DD`)
- Include:
  - `include=permissions`
- Pagination:
  - `per_page`, `page`

## Create Roles (Single Request, Multi-Company)

`POST /api/v1/app/{company}/access-management/roles`

Request fields:

| Field                 | Required    | Type       | Notes                                               |
| --------------------- | ----------- | ---------- | --------------------------------------------------- |
| `name`                | Yes         | string     | Role name                                           |
| `permissions`         | Yes         | array      | Permission names                                    |
| `company_ids`         | Yes         | array[int] | Company ids where role is created                   |
| `description`         | No          | string     | Optional                                            |
| `guard_name`          | No          | string     | Defaults to `web`                                   |
| `enforce_on_facility` | No          | boolean    | Optional                                            |

Server-side inference:

- `user_group_id` is inferred from the current app user-group context.
- Sending `user_group_id` in payload returns `422`.

Example request:

```json
{
  "name": "company-manager",
  "description": "Manager role",
  "permissions": ["view-lease", "create-lease"],
  "company_ids": [12, 13]
}
```

Example response:

```json
{
  "data": {
    "message": "Role created successfully.",
    "role": {
      "id": 101,
      "name": "Company Manager",
      "permissions": [12, 13]
    }
  }
}
```

## Show Role

`GET /api/v1/app/{company}/access-management/roles/{role}`

- Returns role details (including `permissions` include if loaded by backend).
- Returns `404` when trying to access an `app` role from a different company context.

## Update Role

`PUT/PATCH /api/v1/app/{company}/access-management/roles/{role}`

Updatable fields:

- `name`
- `description`
- `permissions`

Prohibited fields:

- `user_group_id`
- `company_id`
- `company_ids`
- `guard_name`
- `enforce_on_facility`

## Delete Role

`DELETE /api/v1/app/{company}/access-management/roles/{role}`

- Deletes the role in current company scope.
- For `app` roles in another company context, returns `404`.
