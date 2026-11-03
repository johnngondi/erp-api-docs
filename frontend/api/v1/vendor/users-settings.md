# Vendor Users and Settings API

Base route:

`/api/v1/vendor`

## Dashboard

- `GET /`

## Users

- `GET|POST /users`
- `GET|PUT|PATCH|DELETE /users/{user}`
- `GET /roles` - the roles a supplier can give its staff logins

A supplier's staff login (sub-user) signs in to the supplier portal and **acts for the supplier**:
it sees and works on the supplier's LPOs, quotes, bills and so on, and the supplier's pending tasks,
as far as its **roles** allow (row 177). With no role it can sign in but do nothing. Roles are the
vendor user group's, defined by staff in Access Management and shared by every supplier
(Vendor Admin, Vendor Technician, ...). Each login sets its own PIN (`/api/v1/auth/pin`).

### List Supplier Roles

`GET /roles`

For the role picker. Allowed with `create-vendor-user` or `update-vendor-user` (`403` otherwise).

```json
{ "data": [
  { "id": 3, "name": "Vendor Admin", "description": null,
    "permissions": [ { "name": "view-facility-procurement-lpo", "action": "View", "resource": "Facility Procurement Lpo" }, ... ] }
] }
```

Show each role's `permissions` when it is picked, so the supplier sees what the login will be able
to do.

**Menus.** The supplier dashboard (`GET /api/v1/vendor/`) returns `permissions`: the logged-in
person's own view/create names from their roles. Hide a menu whose `view-` permission is missing (an
LPO-only login has `view-facility-procurement-lpo` and no `view-facility-bill`); the API refuses with
`403` anyway.

**Suspended or deactivated logins.** A staff login whose `status` is not `active` is refused on every
supplier-portal route with `403`:

```json
{ "message": "Your access to this supplier account is suspended. Ask your company's admin.", "code": "supplier_login_suspended", "status": "suspended" }
```

`code` is `supplier_login_suspended` or `supplier_login_inactive`; branch on it, not the message. It
works again as soon as it is set back to `active`. The supplier's own account is never a staff login,
so this never applies to it.

### List Vendor Users

`GET /users`

Query params:

- `filter[search]`: the login's name, email or phone. Every word must match one of them, so
  `jane gmail` finds Jane at a gmail address. A phone is found however it is typed: `0712345678`,
  `254712345678`, `+254712345678` or `712345678`
- `filter[role_id]`: logins holding that supplier role. Several ids, comma-separated, match any of
  them (`filter[role_id]=4,9`). The ids are the ones `GET /roles` lists
- `filter[status]`: exact. `active`, `suspended` or `inactive`; several comma-separated match any
- `filter[is_technician]`: exact, `1` or `0`
- `filter[id]`
- `filter[created_at]`
- `filter[created_between]`
- `filter[user.name]`, `filter[user.email]`: partial, one column each; `filter[search]` covers both
- `include=attachment` (optional)
- `sort=id,created_at,user.name,user.email`
- `per_page` (optional)
- `page` (optional)

Response item shape (`VendorUserResource`):

- `id` (the `vendor_users` row, which `{user}` in the routes means)
- `profile` (`UserResource`: `id`, `name`, `email`, `phone`, `tax_pin`, `profile_photo_url`)
- `is_me`: `true` on the caller's own login; hide the role picker there
- `is_technician`
- `qualifications`
- `attachment` (`UploadResource`, when included/loaded)
- `roles` (`[{ id, name }]`): the supplier roles the login holds
- `status` (`{ value, color }`)
- `created` (`raw`, `formatted`, `diff`)
- `permissions` (`{ view, update, delete }`): which row actions to show

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
- If user exists, user is elevated to vendor group (existing `is_statutory_vendor=true` is preserved).
- If user does not exist, backend creates a new user and sets `is_statutory_vendor=false`.
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
| `role_ids` | integer[] | No | Supplier roles from `GET /roles`; Vendor Admin is allowed. Any other role answers `422` `errors.role_ids: ["Choose from the supplier roles only."]`. None leaves the login unable to act until given one |

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
  "status": "active",
  "role_ids": [4]
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
- `role_ids`: the login's supplier roles, **all of them** (send `[]` to take them all away). Left out
  keeps them. Only the login's supplier roles change; roles it has in another portal stay. Nobody
  changes their own roles: `422` `errors.role_ids: ["You cannot change your own roles. Ask another admin of your company."]`

Success response (`DataResource`):

- `message`
- `user` (`VendorUserResource`)

### Delete Vendor User

`DELETE /users/{user}`

Behavior:

- Deletes vendor user mapping.
- Removes `vendor` user group, and the supplier roles, from linked `users` record only when that user is not linked in any other `vendor_users` row.

Success response (`DataResource`):

- `message`

## Settings

- `GET /settings`
- `POST /settings`
