# Permissions API

Domain: `Users`

Base prefix:

`/api/v1/app/{company}/users/permissions`

## Endpoint

- `GET /permissions/{group}`

`{group}` is the `user_group_id` integer.

## Get Permissions By User Group

`GET /api/v1/app/{company}/users/permissions/{group}`

Supported query params:

- Filter: not supported
- Sort: not supported
- Include: not supported
- Select fields: not supported

Example:

- `GET /api/v1/app/12/users/permissions/4`

Example response:

```json
{
  "data": [
    {
      "id": 17,
      "tag": "finance",
      "name": "View Bill"
    },
    {
      "id": 18,
      "tag": "finance",
      "name": "Approve Bill"
    }
  ]
}
```

## Errors

Use shared behavior in `docs/frontend/app/README.md`:

- Show backend `4xx` messages and validation errors to users.
- Show generic fallback for `5xx` responses.
