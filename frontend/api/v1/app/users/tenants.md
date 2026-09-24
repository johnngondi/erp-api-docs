# Tenants API

Domain: `Users`

Base prefix:

`/api/v1/app/{company}/users/tenants`

A tenant is not a separate table. It is a `users` row that holds the `tenant` user
group. Creating a tenant either creates that user or elevates an existing one;
deleting a tenant takes the group away again rather than erasing the person.

## Endpoints

| Method | URI | Purpose |
|---|---|---|
| `GET` | `/tenants` | List tenants |
| `POST` | `/tenants` | Create a tenant, or elevate an existing user to one |
| `GET` | `/tenants/{tenant}` | Show one tenant |
| `PUT\|PATCH` | `/tenants/{tenant}` | Update a tenant's details |
| `DELETE` | `/tenants/{tenant}` | Deactivate a tenant (see below — this is not a hard delete) |

`{tenant}` is a `users.id`. If that user does not hold the `tenant` group, every
single-record route answers `404` with `{"message": "Tenant not found"}`, so a
landlord or vendor id cannot be read or edited through this surface.

## Permissions

Tenants are managed from the lease permission set — there is no separate
`*-tenant` permission.

| Endpoint | Permission required |
|---|---|
| `GET /tenants`, `GET /tenants/{tenant}` | `view-lease` |
| `POST /tenants` | `create-lease` |
| `PUT\|PATCH /tenants/{tenant}` | `update-lease` |
| `DELETE /tenants/{tenant}` | `delete-lease` |

A caller without the permission gets `403`. Hide the action rather than letting
the request fail where you already know the user's permissions.

## Tenant object

Every endpoint that returns a tenant returns this shape.

```json
{
  "id": 440,
  "name": "Brian Otieno",
  "email": "brian.otieno@example.com",
  "phone": "+254700111222",
  "has_vat": true,
  "tax_pin": "A001234567X",
  "address": "P.O. Box 1234-00100, Nairobi",
  "withholds": [1, 3],
  "profile_photo_url": "https://.../default.png",
  "created_at": {
    "raw": "2026-09-24T09:15:00.000000Z",
    "formatted": "24 Sep, 2026",
    "diff": "2 hours ago"
  },
  "status": {
    "value": "active",
    "color": "success",
    "label": "Active"
  },
  "permissions": {
    "view": true,
    "update": true,
    "delete": false
  }
}
```

Notes:

- `withholds` is an array of `withholding_taxes.id` values, not objects.
- `status` is the user account's own status, not a tenancy status. A tenant
  removed from the tenant group disappears from this surface entirely rather
  than turning up here as `inactive` — see *Deactivate*.
- `permissions` is per-request and reflects the caller, so it is safe to drive
  row-level action buttons from it.
- `has_vat`, `tax_pin`, `address`, `withholds` and `created_at` are emitted only
  when the attribute was selected on the query. All current tenant endpoints
  select the full row, so treat them as always present here.

## List Tenants

`GET /api/v1/app/{company}/users/tenants`

Query params:

- Filters:
  - `filter[search]` — Scout search across name, email, phone and tax pin. Also
    accepts a CSV of user ids (`filter[search]=12,45`), which is how you fetch a
    known set in one call.
  - `filter[id]`
  - `filter[name]`
  - `filter[email]`
  - `filter[phone]`
  - `filter[created_at]`
- Sort: `sort=id,name,created_at` (prefix with `-` to reverse)
- Include: not supported
- Select fields: not supported
- Pagination: `per_page`, `page`

There is no `filter[status]` on this endpoint. Deactivated tenants are not in the
result set at all, so a status filter would have nothing to select.

Example:

```http
GET /api/v1/app/12/users/tenants?filter[email]=@gmail.com&sort=name&per_page=25
```

Response is a standard paginated collection: `data`, `links`, `meta`.

## Create / Elevate Tenant

`POST /api/v1/app/{company}/users/tenants`

The backend looks the `email` up first:

- **No user with that email** — creates the user, then grants the `tenant` group
  and its roles. Message: `"Tenant created successfully."`
- **User already exists** — leaves their details untouched and only grants the
  `tenant` group and roles. Message:
  `"User with details already exists. They have been elevated to a tenant."`

Branch your success toast on the returned `message`; there is no separate flag.
This means a `POST` for an existing landlord or staff member succeeds and makes
them a tenant as well — it is not an error.

Request fields:

| Field | Required | Type | Notes |
|---|---|---|---|
| `email` | Yes | string (email) | Max 255. Unique across `users` when creating a new user |
| `name` | Conditional | string | Required when the email is new. Max 255 |
| `phone` | Conditional | string | Required when the email is new. Unique across `users` |
| `terms` | Conditional | accepted | Required when the email is new |
| `policy` | Conditional | accepted | Required when the email is new |
| `has_vat` | No | boolean | Defaults to `false` |
| `tax_pin` | Conditional | string | Required when `has_vat` is `true`. Unique across `users` |
| `withholds` | No | array of int | `withholding_taxes.id` values |

"Conditional" fields are only validated when a brand-new user is being created.
Because the frontend cannot know in advance whether the email exists, always send
the full set — the extra fields are ignored on the elevate path.

**Do not send `portal`.** The portal a new tenant lands in comes from the
company's `tenant_user_group_id` setting, and the backend injects it after
reading the payload. A `portal` in the request body is overwritten, so it cannot
be used to place the new user in a different group.

The generated password is created server-side and is not returned.

Request:

```json
{
  "name": "Brian Otieno",
  "email": "brian.otieno@example.com",
  "phone": "+254700111222",
  "terms": true,
  "policy": true,
  "has_vat": true,
  "tax_pin": "A001234567X"
}
```

Response `200`:

```json
{
  "message": "Tenant created successfully.",
  "user": { "id": 440, "name": "Brian Otieno", "...": "..." }
}
```

## Show Tenant

`GET /api/v1/app/{company}/users/tenants/{tenant}`

Returns the tenant object directly under `data`. `404` if the id is not a tenant.

## Update Tenant

`PUT|PATCH /api/v1/app/{company}/users/tenants/{tenant}`

Edits the person's own details. It does not touch user groups, roles or leases —
use `POST /tenants` to grant the tenant group and `DELETE` to take it away.

Every field is optional; send only what changed. A field that is absent is left
as it is, so a `PATCH` with a single key is a valid request.

| Field | Required | Type | Notes |
|---|---|---|---|
| `name` | No | string | Max 255 |
| `email` | No | string (email) | Max 255. Unique across `users`, ignoring this tenant |
| `phone` | No | string | Unique across `users`, ignoring this tenant |
| `has_vat` | No | boolean | |
| `tax_pin` | No | string | Required when `has_vat` is sent as `true`. Unique across `users`, ignoring this tenant |
| `address` | No | string | Max 255. Send `null` to clear |
| `withholds` | No | array of int | `withholding_taxes.id` values. Send `[]` to clear |

Changing `email` changes the address the tenant signs in with. Confirm that in
the UI before submitting.

Request:

```json
{
  "phone": "+254700999888",
  "has_vat": true,
  "tax_pin": "A001234567X",
  "address": "P.O. Box 1234-00100, Nairobi"
}
```

Response `200`:

```json
{
  "message": "Tenant updated successfully.",
  "user": { "id": 440, "phone": "+254700999888", "...": "..." }
}
```

Validation failures return the standard `422` shape keyed by field name.

## Deactivate Tenant

`DELETE /api/v1/app/{company}/users/tenants/{tenant}`

The endpoint removes the `tenant` user group from the person rather than deleting
a tenant record. For anyone who holds another group as well, their `users` row,
leases, invoices and receipts stay exactly as they were.

Read the last bullet below before wiring this up: for a tenant who holds no other
group, the account itself is removed.

What the caller sees:

- The user stops appearing in `GET /tenants` and `GET /tenants/{tenant}` returns
  `404`, because both are scoped to holders of the `tenant` group.
- The user keeps every other group they hold. Someone who is both a tenant and a
  landlord stays a landlord and remains visible under `/users/landlords`.
- Only when the `tenant` group was their **last** group is the user account
  itself deleted, on the grounds that they can no longer sign in as anything.
  **That deletion is permanent** — the `User` model carries no soft-delete
  behaviour, so the row is gone. In practice this is the common case: a tenant
  who is only ever a tenant holds no other group, so deactivating them removes
  the account. Warn accordingly in the confirmation dialog.

Re-activating is just `POST /tenants` with the same email, which takes the
elevate path and grants the group back. There is no separate restore endpoint.

Response `200`:

```json
{
  "message": "Tenant deactivated successfully.",
  "user": { "id": 440, "name": "Brian Otieno", "...": "..." }
}
```

The returned `user` is the tenant as it was immediately before deactivation, so
you can render a confirmation without a second fetch.

UI guidance: label the action **Deactivate**, not *Delete*, and word the
confirmation around losing tenant access rather than losing the record.

## Errors

Shared behaviour is in `docs/frontend/api/v1/app/README.md`.

| Status | When | Handling |
|---|---|---|
| `403` | Caller lacks the lease permission for the action | Hide or disable the action |
| `404` | Id is not a tenant, or was already deactivated | Refresh the list |
| `422` | Validation — duplicate email/phone/tax pin, missing conditional field | Render field errors, keep the form |
| `500` | `"Error creating tenant"` / `"Error updating tenant"` / `"Error deactivating tenant"` | Generic retry message. The write is rolled back, so nothing partial was saved |

All four writes run in a transaction, so a `500` never leaves a half-created
tenant behind.
