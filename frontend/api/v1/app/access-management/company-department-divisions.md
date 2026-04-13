# Access Management: Company Department Divisions

Base prefix:

`/api/v1/app/{company}/access-management`

## Endpoints

Implemented (nested under department):

- `GET /company-departments/{companyDepartment}/divisions`
- `POST /company-departments/{companyDepartment}/divisions`
- `GET /company-departments/{companyDepartment}/divisions/{companyDepartmentDivision}`
- `PUT/PATCH /company-departments/{companyDepartment}/divisions/{companyDepartmentDivision}`
- `DELETE /company-departments/{companyDepartment}/divisions/{companyDepartmentDivision}`

## Scope Rules

- Default index behavior requires `{companyDepartment}` to belong to route `{company}`.
- With `filter[company_id]`, index allows accessible company scopes, and `{companyDepartment}` must belong to one of filtered company ids.
- Division is always scoped to its parent department.
- Do not send `company_id` or `company_department_id` in create/update payload.

## List Department Divisions

`GET /api/v1/app/{company}/access-management/company-departments/{companyDepartment}/divisions`

Supported query params:

- Filters:
  - `filter[search]` (Scout search; supports CSV division ids e.g. `10,21`)
  - `filter[company_id]` (single, CSV, or array of accessible company ids)
  - `filter[status]`
  - `filter[created_at]` (date: `YYYY-MM-DD`)
- Sort:
  - `sort=name`, `sort=code`, `sort=status`, `sort=created_at`
- Pagination:
  - `per_page`, `page`

`filter[company_id]` notes:

- Omitted: defaults to route `{company}` behavior.
- Inaccessible company ids return `422`.

Selectable dropdown values:

- `status`: `active`, `inactive`

## Create Payload

`POST /api/v1/app/{company}/access-management/company-departments/{companyDepartment}/divisions`

| Field | Type | Required | Default | Notes |
| --- | --- | --- | --- | --- |
| `name` | string | Yes | - | Division name |
| `code` | string&#124;null | No | `null` | Optional code |
| `description` | string&#124;null | No | `null` | Optional description |
| `status` | string | No | `active` | `active` or `inactive` |

## Update Payload

`PUT/PATCH /api/v1/app/{company}/access-management/company-departments/{companyDepartment}/divisions/{companyDepartmentDivision}`

Same fields as create, all optional, default behavior is `unchanged` when omitted.
