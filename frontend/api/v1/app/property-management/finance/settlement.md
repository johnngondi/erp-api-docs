# Settlements API

Domain: `Property Management > Finance`

Base route:

`/api/v1/app/{company}/property-management/finance/settlements`

## Endpoints

- `GET /settlements`
- `POST /settlements`
- `GET /settlements/{settlement}`
- `PUT/PATCH /settlements/{settlement}`
- `DELETE /settlements/{settlement}`
- `PATCH /settlements/{settlement}/cancel`
- `PUT /settlements/{settlement}/settle`

## List Settlements

`GET /api/v1/app/{company}/property-management/finance/settlements`

Supported query params:

- Filters:
  - `filter[search]` (Scout-backed search; supports CSV IDs and payment advice numbers)
  - `filter[debit_bank_account_id]`, `filter[status]`, `filter[created_at]`
- Sort:
  - `sort=id,created_at,updated_at`
- Include:
  - `include=paymentVouchers`
- Pagination:
  - `per_page`, `page`

Enum filter options:

- `filter[status]`: `pending`, `settled`, `cancelled` (from `SettlementStatus` enum)

Sample list response (`FacilitySettlementResource`):

```json
{
  "data": [
    {
      "id": 120,
      "debit_bank_account": { "id": 13, "account_name": "System Disbursement Account" },
      "payment_advice_number": "SET-ADV-120",
      "payment_advice_upload": { "id": 45, "name": "set-adv-120.pdf" },
      "status": { "value": "pending", "color": "info" },
      "payment_vouchers": [
        { "id": 801, "amount": "2500.00" }
      ],
      "created_at": {
        "raw": "2026-05-09T10:20:00.000000Z",
        "formatted": "09 May, 2026",
        "diff": "moments ago"
      },
      "updated_at": {
        "raw": "2026-05-09T10:20:00.000000Z",
        "formatted": "09 May, 2026",
        "diff": "moments ago"
      }
    }
  ]
}
```

## Create payload (`SettlementData`)

| Field | Required | Type | Allowed Values / Notes |
|---|---|---|---|
| `debit_bank_account_id` | Yes | integer | Must be an active `system` bank account |
| `payment_vouchers` | Yes | array[int] | Vouchers must be `released` and unused in pending/settled settlement |
| `payment_advice_number` | No | string | Optional |
| `payment_advice_upload_id` | No | integer | Must exist in `uploads.id` |

## Settle payload

| Field | Required | Type |
|---|---|---|
| `payment_advice_number` | Yes | string |
| `payment_advice_upload_id` | Yes | integer (`uploads.id`) |
