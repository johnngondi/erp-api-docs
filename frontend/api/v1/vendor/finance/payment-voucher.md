# Vendor Payment Vouchers API

Domain: `Vendor Portal > Finance`

Base route:

`/api/v1/vendor/finance/payment-vouchers`

## Endpoints

- `GET /payment-vouchers`
- `GET /payment-vouchers/{paymentVoucher}`

## List Payment Vouchers

`GET /api/v1/vendor/finance/payment-vouchers`

Notes:

- Results are restricted to vouchers where `payable_as=vendor` and `payable_user_id` equals the authenticated vendor.

Supported query params:

- Filters:
  - `filter[search]` (Scout-backed search; supports CSV IDs and payment advice numbers)
  - `filter[credit_bank_account_id]`, `filter[status]`
  - `filter[created_at]`, `filter[released_at]`, `filter[paid_at]`
- Sort:
  - `sort=id,amount,paid_at,released_at,created_at,updated_at`
- Include:
  - `include=paymentVoucherItems`
- Pagination:
  - `per_page`, `page`

Enum filter options:

- `filter[status]`: `pending`, `released`, `paid`, `cancelled` (from `PaymentVoucherStatus` enum)

Sample list response (`App\\PropertyManagement\\FacilityPaymentVoucherResource`):

```json
{
  "data": [
    {
      "id": 801,
      "payable_as": "vendor",
      "notes": "Batch payout",
      "amount": "2500.00",
      "payment_advice_number": "ADV-801",
      "paid_at": {
        "raw": "2026-05-08T00:00:00.000000Z",
        "formatted": "08 May, 2026",
        "diff": "1 day ago"
      },
      "released_at": {
        "raw": "2026-05-07T00:00:00.000000Z",
        "formatted": "07 May, 2026",
        "diff": "2 days ago"
      },
      "status": { "value": "released", "color": "info" },
      "payable_user": { "id": 7, "name": "Acme Vendor" },
      "credit_bank_account": { "id": 13, "account_name": "Operations Account" },
      "currency": { "id": 1, "code": "KES" },
      "items": [
        { "id": 1, "amount": "1000.00" }
      ],
      "created": {
        "raw": "2026-05-09T10:15:00.000000Z",
        "formatted": "09 May, 2026",
        "diff": "moments ago"
      },
      "updated": {
        "raw": "2026-05-09T10:15:00.000000Z",
        "formatted": "09 May, 2026",
        "diff": "moments ago"
      }
    }
  ]
}
```

## Show Payment Voucher

`GET /api/v1/vendor/finance/payment-vouchers/{paymentVoucher}`

Notes:

- Returns `404` when the voucher does not belong to the authenticated vendor.
- Response shape matches the list resource.
