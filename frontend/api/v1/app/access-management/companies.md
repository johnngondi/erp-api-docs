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

Key fields:

- `name` (required)
- `tax_pin`, `has_vat`, `service_managed`, `address`
- `withholds` (array of `withholding_taxes.id`)
- `type_of_properties`, `collection_contract`
- `registration_type`, `registration_number`
- `registration_upload_id`, `tax_pin_cert_upload_id`, `other_docs_upload_id`

Compatibility alias:

- `witholds` is accepted and normalized to `withholds`.

## Show / Update / Delete Company

- `GET /companies/{managedCompany}`
- `PATCH /companies/{managedCompany}`
- `DELETE /companies/{managedCompany}`

Access is validated against the target company (`managedCompany`), not only route context company.
