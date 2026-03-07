# Tenant Tickets API

Base route:

`/api/v1/tenant/tickets`

## Endpoints

- `GET /tickets`
- `POST /tickets`
- `GET /tickets/{ticket}`
- `PUT /tickets/{ticket}`
- `PATCH /tickets/{ticket}`
- `DELETE /tickets/{ticket}`

## Request Body (Create/Update)

| Field | Required | Type | Notes |
|---|---|---|---|
| `title` | Yes | string | Ticket title |
| `description` | Yes | string | Ticket details |
| `ticket_type` | Yes | string | `asset`, `facility`, `other` |
| `ticket_category_id` | Yes | integer | Existing ticket category id |
| `facility_id` | Yes | integer | Must belong to a facility where tenant has a lease |
| `facility_space_id` | Conditional | integer | Required when `ticket_type=facility` |
| `asset_id` | Conditional | integer | Required when `ticket_type=asset` |

## Behavior

- Tenant can only access own tickets (`owner_id = authenticated tenant`).
- Delete is treated as ticket withdrawal and soft-deletes the ticket.

## Sample Create Request

```json
{
  "title": "Water leakage in unit",
  "description": "There is a leak from the ceiling.",
  "ticket_type": "other",
  "ticket_category_id": 1,
  "facility_id": 2
}
```

