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

