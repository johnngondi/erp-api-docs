# Payment Vouchers API

Domain: `Property Management > Finance`

Base route:

`/api/v1/app/{company}/property-management/finance/payment-vouchers`

## Endpoints

- `GET /payment-vouchers`
- `GET /payment-vouchers/create` (options preview)
- `POST /payment-vouchers`
- `GET /payment-vouchers/{paymentVoucher}`
- `DELETE /payment-vouchers/{paymentVoucher}`
- `PATCH /payment-vouchers/{paymentVoucher}/cancel`
- `PATCH /payment-vouchers/{paymentVoucher}/release`
- `PUT /payment-vouchers/{paymentVoucher}/pay`

## List Payment Vouchers

`GET /api/v1/app/{company}/property-management/finance/payment-vouchers`

Supported query params:

- Filters:
  - `filter[search]` (Scout-backed search; supports CSV IDs and payment advice numbers)
  - `filter[payable_user_id]`, `filter[credit_bank_account_id]`
  - `filter[payable_as]`, `filter[status]`
  - `filter[created_at]`, `filter[released_at]`, `filter[paid_at]`
- Sort:
  - `sort=id,amount,paid_at,released_at,created_at,updated_at`
- Include:
  - `include=paymentVoucherItems`
- Pagination:
  - `per_page`, `page`

Enum filter options:

- `filter[payable_as]`: `vendor`, `landlord`, `tenant`, `other` (from `FacilityPaymentVoucherData`)
- `filter[status]`: `pending`, `released`, `paid`, `cancelled` (from `PaymentVoucherStatus` enum)

Sample list response (`FacilityPaymentVoucherResource`):

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

## Create payload (`PaymentVoucherData`)

| Field | Required | Type | Allowed Values / Notes |
|---|---|---|---|
| `payable_user_id` | Yes | integer | Must exist in `users.id` |
| `payable_as` | Yes | string | `vendor`, `landlord`, `tenant`, `other` |
| `credit_bank_account_id` | Yes | integer | Must exist in `bank_accounts.id` |
| `items` | Yes | array | At least 1 item |
| `currency_id` | No | integer | Must exist in `currencies.id` |

`items[]` for create:

| Field | Required | Type | Allowed Values / Notes |
|---|---|---|---|
| `payable_id` | Yes | integer | Expense/Remittance id depending on `payable_as` |
| `amount` | Yes | number | Must not exceed payable balance/remittable amount |

## Pay payload (`PaymentAdviceData`)

| Field | Required | Type |
|---|---|---|
| `payment_advice_number` | Yes | string |
| `payment_advice_upload_id` | Yes | integer (`uploads.id`) |
