# Access Management: Permissions

Base prefix:

`/api/v1/app/{company}/access-management`

## Endpoints

Implemented:

- `GET /permissions/{group}`

## List Permissions by User Group

`GET /api/v1/app/{company}/access-management/permissions/{group}`

Path params:

- `group`: user group id.

Response:

- `data`: permission collection for the selected user group.
