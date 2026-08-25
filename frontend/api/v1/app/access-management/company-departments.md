# Access Management: Company Departments

Base prefix:

`/api/v1/app/{company}/access-management`

## Endpoints

Implemented:

- `GET /company-departments`
- `POST /company-departments`
- `GET /company-departments/{companyDepartment}`
- `PUT/PATCH /company-departments/{companyDepartment}`
- `DELETE /company-departments/{companyDepartment}`

## Company Scope

- `company_id` is resolved from route `{company}`.
- Do not send `company_id` in create/update payload.
- For `index` only:
  - Without `filter[company_id]`: results are scoped to route `{company}`.
  - With `filter[company_id]`: results can include accessible companies.

## List Company Departments

`GET /api/v1/app/{company}/access-management/company-departments`

Supported query params:

- Filters:
  - `filter[search]` (Scout search; supports CSV department ids e.g. `3,5,9`)
  - `filter[company_id]` (single, CSV, or array of accessible company ids)
  - `filter[status]`
  - `filter[created_at]` (date: `YYYY-MM-DD`)
- Include:
  - `include=companyDepartmentDivisions`
- Sort:
  - `sort=name`, `sort=code`, `sort=status`, `sort=created_at`
- Pagination:
  - `per_page`, `page`

`filter[company_id]` notes:

- Omitted: defaults to route `{company}`.
- Inaccessible company ids return `422`.

Selectable dropdown values:

- `status`: `active`, `inactive`

## Create Payload

`POST /api/v1/app/{company}/access-management/company-departments`

| Field | Type | Required | Default | Notes |
| --- | --- | --- | --- | --- |
| `name` | string | Yes | - | Department name |
| `code` | string&#124;null | No | `null` | Optional code |
| `description` | string&#124;null | No | `null` | Optional description |
| `status` | string | No | `active` | `active` or `inactive` |

## Update Payload

`PUT/PATCH /api/v1/app/{company}/access-management/company-departments/{companyDepartment}`

Same fields as create, all optional, default behavior is `unchanged` when omitted.
