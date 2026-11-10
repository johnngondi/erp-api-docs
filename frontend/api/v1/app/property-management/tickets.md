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
- `PATCH /tickets/{ticket}/assign`: hand the ticket to a member of staff (row 179)
- `PATCH /tickets/{ticket}/close`: close it with a resolution (row 179)

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

Use this to populate create/update forms. It returns **active categories only** - one
deactivated in settings drops out of here on its own, so nothing has to change on this screen.

Managing the categories themselves (create, edit, activate, deactivate) lives in Settings:
[Ticket Categories Settings API](settings/ticket-categories.md).

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
    "priority": { "value": "high", "color": "danger", "label": "High" },
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

Who may elevate, and when. Drive the button off `permissions.elevate`; it is `true` only when all
of these hold, and the endpoint refuses with `403` otherwise:

- the caller has `elevate-facility-ticket` and is allocated to the ticket's property;
- the ticket's category allows elevation (`is_procurable` on the
  [ticket category](settings/ticket-categories.md)). Otherwise `403`:
  `Tickets in this category cannot be elevated to a procurement request.`;
- the ticket is open. Otherwise `403`: `This ticket is already closed.` Anything new after a
  ticket is closed is raised as a new ticket;
- the ticket has not been elevated already. Otherwise `403`:
  `This ticket has already been elevated to procurement request #320.` A ticket raises one
  procurement request; `procurement_request` on the ticket links it.

A category allowed to elevate still needs its expense type and sub type set; without them the
endpoint answers `422` on `ticket_category_id` naming what is missing.

## After a ticket is raised (row 179)

Every ticket response now carries `assigned_to_id` and `assigned_to` (`{ id, name }`), null until
someone picks it up, and `permissions.assign` / `permissions.close`, which say whether the caller
may do those - drive the two buttons off them.

**Who hears about a new ticket.** A ticket category can name a person (see
[ticket categories](settings/ticket-categories.md)). Raising a ticket under it - from the staff
form, the tenant portal, or a dispute on an invoice, credit note or receipt - sends that person an
in-app notification and an email, and gives them the ticket's **pending task**. A category with
nobody named tells nobody, as before.

**Tickets raised from the tenant portal.** A ticket belongs to its property's company, whoever
raises it. Tickets and disputes raised from the tenant portal used to be saved with no company, so
they never appeared on these lists and opening one answered `404`. New ones now appear like any
other.

**The ticket's history.** `GET /tickets/{ticket}` returns `entries`, oldest first, each with
`title`, `description`, `created_at` and `owner` (`{ id, name }`, who wrote it). Closing with a
resolution adds the last one, titled `Resolved`.

### Assign

`PATCH /tickets/{ticket}/assign` with `{ "user_id": 42 }`

Fill the picker from `GET /facilities/{facility}/admins` with the ticket's `facility_id`: the
active staff allocated to that property, which is who assign accepts. Send a row's `id`.

- `user_id` must be active staff of the company, allocated to the ticket's property. Otherwise
  `422` on `user_id` (`Choose an active member of staff.` / `<name> is not assigned to this
  property.`).
- Allowed for anyone with `update-facility-ticket` on the property, and for whoever the ticket is
  currently with (so it can be passed on without the edit permission). `403` otherwise.
- A closed ticket cannot be assigned: `403` with `message: "This ticket is already closed."`, and
  `permissions.assign` is false.
- The assignee gets an in-app notification and an email, and the pending task moves to them; the
  person who had it loses the task.

Response: `{ "data": { "message": "Ticket assigned to Jane.", "ticket": { ... } } }`.

### Close

`PATCH /tickets/{ticket}/close` with `{ "resolution": "Rent corrected and a credit note raised." }`
(`resolution` optional, up to 2,000 characters).

- Same people as assign. The person who raised the ticket does not close it. Already closed: `403`
  with `message: "This ticket is already closed."`, and `permissions.close` is false.
- A closed ticket is no longer edited: `PUT /tickets/{ticket}`, and the tenant portal's own edit,
  answer `403` with the same message, and `permissions.update` is false.
- The resolution is kept as the ticket's last entry (`entries[]`, `title: "Resolved"`), `status`
  becomes `closed` with `closed`, the pending task is cleared, and the person who raised the ticket
  (`owner`) gets an in-app notification and an email with the resolution.
- A ticket raised by a **dispute** settles it: the invoice, credit note or receipt stops showing as
  disputed (`disputed_at`, `dispute_details`, `disputed_by` cleared) and keeps `dispute_ticket_id`
  pointing at the closed ticket as the record. The document can then be disputed again: that raises
  a new ticket, `dispute_ticket_id` moves to it, and the closed one stays in this list.

Response: `{ "data": { "message": "Ticket closed.", "ticket": { ... entries ... } } }`.

## Common enum values in responses

- `status`: `open` (colour `secondary`), `closed` (colour `danger`)
- `priority`: `high`, `normal`, `low`

## Errors

Use shared error handling from `docs/frontend/app/README.md`:

- Show `4xx` validation/permission errors to users.
- Show generic message for `5xx`.
