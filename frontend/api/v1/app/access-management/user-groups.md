# Access Management: User Groups

Base prefix:

`/api/v1/app/{company}/access-management`

## Endpoints

Currently implemented:

- `GET /user-groups`

Routes exist but are not implemented in controller:

- `POST /user-groups`
- `GET /user-groups/{user_group}`
- `PUT/PATCH /user-groups/{user_group}`
- `DELETE /user-groups/{user_group}`

## Create Payload

`POST /api/v1/app/{company}/access-management/user-groups`

| Field | Type | Required | Default | Notes |
| --- | --- | --- | --- | --- |
| N/A | N/A | N/A | N/A | Create endpoint is not implemented in backend yet |

## List User Groups

`GET /api/v1/app/{company}/access-management/user-groups`

Supported query params:

- Filters:
  - `filter[id]`
  - `filter[title]`
  - `filter[description]`
- Include:
  - `include=users`
- Pagination:
  - `per_page`, `page`

Example:

- `GET /api/v1/app/12/access-management/user-groups?filter[title]=app&include=users`
