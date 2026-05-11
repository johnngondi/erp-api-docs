# Landlord Remittances API

Base route:

`/api/v1/landlord/finance/remittances`

## Endpoints

- `GET /finance/remittances`
- `GET /finance/remittances/{remittance}`

## List Filters

- `filter[search]`
- `filter[facility_id]`
- `filter[status]`
- `filter[created_at]`
- `filter[period_from]`
- `filter[period_to]`
- `filter[is_advance]`

## Enum Filter Options

- `filter[status]`: `pending`, `unpaid`, `paid`, `cancelled`

## Sorts

- `sort=id,period_from,period_to,total_income,total_expenses,remittable_amount,created_at,updated_at`

## Includes

- `include=collections`
- `include=expenses`

## Scope

- Returns only remittances where `landlord_id = authenticated landlord`.

## Show Behavior

- `show` loads:
- Facility summary (`facility`)
- Currency summary (`currency`)
- Collections (via landlord `LeaseCollectionResource`)
- Expenses (via landlord `FacilityExpenseResource`)
