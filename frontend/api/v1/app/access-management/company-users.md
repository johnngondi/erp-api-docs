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

Create payload table (`company_ids.*` entry):

| Field | Type | Required | Default | Notes |
| --- | --- | --- | --- | --- |
| `company_id` | int | Yes | - | Target company ID |
| `role` | string | No | `staff` | `staff` or `agent` |
| `roles` | array[int] | Yes | - | Role IDs scoped to that company and app group |
| `facilities` | array[int] | No | - | Property (facility) IDs scoped to that company. Authoritative within that company: the user ends up with exactly these properties there |
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
      "facilities": [3, 7],
      "company_office_id": 1,
      "company_department_id": 1,
      "company_department_division_id": 1
    },
    {
      "company_id": 15,
      "role": "agent",
      "roles": [9],
      "facilities": [22]
    }
  ]
}
```

Rules:

- `company_ids` is required and must be unique by `company_id`.
- Caller must have access to every target company.
- Each `roles` entry must belong to the given `company_id` and `app` user group.
- If provided, `company_office_id` / `company_department_id` / `company_department_division_id` must belong to that assignment `company_id`.
- If `company_department_division_id` is provided, `company_department_id` is required and division must belong to that department.
- If user does not exist, backend creates user and assigns app user-group.
- `facilities` is optional per assignment entry. Every id must belong to that entry's `company_id`; a property from another company returns `422` on `company_ids`.
- Property assignment is what drives property visibility across the app, so a user only sees the properties listed here (plus any granted by naming them to a property role).

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
| `facilities` | array[int] | No | unchanged | Property (facility) IDs for the route company. Omit to leave access untouched; send `[]` to clear access for this company; send a list to sync to exactly that list |

## Property Assignment

Property access lives on each company assignment, exactly like `roles` does, because a
property belongs to one company. A staff user in two companies has an independent
property list in each.

Scope rules:

- Every id must belong to the company it is sent under. Cross-company ids are rejected
  with `422` (`company_ids` on create, `facilities` on update).
- Writes are scoped to that one company. Syncing, narrowing or clearing a user's
  properties in company A never changes the properties they hold in company B.
- On update, `facilities` targets the route company only, so `[]` clears that company's
  properties and leaves every other company's untouched.

Reading it back:

- `data.company_user.facility_ids` — ids, for populating a multi-select.
- `data.company_user.facilities` — `[{id, name}]`, for display.
- Both are returned on `index`, `show`, `store` and `update`, and both are scoped to the
  company of that `company_user` row, not to every property the user can reach.

Fetching the options list:

- Use `GET /api/v1/app/{company}/property-management/facilities`.
- The frontend already puts the selected company's id into that path segment when it
  resolves the URL, e.g. `api/v1/app/1/property-management/facilities` for company `1`.
  The backend scopes properties off that route company, so the response only ever
  contains that company's properties — there is no `company_id` filter to pass and no
  client-side filtering to do.
- On the staff form, refetch per company row using that row's `company_id` in the path,
  so each row offers only the properties that company owns.

Granted by role selection too:

- Naming a user against a role on the property create/update payload
  (`roles: [{role_id, user_id}]`) also grants that user access to that property.
- That path is additive: it never removes access the user holds to other properties, in
  that company or any other.

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
