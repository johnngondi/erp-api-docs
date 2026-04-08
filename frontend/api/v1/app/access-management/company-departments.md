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

## List Company Departments

`GET /api/v1/app/{company}/access-management/company-departments`

Supported query params:

- Filters:
  - `filter[search]` (Scout search; supports CSV department ids e.g. `3,5,9`)
  - `filter[status]`
  - `filter[created_at]` (date: `YYYY-MM-DD`)
- Include:
  - `include=companyDepartmentDivisions`
- Sort:
  - `sort=name`, `sort=code`, `sort=status`, `sort=created_at`
- Pagination:
  - `per_page`, `page`

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
