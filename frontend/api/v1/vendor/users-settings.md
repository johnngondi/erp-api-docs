# Vendor Users and Settings API

Base route:

`/api/v1/vendor`

## Dashboard

- `GET /`

## Users

- `GET|POST /users`
- `GET|PUT|PATCH|DELETE /users/{user}`

### List Vendor Users

`GET /users`

Query params:

- `filter[id]`
- `filter[created_at]`
- `filter[user.name]`
- `filter[user.email]`
- `filter[is_technician]`
- `include=attachment` (optional)
- `sort=id,created_at,user.name,user.email`
- `per_page` (optional)
- `page` (optional)

Response item shape (`VendorUserResource`):

- `id`
- `profile` (`UserResource`)
- `is_technician`
- `qualifications`
- `attachment` (`UploadResource`, when included/loaded)
- `status` (`{ value, color }`)
- `created` (`raw`, `formatted`, `diff`)

### Show Vendor User

`GET /users/{user}`

Rules:

- `{user}` is a `vendor_users.id`.
- User must belong to authenticated vendor.

Response shape (`VendorUserResource`):

- `id`
- `profile`
- `is_technician`
- `qualifications`
- `attachment`
- `status`
- `created`

### Create Vendor User

`POST /users`

Behavior:

- Finds user by `email`.
- If user exists, user is elevated to vendor group (existing `is_prequalified_vendor=true` is preserved).
- If user does not exist, backend creates a new user and sets `is_prequalified_vendor=false`.
- Creates `vendor_users` record under currently authenticated vendor.

Request body:

| Field | Type | Required | Notes |
|---|---|---|---|
| `email` | string | Yes | Existing user lookup key |
| `name` | string | Conditionally | Required when creating a new user |
| `phone` | string | Conditionally | Required when creating a new user |
| `is_technician` | boolean | No | Defaults to `false` |
| `qualifications` | string/null | No | Technician profile |
| `attachment_id` | integer/null | No | Must exist in `uploads.id` |
| `status` | string | No | `active`, `suspended`, `inactive` |

Success response (`DataResource`):

- `message`
- `user` (`VendorUserResource`)

Sample JSON (create vendor user):

```json
{
  "name": "Jane Technician",
  "email": "jane.technician@example.com",
  "phone": "712345678",
  "is_technician": true,
  "qualifications": "Certified HVAC and electrical technician",
  "attachment_id": 12,
  "status": "active"
}
```

### Update Vendor User

`PUT/PATCH /users/{user}`

Rules:

- Only `vendor_users` table fields are updatable.
- `name`, `email`, `phone`, and `user_id` are rejected.
- Vendor scoping is enforced: user must belong to authenticated vendor.

Updatable fields:

- `is_technician`
- `qualifications`
- `attachment_id`
- `status`

Success response (`DataResource`):

- `message`
- `user` (`VendorUserResource`)

### Delete Vendor User

`DELETE /users/{user}`

Behavior:

- Deletes vendor user mapping.
- Removes `vendor` user group from linked `users` record only when that user is not linked in any other `vendor_users` row.

Success response (`DataResource`):

- `message`

## Settings

- `GET /settings`
- `POST /settings`
