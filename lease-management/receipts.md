# Receipts API

Base route:
`/api/v1/crm/lease-management/receipts`

## Endpoints

### List receipts
`GET /api/v1/crm/lease-management/receipts`

Query parameters (Spatie Query Builder v6):
- `filter[id]` (exact)
- `filter[receiving_account_id]` (exact)
- `filter[transaction_number]` (exact)
- `filter[transaction_date]` (exact)
- `filter[payment_method_id]` (exact)
- `filter[paying_user_id]` (exact)
- `filter[amount]` (exact)
- `filter[allocated]` (exact)
- `filter[balance]` (exact)
- `filter[receiving_user_id]` (exact)
- `sort` (comma-separated): `id`, `receiving_account_id`, `transaction_number`, `transaction_date`, `payment_method_id`, `paying_user_id`, `amount`, `allocated`, `balance`, `receiving_user_id` (prefix with `-` for desc)
- `per_page` (int, optional; defaults to `config('app.query.default_per_page')`)
- `page` (int, optional)

Notes:
- Relationships are eagerly loaded by default: `currency`, `receiptAllocations` (with `invoice`), `receivingAccount` (with `bankBranch.bank`), `paymentMethod`, `payingUser`, `receivingUser`.
- `include` and `fields` parameters are not enabled for this endpoint.

Example:
`GET /api/v1/crm/lease-management/receipts?filter[transaction_number]=RCPT-001&sort=-transaction_date`

### Create a receipt
`POST /api/v1/crm/lease-management/receipts`

Request body:
- `transaction_number` (string, required)
- `receiving_account_id` (int, required; must exist in `bank_accounts.id`)
- `transaction_date` (date, required)
- `payment_method_id` (int, required; must exist in `payment_methods.id`)
- `paying_user_id` (int, required; must exist in `users.id`)
- `amount` (int, required)
- `currency_id` (int, optional; must exist in `currencies.id`)
- `allocations` (array, required)
  - `invoice_id` (int, required; must exist in `invoices.id`)
  - `amount` (int, required)

Response:
```json
{
  "message": "Receipt created successfully",
  "receipt": {
    "id": 1,
    "transaction_number": "RCPT-001",
    "transaction_date": {
      "raw": "2024-06-01T10:00:00.000000Z",
      "formatted": "01 Jun, 2024",
      "diff": "2 days ago"
    },
    "payment_method": {
      "id": 2,
      "name": "Bank Transfer"
    },
    "paying_user": {
      "id": 77,
      "name": "John Doe"
    },
    "amount": 1000,
    "allocated": 1000,
    "balance": 0,
    "status": "paid",
    "receipt_allocations": [
      {
        "id": 55,
        "invoice": {
          "id": 9001,
          "amount": 1050
        },
        "amount": 1000
      }
    ],
    "created_at": {
      "raw": "2024-06-01T10:00:00.000000Z",
      "formatted": "01 Jun, 2024",
      "diff": "2 days ago"
    }
  }
}
```

### Show a receipt
`GET /api/v1/crm/lease-management/receipts/{receipt}`

### Update a receipt
`PUT /api/v1/crm/lease-management/receipts/{receipt}`

Request body:
- `transaction_number` (string, required)
- `receiving_account_id` (int, required; must exist in `bank_accounts.id`)
- `transaction_date` (date, required)
- `payment_method_id` (int, required; must exist in `payment_methods.id`)
- `paying_user_id` (int, required; must exist in `users.id`)
- `receiving_user_id` (int, required; must exist in `users.id`)

Response:
```json
{
  "message": "Receipt updated successfully",
  "receipt": {
    "id": 1,
    "transaction_number": "RCPT-001"
  }
}
```

### Cancel a receipt
`PATCH /api/v1/crm/lease-management/receipts/{receipt}/cancel`

Notes:
- If the receipt is already cancelled, the API returns a validation error.
- Cancelling updates receipt allocations and related invoice figures.

Response:
```json
{
  "message": "Receipt cancelled successfully",
  "receipt": {
    "id": 1,
    "status": "cancelled"
  }
}
```

### Delete a receipt
`DELETE /api/v1/crm/lease-management/receipts/{receipt}`

Response:
```json
{
  "message": "Receipt deleted successfully",
  "receipt": {
    "id": 1
  }
}
```
