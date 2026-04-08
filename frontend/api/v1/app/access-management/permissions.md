# Access Management: Permissions

Base prefix:

`/api/v1/app/{company}/access-management`

## Endpoints

Implemented:

- `GET /permissions/{group}`

## Create Payload

`POST /api/v1/app/{company}/access-management/permissions`

| Field | Type | Required | Default | Notes |
| --- | --- | --- | --- | --- |
| N/A | N/A | N/A | N/A | Create endpoint is not available for this module |

## List Permissions by User Group

`GET /api/v1/app/{company}/access-management/permissions/{group}`

Path params:

- `group`: user group id.

Response:

- `data`: permission collection for the selected user group.
