# Access Management: Companies

Base prefix:

`/api/v1/app/{company}/access-management`

## Endpoint Shape

Implemented:

- `GET /companies`
- `POST /companies`
- `GET /companies/{managedCompany}`
- `PUT/PATCH /companies/{managedCompany}`
- `DELETE /companies/{managedCompany}`

`{company}` is request context.
`{managedCompany}` is the target company for show/update/delete.

## List Companies

`GET /api/v1/app/{company}/access-management/companies`

Returns companies accessible to current user (owner or active member).

Supported query params:

- Filters:
  - `filter[search]` (Scout search; supports CSV company ids e.g. `3,8,12`)
  - `filter[status]`
  - `filter[created_at]` (date: `YYYY-MM-DD`)
- Include:
  - `include=companyUsers`
- Sort:
  - `sort=name`, `sort=status`, `sort=created_at`
- Pagination:
  - `per_page`, `page`

Selectable status values for dropdown:

- `active`
- `inactive`

## Create Company

`POST /api/v1/app/{company}/access-management/companies`

Create payload table:

| Field | Type | Required | Default | Notes |
| --- | --- | --- | --- | --- |
| `name` | string | Yes | - | Company name |
| `tax_pin` | string&#124;null | No | `null` | Tax PIN |
| `withholds` | array[int]&#124;null | No | `null` | IDs from `withholding_taxes` |
| `has_vat` | boolean&#124;null | No | `false` | VAT enabled flag |
| `service_managed` | string&#124;null | No | `null` | `rent` or `sc` or `both` |
| `address` | string&#124;null | No | `null` | Address text |
| `profile_photo_path` | string&#124;null | No | `null` | Optional image path |
| `type_of_properties` | array&#124;null | No | `null` | Optional array |
| `collection_contract` | boolean | No | `false` | Collection contract enabled flag |
| `registration_type` | string&#124;null | No | `null` | `national_id` or `business_license` or `passport` |
| `registration_number` | string&#124;null | No | `null` | Registration number |
| `registration_upload_id` | int&#124;null | No | `null` | Upload ID |
| `tax_pin_cert_upload_id` | int&#124;null | No | `null` | Upload ID |
| `other_docs_upload_id` | int&#124;null | No | `null` | Upload ID |
| `status` | string | No | `active` | `active` or `inactive` |

Compatibility alias:

- `witholds` is accepted and normalized to `withholds`.

## Show / Update / Delete Company

- `GET /companies/{managedCompany}`
- `PATCH /companies/{managedCompany}`
- `DELETE /companies/{managedCompany}`

Access is validated against the target company (`managedCompany`), not only route context company.
