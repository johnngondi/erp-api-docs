# Bulk Bills Payment API

Domain: `Property Management > Finance`

Base route:

`/api/v1/app/{company}/property-management/finance/bills`

## Endpoints

- `POST /bills/bulk-settlement`

Pays several bills in one submission. The backend groups the selected bills by supplier, creates
**one payment voucher per supplier**, wraps every voucher in a **single settlement**, and releases
them all to the cashbook. Related docs: [Bills](./bill.md), [Payment Vouchers](./payment-voucher.md),
[Settlements](./settlement.md).

Permission: `create-facility-settlement`.

---

## Frontend flow

### 1. Selecting the bills

The bills table (`BillsTable`) lists payable bills from:

`GET /bills?filter[payable]=true`

That filter returns bills that are `unpaid` or `partially-paid` **and** have a non-zero
`payable_amount` (balance net of tax still withheld). The user multi-selects rows and clicks
**Pay Bills**, which opens the payment Side Drawer like we have in FacilityNodeDrawer.

### 2. Side Drawer — bill table

One row per selected bill. Every value comes straight off `FacilityBillResource`:

| Column | Primary | Secondary |
|---|---|---|
| Bill # | `id` | `invoice_number` |
| Date | `expense_posted_at.formatted` | — |
| Supplier | `vendor.name` | — |
| Property | `facility.name` | — |
| Total | `total` | `paid` |
| Balance | `balance` | `withheld_amount` |
| Payable | `payable_amount` | — |
| Amount (To Pay) | editable input, prefilled with `payable_amount` | — | see how 

The **Amount** input is what is submitted as `to_pay`. It may be reduced for a part payment; it may
not exceed the bill's payable amount.

### 3. The Side Drawer — header inputs

| Input | Sourced from | Notes |
|---|---|---|
| Debit Bank Account | `GET /finance/bank-accounts` | Only **active** accounts of `type = system`, **or** `type = user` whose `user_group_id` equals the company's `landlord_user_group_id` setting (i.e. landlord accounts). |
| Currency | `GET /settings/accounting/currencies` | Every amount submitted is expressed in this currency. |
| Payment Method | `GET /settings/accounting/payment-methods` | Carries `code` and `transaction_number_regex`. |
| Transaction Number | free text | Validated client-side, see below. |

### 4. Currency conversion

Bills may be in different currencies. **The frontend converts every bill's amount into the selected
currency before submitting** — `to_pay` is always in `currency_id`, never in the bill's own currency.
The backend converts back into each bill's currency when applying the payment, so the amount posted
against the bill is correct either way.

Changing the currency selector should re-convert and re-prefill every row's amount input.

### 5. Client-side validation

- `transaction_number` must match the selected payment method's `transaction_number_regex` when one
  is set. Apply it as a live regex check on the input.
- When `payment_method.code === 'CHEQUE'`, the number must be **digits only** — the backend
  increments it once per supplier voucher, so it has to be numeric. Required in this case.
- Each amount input: greater than `0` and not greater than that row's converted `payable_amount`.
- At least one bill must be selected.

---

## Request payload (`BulkBillsSettlementData`)

`POST /api/v1/app/{company}/property-management/finance/bills/bulk-settlement`

| Field | Required | Type | Allowed Values / Notes |
|---|---|---|---|
| `debit_bank_account_id` | Yes | integer | Active bank account; `type = system`, or `type = user` in the landlord user group |
| `payment_method_id` | Yes | integer | Must exist in `payment_methods.id` and be active |
| `currency_id` | Yes | integer | Must exist in `currencies.id`. Every `to_pay` is in this currency |
| `transaction_number` | Conditional | string | Required and numeric when the payment method's `code` is `CHEQUE`; otherwise optional, but must match `transaction_number_regex` when one is configured |
| `bills` | Yes | array | At least one entry |
| `bills[].id` | Yes | integer | Must exist in `facility_bills.id`; must be distinct across the array |
| `bills[].to_pay` | Yes | number | `> 0`, and not greater than the bill's `payable_amount` converted into `currency_id` |

Example:

```json
{
  "debit_bank_account_id": 13,
  "payment_method_id": 4,
  "currency_id": 1,
  "transaction_number": "123456",
  "bills": [
    { "id": 1201, "to_pay": 1100.00 },
    { "id": 1202, "to_pay": 450.00 },
    { "id": 1310, "to_pay": 2000.00 }
  ]
}
```

## Server-side validation

Beyond the field rules above, four cross-field checks run. All are returned as a standard `422`
validation error response.

| # | Check | Error key |
|---|---|---|
| a | Every supplier on the selected bills must have an **active bank account** — it becomes the voucher's `credit_bank_account_id`. Error names the supplier. | `bills` |
| b | Each `to_pay` must be `> 0` and within the bill's payable amount (`total − withheld − paid`) converted into `currency_id`. The bill must also still be `unpaid` / `partially-paid`. | `bills.{index}.to_pay`, `bills.{index}.id` |
| c | If the debit account is a **landlord** (`type = user`) account, every selected bill's property must belong to that landlord (`bank_account.user_id === bill.facility.landlord_id`). Error names the offending bill and property. | `debit_bank_account_id` |
| d | `transaction_number` must match the payment method's `transaction_number_regex`, and must be digits only for `CHEQUE`. | `transaction_number` |

Sample error:

```json
{
  "message": "The given data was invalid.",
  "errors": {
    "bills": ["Supplier Acme Vendor has no active bank account, so their bills cannot be paid."],
    "transaction_number": ["A cheque number must contain digits only, so it can be incremented for each supplier."]
  }
}
```

---

## What the backend does

1. Groups the selected bills by `vendor_id`, ordered by vendor id so the result is deterministic.
2. Creates one payment voucher per supplier (`payable_as = vendor`), with:
   - `credit_bank_account_id` = that supplier's first active bank account,
   - `debit_bank_account_id`, `payment_method_id`, `currency_id` from the payload,
   - one voucher item per bill, at the submitted `to_pay`.
   Each bill's `paid` is increased (converted back into the bill's own currency) and its status
   recomputed to `partially-paid` or `paid`; a linked expense is updated the same way.
3. **Cheque numbering**: the first voucher uses `transaction_number` as submitted; each subsequent
   voucher increments it by one — `123456`, `123457`, `123458`. Every other payment method reuses
   the same reference on all vouchers.
4. Adds every voucher to one settlement and **releases them**, which writes the cashbook credit
   against `debit_bank_account_id` and the supplier statement lines. Vouchers end up `paid`.
5. The settlement itself is left `pending` until a payment advice is attached via
   `PUT /finance/settlements/{settlement}/settle`.

The whole operation runs in one database transaction: if any bill fails, nothing is created.

## Response

`200 OK`

```json
{
  "data": {
    "message": "Bills paid successfully.",
    "settlement": {
      "id": 120,
      "debit_bank_account": { "id": 13, "account_name": "System Disbursement Account" },
      "status": { "value": "pending", "color": "info" },
      "created_at": { "raw": "2026-08-05T10:20:00.000000Z", "formatted": "05 Aug, 2026", "diff": "moments ago" }
    },
    "payment_vouchers": [
      {
        "id": 801,
        "payable_as": "vendor",
        "amount": "1550.00000",
        "transaction_number": "123456",
        "status": { "value": "paid", "color": "success" },
        "payable_user": { "id": 7, "name": "Acme Vendor" },
        "currency": { "id": 1, "code": "KES" },
        "payment_method": { "id": 4, "name": "Cheque", "code": "CHEQUE" },
        "items": [
          { "id": 9001, "amount": "1100.00000" },
          { "id": 9002, "amount": "450.00000" }
        ]
      },
      {
        "id": 802,
        "payable_as": "vendor",
        "amount": "2000.00000",
        "transaction_number": "123457",
        "status": { "value": "paid", "color": "success" },
        "payable_user": { "id": 9, "name": "Beta Supplies" },
        "items": [{ "id": 9003, "amount": "2000.00000" }]
      }
    ]
  }
}
```

Display the settlement and the vouchers as a result summary, each opening its own detail screen:

- Settlement — `GET /finance/settlements/{settlement}` - See the implementaion of route leading to @C:\Users\user\dev\apps\propx\core\src\pages\app\property-management\supplier-settlements\ViewSupplierSettlementPage.vue
- Voucher — `GET /finance/payment-vouchers/{paymentVoucher}` - See implementation of route leading to @C:\Users\user\dev\apps\propx\core\src\pages\app\property-management\supplier-payment-vouchers\ViewSupplierPaymentVoucherPage.vue


