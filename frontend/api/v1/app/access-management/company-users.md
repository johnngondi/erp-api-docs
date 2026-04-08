# Access Management: Company Users

Base prefix:

`/api/v1/app/{company}/access-management`

## Endpoints

Implemented:

- `GET /company-users`
- `POST /company-users`
- `GET /company-users/{companyUser}`
- `PUT/PATCH /company-users/{companyUser}`
- `DELETE /company-users/{companyUser}`
- `PATCH /company-users/{companyUser}/activate`
- `PATCH /company-users/{companyUser}/suspend`
- `PATCH /company-users/{companyUser}/terminate`

## List Company Users

`GET /api/v1/app/{company}/access-management/company-users`

Returns only users for route company `{company}`.

Supported query params:

- Filters:
  - `filter[search]` (Scout search; supports CSV company-user ids e.g. `101,104`)
  - `filter[role]`
  - `filter[status]`
  - `filter[created_at]` (date: `YYYY-MM-DD`)
- Sort:
  - `sort=role`, `sort=status`, `sort=created_at`
- Pagination:
  - `per_page`, `page`

Selectable dropdown values:

- `role`: `staff`, `agent`
- `status`: `active`, `suspended`, `inactive`

## Create Company User (Independent + Multi-Company Assignments)

`POST /api/v1/app/{company}/access-management/company-users`

Request shape:

```json
{
  "name": "Jane Doe",
  "email": "jane@example.com",
  "phone": "0712345678",
  "company_ids": [
    {
      "company_id": 12,
      "role": "staff",
      "roles": [1, 5, 6]
    },
    {
      "company_id": 13,
      "role": "agent",
      "roles": [11, 12]
    }
  ]
}
```

Rules:

- `company_ids` is required and must be unique by `company_id`.
- Caller must have access to every target company.
- Each `roles` entry must belong to the given `company_id` and `app` user group.
- If user does not exist, backend creates user and assigns app user-group.

## Show / Update / Delete

- `GET /company-users/{companyUser}`
- `PATCH /company-users/{companyUser}`
- `DELETE /company-users/{companyUser}`

Update payload supports:

- `role` (`staff|agent`)
- `roles` (array of role IDs for current route company)

## Lifecycle Actions

- Activate:
  - `PATCH /company-users/{companyUser}/activate`
  - Sets `status=active`
- Suspend:
  - `PATCH /company-users/{companyUser}/suspend`
  - Sets `status=suspended`
- Terminate:
  - `PATCH /company-users/{companyUser}/terminate`
  - Sets `status=inactive`
