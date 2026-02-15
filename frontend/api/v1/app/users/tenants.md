# Tenants API

Domain: `Users`

Base prefix:

`/api/v1/app/{company}/users/tenants`

## Endpoints

Currently implemented:

- `GET /tenants`
- `POST /tenants`
- `GET /tenants/{tenant}`
- `DELETE /tenants/{tenant}`

Route exists but is not implemented in controller:

- `PUT/PATCH /tenants/{tenant}`

## List Tenants

`GET /api/v1/app/{company}/users/tenants`

Supported query params:

- Filters:
  - `filter[id]`
  - `filter[name]`
  - `filter[email]`
  - `filter[phone]`
  - `filter[created_at]`
- Sort:
  - `sort=id,name,created_at`
- Include:
  - Not supported
- Select fields:
  - Not supported
- Pagination:
  - `per_page`, `page`

Example:

- `GET /api/v1/app/12/users/tenants?filter[email]=@gmail.com&sort=name`

## Create/Elevate Tenant

`POST /api/v1/app/{company}/users/tenants`

Behavior:

- If `email` matches an existing user, backend elevates that user to tenant.
- If no user exists, backend creates a new user then elevates.

Request fields:

| Field | Required | Type | Allowed Values / Notes |
|---|---|---|---|
| `email` | Yes | string (email) | Used to check existing user |
| `name` | Conditional | string | Required when creating a brand-new user |
| `phone` | Conditional | string | Required when creating a brand-new user |
| `portal` | Conditional | integer | Required for brand-new user; must exist in `user_groups.id` |
| `terms` | Conditional | boolean/accepted | Required for brand-new user |
| `policy` | Conditional | boolean/accepted | Required for brand-new user |
| `has_vat` | No | boolean | - |
| `tax_pin` | Conditional | string | Required if `has_vat=true`; unique |
| `withholds` | No | array | Optional |

Example request:

```json
{
  "name": "Brian Otieno",
  "email": "brian.otieno@example.com",
  "phone": "+254700111222",
  "portal": 3,
  "terms": true,
  "policy": true
}
```

Example response:

```json
{
  "message": "Tenant created successfully.",
  "user": {
    "id": 440,
    "name": "Brian Otieno",
    "email": "brian.otieno@example.com",
    "phone": "+254700111222"
  }
}
```

## Important Behavior Note

Current `DELETE /tenants/{tenant}` implementation returns a response but tenant deletion logic is not yet wired (backend code is commented). Treat this endpoint as pending full delete behavior.

## Errors

Use shared behavior in `docs/frontend/app/README.md`:

- Show backend `4xx` messages and validation errors to users.
- Show generic fallback for `5xx` responses.
