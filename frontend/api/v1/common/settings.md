# Common Settings API

Base route:

`/api/v1/settings`

All endpoints in this document require `auth:sanctum`.

## General

- `GET /settings/facility-types`
- `GET /settings/asset-types`

## Accounting

- `GET|POST /settings/accounting/taxes`
- `GET|PUT|PATCH|DELETE /settings/accounting/taxes/{tax}`
- `GET|POST /settings/accounting/withholdings`
- `GET|PUT|PATCH|DELETE /settings/accounting/withholdings/{withholding}`
- `GET|POST /settings/accounting/currencies`
- `GET|PUT|PATCH|DELETE /settings/accounting/currencies/{currency}`
- `GET|POST /settings/accounting/banks`
- `GET|PUT|PATCH|DELETE /settings/accounting/banks/{bank}`
- `GET|POST /settings/accounting/payment-methods`
- `GET|PUT|PATCH|DELETE /settings/accounting/payment-methods/{paymentMethod}`
- `PATCH /settings/accounting/payment-methods/{paymentMethod}/activate`
- `PATCH /settings/accounting/payment-methods/{paymentMethod}/deactivate`

## Procurement

- `GET|POST /settings/skus`
- `GET|PUT|PATCH|DELETE /settings/skus/{sku}`
- `GET|POST /settings/procurement/skus`
- `GET|PUT|PATCH|DELETE /settings/procurement/skus/{sku}`
- `GET|POST /settings/procurement/inventory-categories`
- `GET|PUT|PATCH|DELETE /settings/procurement/inventory-categories/{inventoryCategory}`

## Power Management

- `GET|POST /settings/power-management/fuel-types`
- `GET|PUT|PATCH|DELETE /settings/power-management/fuel-types/{fuel_type}`
- `PATCH /settings/power-management/fuel-types/{fuelType}/activate`
- `PATCH /settings/power-management/fuel-types/{fuelType}/deactivate`
- `GET|POST /settings/power-management/power-source-types`
- `GET|PUT|PATCH|DELETE /settings/power-management/power-source-types/{source_type}`
- `PATCH /settings/power-management/power-source-types/{powerSourceType}/activate`
- `PATCH /settings/power-management/power-source-types/{powerSourceType}/deactivate`

## Fleet

- `GET|POST /settings/fleet/vehicle-categories`
- `GET|PUT|PATCH|DELETE /settings/fleet/vehicle-categories/{vehicleCategory}`
- `GET|POST /settings/fleet/vehicle-models`
- `GET|PUT|PATCH|DELETE /settings/fleet/vehicle-models/{vehicleModel}`
- `GET|POST /settings/fleet/vehicle-makes`
- `GET|PUT|PATCH|DELETE /settings/fleet/vehicle-makes/{vehicleMake}`

## File Management

- `GET|POST /settings/file-management/uploads`
- `GET|PUT|PATCH|DELETE /settings/file-management/uploads/{upload}`
- `GET /settings/file-management/uploads/{upload:uuid}/preview` (public)
- `GET /settings/file-management/uploads/{upload:uuid}/download` (public)

## Companies

- `GET|POST /settings/companies`
- `GET|PUT|PATCH|DELETE /settings/companies/{company}`
- `GET|POST /settings/companies/{company}/users`
- `GET|PUT|PATCH|DELETE /settings/companies/{company}/users/{companyUser}`

Scoped to companies owned by the current user (`user_id`). List supports
`filter[search]`, `filter[id]`, `filter[status]`, `filter[created_at]`,
`include=companyUsers`, `sort=name|status`, and `per_page`.

Create / update payload:

| Field | Type | Required | Default | Notes |
| --- | --- | --- | --- | --- |
| `name` | string | Yes | - | Company name |
| `tax_pin` | string&#124;null | No | `null` | Tax PIN |
| `withholds` | array[int]&#124;null | No | `null` | IDs from `withholding_taxes` |
| `has_vat` | boolean&#124;null | No | `false` | VAT enabled flag |
| `service_managed` | string&#124;null | No | `null` | `rent` or `sc` or `both` |
| `address` | string&#124;null | No | `null` | Address text |
| `company_phone` | string&#124;null | No | `null` | Public contact phone |
| `company_email` | string&#124;null | No | `null` | Public contact email |
| `company_website` | string&#124;null | No | `null` | Website URL or domain |
| `company_tagline` | string&#124;null | No | `null` | Short tagline / slogan |
| `profile_photo_path` | string&#124;null | No | `null` | Company logo storage path from `POST /settings/file-management/uploads` |
| `type_of_properties` | array&#124;null | No | `null` | Optional array |
| `collection_contract` | boolean | No | `false` | Collection contract enabled flag |
| `registration_type` | string&#124;null | No | `null` | `national_id` or `business_license` or `passport` |
| `registration_number` | string&#124;null | No | `null` | Registration number |
| `registration_upload_id` | int&#124;null | No | `null` | Upload ID |
| `tax_pin_cert_upload_id` | int&#124;null | No | `null` | Upload ID |
| `other_docs_upload_id` | int&#124;null | No | `null` | Upload ID |
| `status` | string | No | `active` | `active` or `inactive` (the `companies.status` column enum; `CompanyData` also lists `suspended`, but the column rejects it) |

`witholds` is accepted as an alias for `withholds`.

Update caveat — send the full object:

- `PUT/PATCH /settings/companies/{company}` builds a complete `CompanyData` from the request, so **omitted fields are overwritten with their defaults** (`null`, or `false` for `has_vat` / `collection_contract`). Always send the full object when updating here.
- The access-management endpoint (`PATCH /api/v1/app/{company}/access-management/companies/{managedCompany}`) leaves omitted fields unchanged and is the better fit for partial edits.

Responses use the same company object as the access-management module — see
`docs/frontend/api/v1/app/access-management/companies.md` for the full key list,
including `profile_photo_path` and the resolved `profile_photo_url`.

