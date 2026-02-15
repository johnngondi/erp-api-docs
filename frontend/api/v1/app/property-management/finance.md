# Finance API

Domain: `Property Management`

Base prefix:

`/api/v1/app/{company}/property-management/finance`

## Bills

Endpoints:

- `GET /bills`
- `POST /bills`
- `GET /bills/{bill}`
- `PUT/PATCH /bills/{bill}`
- `DELETE /bills/{bill}`
- `PATCH /bills/{bill}/cancel`
- `PUT /bills/{bill}/post-bill`
- `PUT /bills/{bill}/reject-invoice`

List query support:

- Filters:
  - `filter[id]`, `filter[invoice_number]`, `filter[tax_invoice_number]`, `filter[type]`, `filter[status]`
  - `filter[facility_id]`, `filter[expense_type_id]`, `filter[vendor_id]`
- Sort:
  - `sort=id,invoice_number,invoice_date`
- Include:
  - `include=items`

Create payload (`BillData`):

| Field | Required | Type | Allowed Values / Notes |
|---|---|---|---|
| `vendor_id` | Yes | integer | Must exist in `users.id` |
| `facility_id` | Yes | integer | Must exist in `facilities.id` |
| `expense_type_id` | Yes | integer | Must exist in `expense_types.id` |
| `type` | No | string | `lpo`, `contract`, `liability`, `other` |
| `items` | No | array | Bill item objects |
| `currency_id` | No | integer | Must exist in `currencies.id` |
| `billable_type` | No | string | Optional |
| `billable_id` | No | integer | Optional |
| `notes` | No | string | Optional |
| `invoice_number` | No | string | Optional |
| `invoice_date` | No | date | Optional |
| `invoice_upload_id` | No | integer | Must exist in `uploads.id` |
| `tax_invoice_number` | No | string | Optional |

Bill item object:

| Field | Required | Type | Allowed Values / Notes |
|---|---|---|---|
| `tax_id` | No | integer | Must exist in `taxes.id` |
| `quantity` | No | integer | Defaults to `1` |
| `cost` | No | number | Defaults to `0` |
| `title` | No | string | Optional |
| `notes` | No | string | Optional |

Upload/post payload (`UploadInvoiceBillData`, used by `post-bill`, `reject-invoice`, and some updates):

| Field | Required | Type |
|---|---|---|
| `invoice_number` | Yes | string |
| `invoice_date` | Yes | date |
| `tax_invoice_number` | Yes | string |
| `invoice_upload_id` | No | integer (`uploads.id`) |
| `notes` | No | string |

## Expenses

Endpoints:

- `GET /expenses`
- `POST /expenses`
- `GET /expenses/{expense}`
- `PUT/PATCH /expenses/{expense}`
- `DELETE /expenses/{expense}`
- `PATCH /expenses/{expense}/mark-paid`
- `PATCH /expenses/{expense}/cancel`

List query support:

- Filters:
  - `filter[id]`, `filter[invoice_number]`, `filter[notes]`, `filter[facility_id]`, `filter[vendor_id]`
  - `filter[created_at]`, `filter[transaction_at]`
- Sort:
  - `sort=id,transaction_at,amount,created_at`
- Include:
  - `include=taxType,invoiceUpload,creatingUser,bill`

Create/Update payload (`ExpenseData`):

| Field | Required | Type | Allowed Values / Notes |
|---|---|---|---|
| `vendor_id` | Yes | integer | Must exist in `users.id` |
| `facility_id` | Yes | integer | Must exist in `facilities.id` |
| `transaction_at` | Yes | date | - |
| `expense_type_id` | Yes | integer | Must exist in `expense_types.id` |
| `invoice_number` | Yes | string | - |
| `amount` | Yes | number | - |
| `status` | No | string | `pending`, `unpaid`, `partially-paid`, `paid`, `cancelled` |
| `creating_user_id` | No | integer | Must exist in `users.id` |
| `notes` | No | string | Optional |
| `bill_id` | No | integer | Must exist in `bills.id` |
| `invoice_upload_id` | No | integer | Must exist in `uploads.id` |
| `tax_invoice_number` | No | string | Optional |
| `currency_id` | No | integer | Must exist in `currencies.id` |
| `tax_id` | No | integer | Must exist in `taxes.id` |
| `paid` | No | number | Default `0` |

## Remittances

Endpoints:

- `GET /remittances`
- `GET /remittances/create` (preview options/calculation before create)
- `POST /remittances`
- `GET /remittances/{remittance}`
- `DELETE /remittances/{remittance}`
- `PATCH /remittances/{remittance}/approve`
- `PATCH /remittances/{remittance}/cancel`

Notes:

- `PUT/PATCH /remittances/{remittance}` route exists but is currently not implemented in controller.

List query support:

- Filters:
  - `filter[id]`, `filter[status]`, `filter[facility_id]`, `filter[landlord_id]`, `filter[period_from]`, `filter[period_to]`
- Sort:
  - `sort=id,period_from,period_to`
- Include:
  - `include=collections,expenses`

Create/preview payload:

| Field | Required | Type |
|---|---|---|
| `facility_id` | Yes | integer (`facilities.id`) |
| `period_from` | Yes | date |
| `period_to` | Yes | date |

## Payment Vouchers

Endpoints:

- `GET /payment-vouchers`
- `GET /payment-vouchers/create` (options preview)
- `POST /payment-vouchers`
- `GET /payment-vouchers/{paymentVoucher}`
- `DELETE /payment-vouchers/{paymentVoucher}`
- `PATCH /payment-vouchers/{paymentVoucher}/cancel`
- `PATCH /payment-vouchers/{paymentVoucher}/release`
- `PUT /payment-vouchers/{paymentVoucher}/pay`

List query support:

- Filters:
  - `filter[id]`, `filter[status]`, `filter[payable_as]`, `filter[payable_user_id]`, `filter[created_at]`
- Sort:
  - `sort=id,amount,paid_at,released_at,created_at`
- Include:
  - `include=paymentVoucherItems`

Create payload (`PaymentVoucherData`):

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

Pay payload (`PaymentAdviceData`):

| Field | Required | Type |
|---|---|---|
| `payment_advice_number` | Yes | string |
| `payment_advice_upload_id` | Yes | integer (`uploads.id`) |

## Settlements

Endpoints:

- `GET /settlements`
- `POST /settlements`
- `GET /settlements/{settlement}`
- `PUT/PATCH /settlements/{settlement}`
- `DELETE /settlements/{settlement}`
- `PATCH /settlements/{settlement}/cancel`
- `PUT /settlements/{settlement}/settle`

List query support:

- Filters:
  - `filter[id]`, `filter[status]`, `filter[debit_bank_account_id]`, `filter[created_at]`
- Sort:
  - `sort=id,created_at`
- Include:
  - `include=paymentVouchers`

Create payload (`SettlementData`):

| Field | Required | Type | Allowed Values / Notes |
|---|---|---|---|
| `debit_bank_account_id` | Yes | integer | Must be an active `system` bank account |
| `payment_vouchers` | Yes | array[int] | Vouchers must be `released` and unused in pending/settled settlement |
| `payment_advice_number` | No | string | Optional |
| `payment_advice_upload_id` | No | integer | Must exist in `uploads.id` |

Settle payload:

| Field | Required | Type |
|---|---|---|
| `payment_advice_number` | Yes | string |
| `payment_advice_upload_id` | Yes | integer (`uploads.id`) |

## Common enum values returned in responses

- Bill status: `pending`, `unpaid`, `partially-paid`, `paid`, `cancelled`
- Expense status: `pending`, `unpaid`, `partially-paid`, `paid`, `cancelled`
- Remittance status: `pending`, `unpaid`, `paid`, `cancelled`
- Payment voucher status: `pending`, `released`, `paid`, `cancelled`
- Settlement status: `pending`, `settled`, `cancelled`

## Errors

Use shared behavior in `docs/frontend/app/README.md`:

- Render `4xx` errors/messages directly.
- Show generic fallback for `5xx`.
