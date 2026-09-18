# Bill Payment Report API

Domain: `Property Management > Reports > Suppliers > Expenditure Reports`

Base route:

`/api/v1/app/{company}/property-management/reports`

The Bill Payment report lists **every payment made against a supplier bill in the period**. Each row is
one payment, with the bill it paid and that bill's current position. It is **read-only** and computed
on the fly.

> **Read [the shared reports contract](./README.md) first.** This page documents what is specific to
> this report. The response envelope, the bucket keys, the field keys, the per-cell `{ value, color }`
> shape and the colour vocabulary live in the README and apply here unchanged.

## Endpoints

- `GET /reports/suppliers/expenditure-reports/bill-payment` — requires `view-bill-payment-report`
- `GET /reports/suppliers/expenditure-reports/bill-payment/export` — requires
  `export-bill-payment-report`. Takes a required `format` (`excel` | `pdf`) plus the same filters as the
  report; any other format, or none, returns `422` with a `format` error.
- Listed by [`GET /reports`](./README.md#discovering-reports--get-reports) under the supplier
  `expenditure-reports` group.

The view and export permissions are **independent**, and neither is shared with
[Bill Submission](./bill-submission.md).

## Rows are payments, not bills

A payment is **one line on a payment voucher** (`facility_payment_voucher_items`) whose `payable` is a
Facility Bill. **A bill paid on two vouchers appears on two rows.** The bill's own columns (and `paid`,
`balance`, `withheld`, `payable`) describe the bill, so they **repeat on each of its rows** — never add
them up down the column. Only `payment_amount` belongs to the row.

- A payment falls in the period by its **payment date**: the voucher's `paid_at`, or when the voucher
  was raised if it has not been released yet.
- **Cancelled vouchers are excluded** (cancelling a voucher reverses what it paid on the bill).
  Pending (unreleased) vouchers are included: the bill's `paid` already counts them.
- Voucher lines paying anything other than a bill (e.g. a remittance) never appear.
- Settlements (`FacilityPaymentVoucherSettlement`) are **not** read.

## Where every amount comes from

| Column | `key` | Source | Currency |
|---|---|---|---|
| Amount | `amount` | `facility_bills.amount`, as stored | bill |
| Tax | `tax` | `facility_bills.tax`, as stored | bill |
| Total | `total` | `facility_bills.total`, as stored | bill |
| Withheld | `withheld` | **sum of the bill's `facility_bill_withholdings.amount` lines** (every line, settled or not); `0` when it has none | bill |
| Payable | `payable` | **`total − withheld`**: what the supplier is owed before any payment | bill |
| Paid | `paid` | `facility_bills.paid`, as stored — **everything paid on the bill so far**, not just this row's payment | bill |
| Balance | `balance` | `facility_bills.balance`, as stored | bill |
| Payment Amount | `payment_amount` | `facility_payment_voucher_items.amount` — **this payment** | voucher (`payment_currency`) |

`paid` and `balance` are read as stored and are **today's** figures for the bill: a bill paid 400 in
April and 300 in May shows `paid: 700` on both rows, even with a period that ends in April.

## Filters

The six [global filters](./README.md#global-filters) plus:

| Param | Type | Meaning when omitted |
|---|---|---|
| six global filters | see the shared contract | all / current month |
| `payment_method_id` | `payment_methods.id` (the voucher's) | All payment methods |
| `supplier_id` | `users.id` (the bill's `vendor_id`) | All suppliers |
| `debit_bank_account_id` | `bank_accounts.id` (the voucher's) | All bank accounts |

`header.filters` echoes all nine.

```
GET …/reports/suppliers/expenditure-reports/bill-payment?period_from=2026-04-01&period_to=2026-04-30&debit_bank_account_id=3
```

## Header

- `property` — `{ id, name }` when a `facility_id` is given, otherwise `null`.
- `currency` — `{ code, name }` when every bill **and** every voucher listed is in one currency; `null`
  otherwise.

## Bucket

One bucket, `overview` (`header.label`: `Bill Payments`): one row per payment, oldest payment first,
then a closing `Total Paid` row.

## Columns

The first nine columns are **shared with [Bill Submission](./bill-submission.md)**, with identical
definitions: `invoice_no`, `expense_type`, `supplier`, `property`, `billed_date`, `amount`, `tax`,
`total`, `status` (the bill's own `facility_bills.status`, with its status colour). Then:

| Column | `key` | `format` | Notes |
|---|---|---|---|
| Withheld | `withheld` | money | see the source table |
| Payable | `payable` | money | see the source table |
| Paid | `paid` | money | see the source table |
| Balance | `balance` | money | see the source table |
| Payment Date | `payment_date` | string `Y-m-d` | the voucher's `paid_at`, or its creation date while unreleased |
| Payment Method | `payment_method` | string | the voucher's payment method name |
| Debit Account | `debit_account` | string | the voucher's debit bank account (`account name — number`); `togglable` |
| Reference | `payment_reference` | string | the voucher's `transaction_number`; `togglable` |
| Payment Amount | `payment_amount` | money, `grosstotal` | this payment |
| Payment Currency | `payment_currency` | string | the voucher currency code; `togglable` |
| Payment Status | `payment_status` | string | the voucher status (`pending` secondary, `paid` success) |

The closing row is a `Total Paid` label with `col_span: 5`, then **only** `payment_amount` (the sum of
the payments). The bill columns are deliberately not summed, because a bill repeats once per payment.

## Overdue tint

A row gets **`background_color: warning`** when **both** hold:

- more than **90 days** have passed since the bill's `billed_date` (counted to today), and
- the bill is still not fully paid: its status is not `paid` or `cancelled`.

## Summary

| Key | Meaning |
|---|---|
| `total_payments` | Sum of `payment_amount` (in voucher currencies) |
| `payment_count` | Rows listed |
| `bill_count` | Distinct bills paid |
| `overdue_unpaid_count` | Rows tinted warning |

## Example response

One bill (total 1,160, withholding lines of 30 and 20) paid on two vouchers:

```jsonc
{
  "data": {
    "header": {
      "property": null,
      "currency": { "code": "KES", "name": "Kenya Shilling" },
      "period": { "from": "2026-04-01", "to": "2026-06-30" },
      "filters": { "landlord_id": null, "facility_id": null, "facility_type_id": null, "properties_status": null,
                   "period_from": "2026-04-01", "period_to": "2026-06-30",
                   "payment_method_id": null, "supplier_id": null, "debit_bank_account_id": null },
      "generated_at": "2026-06-30T12:00:00+03:00"
    },
    "report": [
      {
        "bucket": "overview",
        "header": { "label": "Bill Payments" },
        "fields": [ /* the nine shared columns, then the eleven above, in order */ ],
        "items": [
          { "invoice_no": { "value": "INV-0142" }, "expense_type": { "value": "Plumbing" }, "supplier": { "value": "AquaFix Ltd" },
            "property": { "value": "ACK Gardens" }, "billed_date": { "value": "2026-01-28" },
            "amount": { "value": 1000 }, "tax": { "value": 160 }, "total": { "value": 1160 },
            "status": { "value": "partially-paid", "color": "info" },
            "withheld": { "value": 50 }, "payable": { "value": 1110 }, "paid": { "value": 700 }, "balance": { "value": 460 },
            "payment_date": { "value": "2026-04-10" }, "payment_method": { "value": "Bank Transfer" },
            "debit_account": { "value": "Operations — 0102030405" }, "payment_reference": { "value": "TX-1" },
            "payment_amount": { "value": 400 }, "payment_currency": { "value": "KES" },
            "payment_status": { "value": "paid", "color": "success" },
            "type": "normal", "background_color": "warning" },
          { "invoice_no": { "value": "INV-0142" }, /* …same bill columns… */
            "withheld": { "value": 50 }, "payable": { "value": 1110 }, "paid": { "value": 700 }, "balance": { "value": 460 },
            "payment_date": { "value": "2026-05-10" }, "payment_method": { "value": "Bank Transfer" },
            "debit_account": { "value": "Operations — 0102030405" }, "payment_reference": { "value": "TX-2" },
            "payment_amount": { "value": 300 }, "payment_currency": { "value": "KES" },
            "payment_status": { "value": "paid", "color": "success" },
            "type": "normal", "background_color": "warning" },
          { "invoice_no": { "value": "Total Paid", "col_span": 5 }, "payment_amount": { "value": 700 },
            "type": "grosstotal", "background_color": "secondary" }
        ],
        "summary": { "total_payments": 700, "payment_count": 2, "bill_count": 1, "overdue_unpaid_count": 2 }
      }
    ],
    "summary": { "total_payments": 700, "payment_count": 2, "bill_count": 1, "overdue_unpaid_count": 2 }
  }
}
```
