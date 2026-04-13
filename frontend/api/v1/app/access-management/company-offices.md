# Access Management: Company Offices

Base prefix:

`/api/v1/app/{company}/access-management`

## Endpoints

Implemented:

- `GET /company-offices`
- `POST /company-offices`
- `GET /company-offices/{companyOffice}`
- `PUT/PATCH /company-offices/{companyOffice}`
- `DELETE /company-offices/{companyOffice}`

## Company Scope

- `company_id` is resolved from route `{company}`.
- Do not send `company_id` in create/update payload.
- For `index` only:
  - Without `filter[company_id]`: results are scoped to route `{company}`.
  - With `filter[company_id]`: results can include accessible companies.

## List Company Offices

`GET /api/v1/app/{company}/access-management/company-offices`

Supported query params:

- Filters:
  - `filter[search]` (Scout search; supports CSV office ids e.g. `12,18`)
  - `filter[company_id]` (single, CSV, or array of accessible company ids)
  - `filter[country_id]`
  - `filter[city_id]`
  - `filter[status]`
  - `filter[created_at]` (date: `YYYY-MM-DD`)
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

`POST /api/v1/app/{company}/access-management/company-offices`

| Field | Type | Required | Default | Notes |
| --- | --- | --- | --- | --- |
| `name` | string | Yes | - | Office name |
| `code` | string&#124;null | No | `null` | Optional code |
| `email` | string&#124;null | No | `null` | Office email |
| `phone` | string&#124;null | No | `null` | Office phone |
| `country_id` | int&#124;null | No | `null` | Country ID |
| `city_id` | int&#124;null | No | `null` | City ID |
| `address` | string&#124;null | No | `null` | Address |
| `postal_code` | string&#124;null | No | `null` | Postal code |
| `status` | string | No | `active` | `active` or `inactive` |

## Update Payload

`PUT/PATCH /api/v1/app/{company}/access-management/company-offices/{companyOffice}`

Same fields as create, all optional, default behavior is `unchanged` when omitted.
