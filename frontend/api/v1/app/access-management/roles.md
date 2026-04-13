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
- Default listing (without `filter[company_id]`) exposes `app` roles where `roles.company_id == {company}`.
- With `filter[company_id]`, you can query `app` roles across accessible companies (single, CSV, or array values).
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

`filter[company_id]` notes:

- Omitted: defaults to route `{company}`.
- Present: must contain only companies accessible to the authenticated user.
- Inaccessible company ids return `422`.

## Create Roles (Single Request, Multi-Company)

`POST /api/v1/app/{company}/access-management/roles`

Create payload table:

| Field | Type | Required | Default | Notes |
| --- | --- | --- | --- | --- |
| `name` | string | Yes | - | Unique per selected company scope and app user group |
| `user_group_id` | int | No | `app` group id | Target role user group |
| `permissions` | array[int] | Yes | `[]` | Permission IDs for selected `user_group_id` |
| `company_ids` | array[int] | Conditional | `[]` | Required for `app` user group; not allowed for non-app groups |
| `description` | string&#124;null | No | `null` | Optional |
| `guard_name` | string | No | `web` | Guard used for role |
| `enforce_on_facility` | boolean | No | `false` | Optional flag |

Server-side inference:

- If `user_group_id` is omitted, it defaults to `app` group.
- For `app` user-group roles, `company_ids` must be provided and must be accessible.
- For non-app user-group roles, role is created as global (`company_id = null`).

Example request:

```json
{
  "name": "company-manager",
  "user_group_id": 3,
  "description": "Manager role",
  "permissions": [17, 21],
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
- `enforce_on_facility`

Prohibited fields:

- `user_group_id`
- `company_id`
- `company_ids`
- `guard_name`

## Delete Role

`DELETE /api/v1/app/{company}/access-management/roles/{role}`

- Deletes the role in current company scope.
- For `app` roles in another company context, returns `404`.
