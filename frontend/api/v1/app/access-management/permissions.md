# Access Management: Permissions

Base prefix:

`/api/v1/app/{company}/access-management`

## Endpoints

Implemented:

- `GET /permissions`

## Create Payload

`POST /api/v1/app/{company}/access-management/permissions`

| Field | Type | Required | Default | Notes |
| --- | --- | --- | --- | --- |
| N/A | N/A | N/A | N/A | Create endpoint is not available for this module |

## List Permissions

`GET /api/v1/app/{company}/access-management/permissions`

Response:

- `data`: permission collection for selected user group scope.
- Default resolution order:
  - `app` group (if the user has it)
  - first assigned user group
- Optional filter:
  - `filter[user_group_id]`: return permissions for any user group id (supported on app portal routes).
