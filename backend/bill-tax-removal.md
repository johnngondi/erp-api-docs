# Bill tax removal

A console command that takes the tax off bills that should never have carried it, such as a
supplier who is not VAT registered being billed VAT. It has no endpoint. Run it by hand on the
server, giving the bill numbers (`facility_bills.id`, the `Bill#` on statements). It reports only,
unless `--apply` is passed.

```bash
php artisan bills:remove-tax 10234 10240
php artisan bills:remove-tax 10234 10240 --apply
```

## What changes

For each bill it can take, the command writes the following, in one transaction per bill:

| Record | Change |
|---|---|
| Bill items | `tax` → `0`, `total` → `amount`, `tax_id` → the 0% tax |
| Bill | `tax` → `0`, `total` → `amount`, `balance` → `total − paid`; a `rate` / `fixed` `tax_type` becomes `none` |
| Bill withholdings | Rebuilt from the bill's `withholding_tax_ids` against the new totals, the same way editing a pending bill's tax rebuilds them. A tax with `withhold_when = when_has_tax` drops off, and a tax worked out on the amount after tax or on the tax itself is recalculated. A tax with `withhold_when = when_no_tax` that the bill's ids carry now applies, and the report lists it. A posted bill's rebuilt withholdings are `unpaid`, like the bill |
| LPO (when the bill was raised from one) | Each item: `tax` → `0`, `tax_id` → the 0% tax, `total` → its amount after discount. The LPO: `tax` → `0`, `total` → `amount_after_discount` |
| Expense (posted bill) | `tax` → `0`, `total` → `amount`, `balance` → `total − paid`. `amount` is already in the reporting currency and is unchanged |
| Vendor statement (posted bill) | The bill's posting credit → the new total |

## Bills it leaves alone

Each one is listed with the reason, and nothing is written for it:

| Reason | Why |
|---|---|
| Not found / merged | Deleted, or merged into another bill (named). Remove the tax from the bill it went into |
| Merged bill | Its tax came from the bills merged into it, so it is not this command's to change |
| Credit note / liability bill | Neither is a supplier's taxed invoice |
| No tax | Nothing to remove |
| Not pending or unpaid | Money has moved, or the bill is cancelled or reversed |
| Partly paid | `paid` is above zero |
| On a payment voucher | The voucher holds the amount it pays. Cancel the voucher, remove the tax, and raise it again |
| Remitted | Its expense is on a remittance that is not cancelled or rejected, which already reported the taxed total to the landlord |
| Withholding paid or remitted | A withholding has gone out on a liability bill |
| Customer invoice | A fee bill billed to the landlord on a customer invoice, which carries the same tax |

Running it again on a bill that has been done reports `No tax` and changes nothing.
