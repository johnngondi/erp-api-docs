# Ticket Categories Settings API

Domain: `Property Management > Settings > Tickets`

UI placement:

- `SettingsPage > Group Tab (Tickets) > Categories Tab`

Base route:

`/api/v1/app/{company}/property-management/settings/tickets/categories`

## Endpoints

- `GET /settings/tickets/categories`
- `POST /settings/tickets/categories`
- `GET /settings/tickets/categories/{ticketCategory}`
- `PUT/PATCH /settings/tickets/categories/{ticketCategory}`
- `DELETE /settings/tickets/categories/{ticketCategory}`
- `PATCH /settings/tickets/categories/{ticketCategory}/activate`
- `PATCH /settings/tickets/categories/{ticketCategory}/deactivate`

**Use this base, not `/api/v1/settings/ticket-categories`.** That older route has no `{company}`
segment, so a write made there could not be scoped to a company. It is now index-only and serves
the pickers (see [Which list to call](#which-list-to-call)).

## Deactivate, don't delete

A category is taken out of use by **deactivating** it, not by deleting it:

- A deactivated category stays on every ticket already filed under it, so history and reports
  are unchanged.
- It disappears from the pickers behind "raise a ticket" and the dispute buttons.
- It keeps showing in this settings list, where it can be reactivated.

`DELETE` is still there for a category created by mistake that nothing has used yet. The moment a
ticket references it, `DELETE` returns **422** and the message says to deactivate instead.

## List Ticket Categories

`GET /api/v1/app/{company}/property-management/settings/tickets/categories`

Lists **active and inactive** categories.

Supported query params:

- Filters (exact): `filter[id]`, `filter[status]`, `filter[priority]`, `filter[expense_type_id]`,
  `filter[expense_sub_type_id]`, `filter[is_procurable]`
- Filters (partial match): `filter[name]`, `filter[description]`, `filter[created_at]`
- Sort: `sort=id,name,priority,status,created_at` (default: newest first)
- Include: `include=expenseType,expenseSubType`
- Pagination: `per_page`, `page`

Examples:

- `GET .../categories?filter[status]=inactive`
- `GET .../categories?filter[name]=water&sort=name`

`filter[status]` is exact, so `active` does not also return `inactive`.

## Row shape

```json
{
  "id": 6,
  "name": "Lift Breakdowns",
  "description": "Lift and escalator faults.",
  "priority": { "value": "high", "color": "danger", "label": "High" },
  "expense_type_id": 3,
  "expense_sub_type_id": 4,
  "is_procurable": true,
  "status": { "value": "active", "color": "success", "label": "Active" },
  "expense_type": null,
  "expense_sub_type": null,
  "tickets_count": 0,
  "created_at": { "raw": "...", "formatted": "30 Sep, 2026", "diff": "1 second ago" },
  "permissions": { "view": true, "update": true, "delete": true, "activate": false, "deactivate": true }
}
```

Two fields decide which buttons to show, so the client never has to guess:

| Field | Use it for |
|---|---|
| `tickets_count` | `> 0` means delete is impossible. Offer **Deactivate** instead. |
| `permissions.delete` | `false` when the category is in use **or** the user lacks the permission. |
| `permissions.activate` / `permissions.deactivate` | Only one is ever `true` - show that button. |

## Create Ticket Category

`POST /api/v1/app/{company}/property-management/settings/tickets/categories`

| Field | Required | Type | Allowed Values / Notes |
|---|---|---|---|
| `name` | Yes | string (max 191) | Unique per company, ignoring deactivated-then-deleted rows |
| `description` | No | string | - |
| `priority` | No | string | `high`, `normal`, `low`. Defaults to `normal` |
| `expense_type_id` | No | integer | Must exist in `facility_expense_types.id` |
| `expense_sub_type_id` | No | integer | Must exist **and belong to `expense_type_id`** |
| `is_procurable` | No | boolean | Defaults to `false` |
| `status` | No | string | `active`, `inactive`. Defaults to `active` |

Example request:

```json
{
  "name": "Lift Breakdowns",
  "description": "Lift and escalator faults.",
  "priority": "high",
  "expense_type_id": 3,
  "expense_sub_type_id": 4,
  "is_procurable": true
}
```

Example response:

```json
{
  "data": {
    "message": "Ticket category created successfully.",
    "ticket_category": { "id": 6, "name": "Lift Breakdowns", "...": "..." }
  }
}
```

### The expense pair is checked together

A sub type belongs to exactly one expense type, so sending a pair that does not match is rejected
rather than quietly filing the ticket's expense to the wrong account:

```json
{ "message": "The selected expense sub type id is invalid.",
  "errors": { "expense_sub_type_id": ["The selected expense sub type id is invalid."] } }
```

Load the sub type dropdown from the chosen expense type and this cannot happen.

## Update Ticket Category

`PUT/PATCH /api/v1/app/{company}/property-management/settings/tickets/categories/{ticketCategory}`

Same payload shape as create. **`name` is always required**, everything else is optional.

- A field you leave out keeps its stored value.
- A field you send as `null` is cleared - that is how a category is unlinked from an expense
  account (`{"name": "...", "expense_type_id": null, "expense_sub_type_id": null}`).
- Sending the category's own `name` back is fine; uniqueness ignores the row being edited.

## Activate / Deactivate

`PATCH .../categories/{ticketCategory}/activate`
`PATCH .../categories/{ticketCategory}/deactivate`

No body. Both return the updated row under `data.ticket_category`.

```json
{ "data": { "message": "Ticket category deactivated successfully.",
            "ticket_category": { "id": 6, "status": { "value": "inactive", "color": "danger", "label": "Inactive" } } } }
```

Two refusals, both **422**, each with its own error key:

| When | Key | Message |
|---|---|---|
| Already in that state | `ticket_category_already_active` / `ticket_category_already_inactive` | `This ticket category is already inactive.` |
| The **only** active category left | `ticket_category_last_active` | `This is the only active ticket category. Activate another one before deactivating this one, otherwise tickets and disputes cannot be raised.` |

The last one exists because raising a ticket needs a category and a dispute picks one on the
tenant's behalf. With none active, no tenant could dispute an invoice at all.

## Delete Ticket Category

`DELETE /api/v1/app/{company}/property-management/settings/tickets/categories/{ticketCategory}`

Soft-deletes a category **no ticket has used**. Otherwise **422**, keyed
`ticket_category_in_use`:

```json
{ "message": "3 tickets are already filed under this category, so it cannot be deleted. Deactivate it instead to take them out of the pickers and keep those tickets intact.",
  "errors": { "ticket_category_in_use": ["3 tickets are already filed under this category, ..."] } }
```

Read `tickets_count` on the row and offer **Deactivate** up front rather than waiting for this.

## Every refusal has its own key

These endpoints never make you read the message to know what happened. Each refusal is a **422**
under a key of its own, so a client can branch on the key and treat the message purely as text to
display.

| Key | Raised by | What the user should be offered |
|---|---|---|
| `ticket_category_in_use` | `DELETE` | Deactivate instead - tickets are filed under it |
| `ticket_category_last_active` | `PATCH .../deactivate` | Activate another category first |
| `ticket_category_already_active` | `PATCH .../activate` | Nothing; the row was stale, refresh it |
| `ticket_category_already_inactive` | `PATCH .../deactivate` | Nothing; the row was stale, refresh it |

The first two are worth their own dialogs. The last two mean the list on screen is out of date -
refreshing it is the whole fix.

`ticket_category_in_use` should be rare if delete is disabled while `tickets_count` is above zero;
it is there for the race where someone files a ticket between the list loading and the click.

## Which list to call

| You are building | Call |
|---|---|
| The Settings > Tickets > Categories tab | `GET .../settings/tickets/categories` - active **and** inactive |
| A category picker (raise a ticket, dispute) | `GET .../tickets/categories` - **active only**, no change needed on your side |

The picker already drops deactivated categories, so nothing has to be added to those screens when
this tab ships.

## Permissions

| Ability | Permission |
|---|---|
| List, view | `view-facility-ticket-category` |
| Create | `create-facility-ticket-category` |
| Update, activate, deactivate | `update-facility-ticket-category` |
| Delete | `delete-facility-ticket-category` |

These are new names. On an existing install they are created by
`php artisan db:seed --class=NewPermissionsSeeder --force`, which **grants nothing** - a role has
to be given them in Access Management before the tab works.

## Frontend Error Handling

Apply shared rules in `docs/frontend/app/README.md`.
