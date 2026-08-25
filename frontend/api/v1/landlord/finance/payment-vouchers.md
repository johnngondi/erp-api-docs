# Landlord Payment Vouchers API

Base route:

`/api/v1/landlord/finance/payment-vouchers`

## Endpoints

- `GET /finance/payment-vouchers`
- `GET /finance/payment-vouchers/{paymentVoucher}`

## List Filters

- `filter[search]`
- `filter[credit_bank_account_id]`
- `filter[status]`
- `filter[created_at]`
- `filter[released_at]`
- `filter[paid_at]`

## Enum Filter Options

- `filter[status]`: `pending`, `released`, `paid`, `cancelled`

## Sorts

- `sort=id,amount,paid_at,released_at,created_at,updated_at`

## Includes

- `include=paymentVoucherItems`
- `include=paymentVoucherItems.currency`
- `include=paymentAdviceUpload`
- `include=releasingUser`

## Scope

- Returns only vouchers where:
- `payable_as = landlord`
- `payable_user_id = authenticated landlord`

## Show Behavior

- `show` loads:
- `creditBankAccount` with nested `bankBranch.bank` details (`id`, `name`)
- `currency`
- `paymentVoucherItems` and item `currency`
- `paymentAdviceUpload`
- `releasingUser`
