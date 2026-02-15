# Receipts API

Domain: `Property Management > Lease Management`

Base route:

`/api/v1/app/{company}/property-management/lease-management/receipts`

## Endpoints

- `GET /receipts`
- `POST /receipts`
- `GET /receipts/{receipt}`
- `PUT/PATCH /receipts/{receipt}`
- `PATCH /receipts/{receipt}/cancel`
- `DELETE /receipts/{receipt}`

## List Receipts

`GET /api/v1/app/{company}/property-management/lease-management/receipts`

Supported query params:

- Filters:
  - `filter[id]`
  - `filter[receiving_account_id]`
  - `filter[transaction_number]`
  - `filter[transaction_date]`
  - `filter[payment_method_id]`
  - `filter[paying_user_id]`
  - `filter[amount]`
  - `filter[allocated]`
  - `filter[balance]`
  - `filter[receiving_user_id]`
- Sort:
  - `sort=id,receiving_account_id,transaction_number,transaction_date,payment_method_id,paying_user_id,amount,allocated,balance,receiving_user_id`
- Include: not supported
- Select fields: not supported
- Pagination: `per_page`, `page`

Example:

`GET /api/v1/app/12/property-management/lease-management/receipts?filter[transaction_number]=RCPT-1005&sort=-transaction_date`

## Create Receipt

`POST /api/v1/app/{company}/property-management/lease-management/receipts`

Request body:

| Field | Required | Type | Allowed Values / Notes |
|---|---|---|---|
| `transaction_number` | Yes | string | Unique business transaction number |
| `receiving_account_id` | Yes | integer | Must exist in `bank_accounts.id` |
| `transaction_date` | Yes | date (`YYYY-MM-DD`) | - |
| `payment_method_id` | Yes | integer | Must exist in `payment_methods.id` |
| `paying_user_id` | Yes | integer | Must exist in `users.id` |
| `amount` | Yes | integer | - |
| `currency_id` | No | integer | Must exist in `currencies.id` |
| `allocations` | Yes | array | Invoice allocations |

`allocations[]` fields:

| Field | Required | Type | Allowed Values / Notes |
|---|---|---|---|
| `invoice_id` | Yes | integer | Must exist in `invoices.id` |
| `amount` | Yes | integer | Allocation amount |

Example request:

```json
{
  "transaction_number": "RCPT-1005",
  "receiving_account_id": 5,
  "transaction_date": "2026-02-10",
  "payment_method_id": 2,
  "paying_user_id": 77,
  "amount": 1102,
  "currency_id": 1,
  "allocations": [
    { "invoice_id": 9001, "amount": 1102 }
  ]
}
```

Example response:

```json
{
  "message": "Receipt created successfully",
  "receipt": {
    "id": 2501,
    "transaction_number": "RCPT-1005",
    "amount": 1102,
    "allocated": 1102,
    "balance": 0,
    "status": { "value": "confirmed", "color": "success" }
  }
}
```

## Update Receipt

`PUT/PATCH /api/v1/app/{company}/property-management/lease-management/receipts/{receipt}`

Update request body:

| Field | Required | Type | Allowed Values / Notes |
|---|---|---|---|
| `transaction_number` | Yes | string | - |
| `receiving_account_id` | Yes | integer | Must exist in `bank_accounts.id` |
| `transaction_date` | Yes | date (`YYYY-MM-DD`) | - |
| `payment_method_id` | Yes | integer | Must exist in `payment_methods.id` |
| `paying_user_id` | Yes | integer | Must exist in `users.id` |
| `receiving_user_id` | Yes | integer | Must exist in `users.id` |

## Cancel Receipt

`PATCH /api/v1/app/{company}/property-management/lease-management/receipts/{receipt}/cancel`

No request body required.

Possible status values returned by API resource:

- `pending` (`secondary`)
- `confirmed` (`success`)
- `cancelled` (`danger`)

## Delete Receipt

`DELETE /api/v1/app/{company}/property-management/lease-management/receipts/{receipt}`

## Frontend Error Handling

Apply shared rules in `docs/frontend/app/README.md`.
