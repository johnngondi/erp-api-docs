# Landlord Bank Accounts API

Base route:

`/api/v1/landlord/finance/bank-accounts`

## Endpoints

- `GET /finance/bank-accounts`

## List Filters

- `filter[search]`
- `filter[status]`
- `filter[bank_branch_id]`
- `filter[created_at]`

## Enum Filter Options

- `filter[status]`: `active`, `inactive`

## Sorts

- `sort=id,account_name,account_number,status,created_at,updated_at`

## Includes

- No include parameters are supported for this endpoint.

## Default Loaded Relations

- `bankBranch`
- `bankBranch.bank`

## Scope

- Returns only accounts where `user_id = authenticated landlord`.
