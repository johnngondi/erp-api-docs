# User Groups API

Domain: `Users`

Base prefix:

`/api/v1/app/{company}/users/user-groups`

## Endpoints

Currently implemented:

- `GET /user-groups`

Routes exist but are not implemented in controller:

- `POST /user-groups`
- `GET /user-groups/{user_group}`
- `PUT/PATCH /user-groups/{user_group}`
- `DELETE /user-groups/{user_group}`

## List User Groups

`GET /api/v1/app/{company}/users/user-groups`

Supported query params:

- Filters:
  - `filter[id]`
  - `filter[title]`
  - `filter[description]`
- Sort:
  - Not configured
- Include:
  - `include=users`
- Select fields:
  - Not supported
- Pagination:
  - `per_page`, `page`

Example:

- `GET /api/v1/app/12/users/user-groups?filter[title]=vendor&include=users`

Example response:

```json
{
  "data": [
    {
      "id": 4,
      "title": "Vendor",
      "description": "Third-party service providers",
      "can_register": false,
      "require_company": true
    }
  ]
}
```

## Errors

Use shared behavior in `docs/frontend/app/README.md`:

- Show backend `4xx` messages and validation errors to users.
- Show generic fallback for `5xx` responses.
