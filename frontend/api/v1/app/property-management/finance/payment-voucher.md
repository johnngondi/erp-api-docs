# Payment Vouchers API

Domain: `Property Management > Finance`

Base route:

`/api/v1/app/{company}/property-management/finance/payment-vouchers`

## Endpoints

- `GET /payment-vouchers`
- `GET /payment-vouchers/create` (options preview; `payment_voucher_id` adds a voucher's own payables for its edit form)
- `POST /payment-vouchers`
- `GET /payment-vouchers/{paymentVoucher}`
- `PUT /payment-vouchers/{paymentVoucher}` — edit while `pending` (see [Edit a payment voucher](#edit-a-payment-voucher))
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

**Ids, `status` and `type` match exactly.** They were partial until 2026-09-29, so
`filter[status]=paid` also returned `unpaid` and `partially-paid`, and `filter[facility_id]=6`
returned rows for facility 60. Send the whole value. Date filters are still partial, so
`filter[created_at]=2026-09` means that month.
Every date filter also takes a range, `filter[created_between]=2026-09-01,2026-09-30`; see
[List filters](../../../list-filters.md) for each one's name.
- Sort:
  - `sort=id,amount,paid_at,released_at,created_at,updated_at`
- Include:
  - `include=paymentVoucherItems`
- Pagination:
  - `per_page`, `page`

Enum filter options:

- `filter[payable_as]`: `vendor`, `landlord`, `tenant`, `other` (from `FacilityPaymentVoucherData`)
- `filter[status]`: `pending`, `released`, `paid`, `cancelled` (from `PaymentVoucherStatus` enum)

Sample list response (`FacilityPaymentVoucherResource`). `public_url` is the voucher's public payment advice page; see [Public Payment Voucher Page](#public-payment-voucher-page).

```json
{
  "data": [
    {
      "id": 801,
      "public_url": "https://api.example.com/payment-vouchers/Yx3...40 random chars",
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

## Edit a payment voucher

`PUT /api/v1/app/{company}/property-management/finance/payment-vouchers/{paymentVoucher}`

Only while the voucher is `pending` — before it is released (row 138). Needs
`update-facility-payment-voucher`, or to be the voucher's creator.

Body: **the same payload as create**, sent whole. The items sent replace the voucher's items:

- Creating a voucher already marks what it pays: a bill's `paid` goes up by the item's amount, a
  remittance becomes `paid`. The edit takes that back for the old items (the bill's `paid` goes down
  again, a remittance goes back to `unpaid`) and applies the new items the same way create does.
- `amount` is checked against the bill's balance **plus what this voucher already pays on that bill**, so
  sending the voucher back unchanged, or raising an item's amount up to the bill's full balance, passes.
- The status stays `pending`. Nothing reaches the vendor statement until the voucher is released.

Response `200`: `{ "data": { "message": "Payment voucher updated successfully.", "payment_voucher": { ... } } }`

Errors:

- `403` — `"Only a pending payment voucher can be edited."`, or no edit right.
- `422` — validation, as on create.

Building the edit form: call the options endpoint with the voucher's id,
`GET /payment-vouchers/create?payable_as=…&payable_user_id=…&payment_voucher_id={id}`. Alongside what
is still open it returns the bills or remittances this voucher already pays - even ones it pays in
full - in the same shape, and every option carries `voucher_amount`:

- `voucher_amount`: what this voucher pays on it, in the voucher's currency; `null` when it is not on
  this voucher. Tick the options where it is set and prefill their amount with it.
- Bills: `payable_amount` includes what this voucher pays on the bill (remaining balance + this
  voucher's payment, in the bill's currency), so it is the most the edited voucher can pay on it.
  `paid` and `balance` are the bill's own figures, with this voucher's payment already in them.
- Remittances: `remittable_amount` as on create; the amount must still equal it.

The voucher's own items are offered only while the query names the voucher's own payee
(`payable_as` and `payable_user_id`). `payment_voucher_id` needs the edit right on that voucher
(`403` otherwise). Without it the endpoint behaves as before.

## Flags, printing and delete

- `permissions.update` is `true` only on a `pending` voucher the user may edit.
- `permissions.delete` is `true` only on a `pending` or `cancelled` voucher, matching what
  `DELETE` accepts.
- A `pending` or `cancelled` voucher cannot be printed: the template render endpoints refuse it with
  `422` on `resource_id` (`"... is pending and cannot be printed."`), the same rule as invoices, credit
  notes, receipts and LPOs. Hide the print action unless the status is `paid`. The payee's public page
  already hid pending vouchers.

## Pay payload (`PaymentAdviceData`)

| Field | Required | Type |
|---|---|---|
| `payment_advice_number` | Yes | string |
| `payment_advice_upload_id` | Yes | integer (`uploads.id`) |

## Public Payment Voucher Page

Every payment voucher has a public link that the payee can open without signing in, for example
from the email or SMS that tells them they have been paid. Voucher responses carry it as
`public_url`:

```json
{
  "id": 801,
  "public_url": "https://api.example.com/payment-vouchers/Yx3...40 random chars"
}
```

The link ends in a random 40-character token, not the voucher id, so voucher numbers cannot be
guessed. The token is issued when the voucher is created; vouchers that existed before this
feature were given one by the migration.

These are web pages served by the backend, not JSON endpoints:

| Route | Returns |
|---|---|
| `GET /payment-vouchers/{token}` | HTML page. The left column shows the payment advice: the voucher number, status, voucher date, paid date, payment method, reference, credited account and currency; the payee ("Paid to") and the company ("Paid by"); the items paid (date, property, description and memo, total, one column per withholding tax deducted on the voucher, withheld, advance, payable and paid) with a Total paid footer; who prepared it; and the voucher notes. The right column shows the amount paid and a **Download payment advice** button |
| `GET /payment-vouchers/{token}/pdf` | The voucher as a PDF (`inline`, `PV0801.pdf`), printed with the template chosen in the `payment_voucher_advice_template_id` setting (see [Settings](../settings.md)) |

Rules:

- An unknown token returns `404`. So does a `pending` voucher, since it has not been paid yet.
- A `cancelled` voucher is still shown, with its Cancelled badge.
- **Download** is hidden, and `/pdf` returns `404`, when the company has no usable payment
  voucher template.
- The PDF template is the one named by the `payment_voucher_advice_template_id` setting when it
  is still active, else the company's default payment voucher template, else its newest active
  one. A voucher has no property, so only company-wide templates (no `facility_ids`) apply.
- Both routes are limited to 60 requests a minute per IP, and are marked `noindex`.
