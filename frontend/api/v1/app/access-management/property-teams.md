# Access Management: Property Teams

Base prefix:

`/api/v1/app/{company}/access-management`

A property team is a group of staff led by one of them (usually a senior property manager) that owns a set of properties. Leader and members are **user ids** (`users.id`, the same key as property allocation and property roles), not `company_users.id`.

## Endpoints

- `GET /property-teams`
- `POST /property-teams`
- `GET /property-teams/{propertyTeam}`
- `PUT/PATCH /property-teams/{propertyTeam}`
- `DELETE /property-teams/{propertyTeam}`

Route names: `app.access-management.property-teams.{index,store,show,update,destroy}`.

## Permissions

| Action | Permission |
| --- | --- |
| List, show | `view-property-team` |
| Create | `create-property-team` |
| Update | `update-property-team` |
| Delete | `delete-property-team` |

All four are staff permissions (tag `Access Management`). `GET /me/permissions` lists `view-property-team` and `create-property-team` under `modules["Access Management"]`; like every module, it never carries `update-`/`delete-` names. Gate edit/delete on each row's `permissions` (`{ view, update, delete }`). A request without the permission returns `403`.

## Company Scope

- The company comes from route `{company}`. Do not send `company_id` (it is `prohibited`, `422`).
- A team of another company returns `404` on show/update/delete.

## List Property Teams

`GET /api/v1/app/{company}/access-management/property-teams`

- Filters:
  - `filter[search]`: team name or description, or the leader's name; a CSV of ids (`3,5,9`) returns those teams
  - `filter[status]`: `active` | `inactive`
  - `filter[leader_user_id]`: user id of the leader
- Include: `include=leader,members,facilities` (`leader` is always loaded; `members`/`facilities` only when included)
- Sort: `sort=name`, `sort=created_at` (prefix `-` for descending). Default: newest first.
- Pagination: `per_page`, `page`

Each row always carries `leader`, `members_count` and `facilities_count`. To find which team owns each property (e.g. to disable taken properties in a picker), call `?include=facilities&per_page=200`.

## Show Property Team

`GET /api/v1/app/{company}/access-management/property-teams/{propertyTeam}`

Always loads the leader, the members (with their role names in this company, sorted by name) and the properties (sorted by name).

## Resource

```json
{
  "id": 4,
  "name": "Alpha",
  "description": "Westlands portfolio",
  "status": { "value": "active", "color": "success", "label": "Active" },
  "emblem_url": "https://api.example.com/api/v1/common/settings/file-management/uploads/9b1c.../preview",
  "emblem_upload_id": 812,
  "leader": { "id": 31, "name": "Grace Wanjiru", "email": "grace@example.com" },
  "members_count": 3,
  "facilities_count": 2,
  "members": [
    { "id": 31, "name": "Grace Wanjiru", "email": "grace@example.com", "roles": ["Senior Property Manager"] }
  ],
  "facilities": [
    { "id": 120, "name": "Alpha Towers", "status": { "value": "active", "color": "success", "label": "Active" } }
  ],
  "created": { "raw": "2026-11-03T09:12:00.000000Z", "formatted": "03 Nov, 2026", "diff": "1 hour ago" },
  "permissions": { "view": true, "update": true, "delete": true }
}
```

- `leader` is `null` when no leader is set (or the leader user was deleted).
- `emblem_url` is `null` when no emblem is set or the upload is gone.
- `members` and `facilities` are present on show/store/update, and on index only with `include`.

## Response Envelopes

- Index: a paginated collection (`data`, `links`, `meta`).
- Show: `{ "data": { "property_team": { ... } } }`
- Store/Update: `{ "data": { "message": "Property team created successfully.", "property_team": { ... } } }` (`"Property team updated successfully."` on update)
- Delete: `{ "data": { "message": "Property team deleted successfully." } }`

## Create Payload

`POST /api/v1/app/{company}/access-management/property-teams`

| Field | Type | Required | Default | Notes |
| --- | --- | --- | --- | --- |
| `name` | string | Yes | - | Max 255, unique per company (names of deleted teams stay taken) |
| `description` | string&#124;null | No | `null` | |
| `leader_user_id` | integer&#124;null | No | `null` | `users.id`; must be an **active** staff member of this company. Always added to the members. |
| `emblem_upload_id` | integer&#124;null | No | `null` | Id of an upload (upload the image first through file management, e.g. title "Team Emblem") |
| `member_user_ids` | integer[] | No | `[]` | `users.id` values; each must be an active staff member of this company |
| `facility_ids` | integer[] | No | `[]` | Properties of this company; a property can belong to **one team only** |
| `status` | string | No | `active` | `active` or `inactive` |

Example:

```json
{
  "name": "Alpha",
  "description": "Westlands portfolio",
  "leader_user_id": 31,
  "emblem_upload_id": 812,
  "member_user_ids": [31, 44, 52],
  "facility_ids": [120, 121]
}
```

## Update Payload

`PUT/PATCH /api/v1/app/{company}/access-management/property-teams/{propertyTeam}`

Same fields as create, all optional; an omitted field is left unchanged.

- `member_user_ids` given: members become exactly that list **plus the leader**.
- `member_user_ids` omitted but `leader_user_id` changed: the new leader is added and nobody is removed.
- `facility_ids` given: the team's properties become exactly that list (the team's own current properties are allowed).
- `emblem_upload_id: null` removes the emblem.

## Validation (`422`)

| Field | When |
| --- | --- |
| `name` | missing on create, too long, or already used by another team of this company |
| `leader_user_id` | not an active staff member of this company: "The selected leader is not active staff of this company." |
| `member_user_ids` | any member not active staff of this company: "One or more selected members are not active staff of this company." |
| `facility_ids` | a property of another company: "One or more selected properties are invalid for this company." / a property already in another team, one message per property: "Alpha Towers already belongs to the Beta team." |
| `company_id` | sent at all |

## Delete

Soft-deletes the team and **releases its properties**, so they can be assigned to another team straight away. The deleted team then returns `404`.
