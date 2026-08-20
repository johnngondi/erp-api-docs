# Bulk Bills Payment API

Domain: `Property Management > Finance`

Base route:

`/api/v1/app/{company}/property-management/finance/bills`

## Endpoints

- `POST /bills/bulk-settlement`

Pays several bills in one submission. The user splits the selected bills into groups: each **numbered
group** holds one supplier's bills and becomes **one payment voucher** credited to the account the
group names, while the optional trailing **`others`** group holds everything left ungrouped and is
split per supplier against each supplier's own default account. Every voucher is wrapped in a
**single settlement** and released to the cashbook. Related docs: [Bills](./bill.md),
[Payment Vouchers](./payment-voucher.md), [Settlements](./settlement.md).

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
  increments it once per voucher, so it has to be numeric. Required in this case.
- Each amount input: greater than `0` and not greater than that row's converted `payable_amount`.
- At least one bill must be selected.
- Rows with `to_pay <= 0` are stripped before submitting, and a group left empty is omitted entirely.
- Every numbered group must hold a single supplier's bills and must name a credit bank account
  belonging to that supplier.

### 6. Grouping the rows

The user can carve the selected rows into their own groups:

- Groups are numbered from `1` upwards, contiguously — re-index them at submit time.
- A numbered group carries `credit_bank_account_id`: the supplier account **that voucher** is paid
  to, instead of the supplier's default. Source the options from `GET /finance/bank-accounts`
  filtered to active accounts whose `user_id` is that supplier — including `type = system` ones,
  since a company that supplies itself is paid into its own system account.
- Anything ungrouped goes into a single `others` group with `credit_bank_account_id: null`. It may
  span suppliers, and it is always the **last** entry in `bill_groups`.

---

## Request payload (`BulkBillsSettlementData`)

`POST /api/v1/app/{company}/property-management/finance/bills/bulk-settlement`

| Field | Required | Type | Allowed Values / Notes |
|---|---|---|---|
| `debit_bank_account_id` | Yes | integer | Active bank account; `type = system`, or `type = user` in the landlord user group |
| `payment_method_id` | Yes | integer | Must exist in `payment_methods.id` and be active |
| `currency_id` | Yes | integer | Must exist in `currencies.id`. Every `to_pay` is in this currency |
| `transaction_number` | Conditional | string | Required and numeric when the payment method's `code` is `CHEQUE`; otherwise optional, but must match `transaction_number_regex` when one is configured |
| `bill_groups` | Yes | array | At least one entry |
| `bill_groups[].group` | Yes | integer \| string | A 1-based group number, or the literal `"others"`. At most one `"others"`, and it must be last |
| `bill_groups[].credit_bank_account_id` | Conditional | integer \| null | Required on a numbered group: any **active** account whose `user_id` is that group's supplier (a company that supplies itself may be paid into its own `type = system` account). Must be `null` on the `"others"` group |
| `bill_groups[].bills` | Yes | array | At least one entry |
| `bill_groups[].bills[].id` | Yes | integer | Must exist in `facility_bills.id`; must be distinct **across all groups** |
| `bill_groups[].bills[].to_pay` | Yes | number | `> 0`, and not greater than the bill's `payable_amount` converted into `currency_id` |

Example:

```json
{
  "debit_bank_account_id": 13,
  "payment_method_id": 4,
  "currency_id": 1,
  "transaction_number": "123456",
  "bill_groups": [
    {
      "group": 1,
      "credit_bank_account_id": 12,
      "bills": [
        { "id": 1201, "to_pay": 1100.00 },
        { "id": 1202, "to_pay": 450.00 }
      ]
    },
    {
      "group": "others",
      "credit_bank_account_id": null,
      "bills": [{ "id": 1310, "to_pay": 2000.00 }]
    }
  ]
}
```

### Legacy flat payload

The pre-grouping shape — a flat `bills` array in place of `bill_groups`, with the same
`{ id, to_pay }` entries — is still accepted and behaves exactly like a single `others` group. Its
errors keep their flat `bills`, `bills.{index}.id` and `bills.{index}.to_pay` keys. New clients
should send `bill_groups`.

## Server-side validation

Beyond the field rules above, several cross-field checks run. All are returned as a standard `422`
validation error response. `{g}` is the index of the group in `bill_groups`, `{i}` the index of the
row within that group — map them straight back onto the offending input.

| # | Check | Error key |
|---|---|---|
| a | Groups are numbered `1..n`, each number appears once, and at most one `"others"` group is present and last. | `bill_groups.{g}.group` |
| b | A numbered group may only hold **one supplier's** bills. | `bill_groups.{g}.bills` |
| c | A numbered group must name a `credit_bank_account_id` that is active and belongs to that group's supplier (`bank_account.user_id === bill.vendor_id`; any `type`). The `"others"` group must not name one. | `bill_groups.{g}.credit_bank_account_id` |
| d | Every supplier paid through the `"others"` group must have an **active bank account** — it becomes that voucher's `credit_bank_account_id`. Error names the supplier. | `bill_groups.{g}.bills` |
| e | Each `to_pay` must be `> 0` and within the bill's payable amount (`total − withheld − paid`) converted into `currency_id`, with **half a cent of tolerance** (see below). The bill must also still be `unpaid` / `partially-paid`. | `bill_groups.{g}.bills.{i}.to_pay`, `bill_groups.{g}.bills.{i}.id` |
| f | If the debit account is a **landlord** (`type = user`) account, every selected bill's property must belong to that landlord (`bank_account.user_id === bill.facility.landlord_id`). Error names the offending bill and property. | `debit_bank_account_id` |
| g | `transaction_number` must match the payment method's `transaction_number_regex`, and must be digits only for `CHEQUE`. | `transaction_number` |

### Sub-cent payable amounts

Money is stored to five decimals, so a payable amount net of a withholding can land on a fraction of
a cent — `351,678.01 − 6,063.414 = 345,614.596` — while the drawer only shows and submits whole
cents. Paying such a bill off in full therefore arrives a rounding step over the payable amount, so
the check above allows **half a cent** of overshoot, and the backend trims what it posts back to the
exact payable amount. The bill still lands on a zero balance; the voucher total can be a fraction of
a cent below the sum of the submitted rows.

Sample error:

```json
{
  "message": "The given data was invalid.",
  "errors": {
    "bill_groups.0.credit_bank_account_id": ["The selected bank account must be an active account belonging to the supplier being paid."],
    "bill_groups.1.bills.0.to_pay": ["The amount to pay for bill #1310 may not be greater than its payable amount (1,800.00)."],
    "transaction_number": ["A cheque number must contain digits only, so it can be incremented for each supplier."]
  }
}
```

---

## What the backend does

1. Walks the groups in a deterministic order: numbered groups by ascending group number, then the
   `others` group split by `vendor_id` ascending.
2. Creates one payment voucher per numbered group, and one per supplier inside `others`
   (`payable_as = vendor`), with:
   - `credit_bank_account_id` = the group's `credit_bank_account_id`, or — inside `others` — that
     supplier's first active bank account,
   - `debit_bank_account_id`, `payment_method_id`, `currency_id` from the payload,
   - one voucher item per bill, at the submitted `to_pay`.
   Each bill's `paid` is increased (converted back into the bill's own currency) and its status
   recomputed to `partially-paid` or `paid`; a linked expense is updated the same way.
3. **Cheque numbering**: the first voucher uses `transaction_number` as submitted; each subsequent
   voucher increments it by one — `123456`, `123457`, `123458` — in the order above. Note this is one
   number per **voucher**, so an `others` group spanning three suppliers consumes three. Every other
   payment method reuses the same reference on all vouchers.
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


