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

Default behavior returns only users for route company `{company}`.

Supported query params:

- Filters:
  - `filter[search]` (Scout search; supports CSV company-user ids e.g. `101,104`)
  - `filter[company_id]` (single, CSV, or array of accessible company ids)
  - `filter[role]`
  - `filter[status]`
  - `filter[company_office_id]`
  - `filter[company_department_id]`
  - `filter[company_department_division_id]`
  - `filter[created_at]` (date: `YYYY-MM-DD`)
- Sort:
  - `sort=role`, `sort=status`, `sort=created_at`
- Pagination:
  - `per_page`, `page`

`filter[company_id]` notes:

- Omitted: defaults to route `{company}`.
- Present: returns users across the selected accessible companies.
- Inaccessible company ids return `422`.

Selectable dropdown values:

- `role`: `staff`, `agent`
- `status`: `active`, `suspended`, `inactive`

## Create Company User (Independent + Multi-Company Assignments)

`POST /api/v1/app/{company}/access-management/company-users`

Create payload table (root):

| Field | Type | Required | Default | Notes |
| --- | --- | --- | --- | --- |
| `name` | string | Yes | - | User name |
| `email` | string | Yes | - | User email |
| `phone` | string | Yes | - | User phone |
| `company_ids` | array[object] | Yes | - | Company assignment entries |
| `facilities` | array[int] | No | - | Property (facility) IDs the staff user is assigned to. Authoritative: the user ends up with exactly these. |

Create payload table (`company_ids.*` entry):

| Field | Type | Required | Default | Notes |
| --- | --- | --- | --- | --- |
| `company_id` | int | Yes | - | Target company ID |
| `role` | string | No | `staff` | `staff` or `agent` |
| `roles` | array[int] | Yes | - | Role IDs scoped to that company and app group |
| `company_office_id` | int&#124;null | No | `null` | Must belong to `company_id` |
| `company_department_id` | int&#124;null | No | `null` | Must belong to `company_id` |
| `company_department_division_id` | int&#124;null | No | `null` | Must belong to `company_id` and selected department |

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
      "roles": [1, 5, 6],
      "company_office_id": 1,
      "company_department_id": 1,
      "company_department_division_id": 1
    }
  ],
  "facilities": [3, 7]
}
```

Rules:

- `company_ids` is required and must be unique by `company_id`.
- Caller must have access to every target company.
- Each `roles` entry must belong to the given `company_id` and `app` user group.
- If provided, `company_office_id` / `company_department_id` / `company_department_division_id` must belong to that assignment `company_id`.
- If `company_department_division_id` is provided, `company_department_id` is required and division must belong to that department.
- If user does not exist, backend creates user and assigns app user-group.
- `facilities` is optional. Each id must exist. When sent, it becomes the user's full property access list, which is what drives property visibility across the app.

## Show / Update / Delete

- `GET /company-users/{companyUser}`
- `PATCH /company-users/{companyUser}`
- `DELETE /company-users/{companyUser}`

Update payload supports:

| Field | Type | Required | Default | Notes |
| --- | --- | --- | --- | --- |
| `role` | string | No | unchanged | `staff` or `agent` |
| `roles` | array[int] | No | unchanged | Role IDs for current route company |
| `company_office_id` | int&#124;null | No | unchanged | Set `null` to clear |
| `company_department_id` | int&#124;null | No | unchanged | Set `null` to clear; clearing also clears division |
| `company_department_division_id` | int&#124;null | No | unchanged | Requires `company_department_id` when provided |
| `facilities` | array[int] | No | unchanged | Property (facility) IDs. Omit to leave access untouched; send `[]` to remove all access; send a list to sync to exactly that list |

Property assignment notes:

- Read back from `data.company_user.facility_ids` (ids, for a multi-select) and `data.company_user.facilities` (`[{id, name}]`), on `index`, `show`, `store` and `update`.
- Naming a user against a role on the property create/update payload (`roles: [{role_id, user_id}]`) also grants that user property access. That path is additive and never removes access the user holds to other properties.

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
