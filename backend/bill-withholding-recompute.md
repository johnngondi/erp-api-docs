# Bill withholding recompute

A console command that works a bill's withholdings out again under the current rules and rewrites
the ones that come out different. It is for bills raised before a change to how withholdings are
calculated. The case it was built for: a withholding on `only_taxable_amount` (VAT withholding) is
now charged on the amount the tax was levied on, worked back from the tax (each taxed line's tax ÷
its rate). Bills created before that change still carry the old figure.

It has no endpoint. Run it by hand on the server. Give it bill numbers (`facility_bills.id`, the
`Bill#` on statements), or pass `--all` to check every bill it can take. It only reports unless
`--apply` is passed.

```bash
php artisan bills:recompute-withholdings 10234 10240
php artisan bills:recompute-withholdings 10234 10240 --apply
php artisan bills:recompute-withholdings --all
php artisan bills:recompute-withholdings --all --apply
```

Give either bill numbers or `--all`, not both.

The report lists each bill with its date, supplier, property, status, and withholdings before and
after. The date is the supplier's invoice date. A pending bill often has none yet, so it shows the
day the bill was raised, marked `(created)`.

## What changes

For each bill it can take, in one transaction per bill:

| Record | Change |
|---|---|
| Bill withholdings | Rebuilt from the bill's `withholding_tax_ids` against its stored `amount`, `tax` and `total`, with `SyncBillWithholdingsAction`: the same code that creating or editing a bill runs. `withhold_when`, `minimum_amount` and `cap_amount` apply as they do there. A posted bill's rebuilt withholdings are `unpaid`, as posting leaves them |

Nothing else is written. The bill's own totals, `paid`, `balance` and status, its expense, its
vendor statement and any LPO stay as they are, because a withholding only changes what the supplier
is paid.

## Bills with money paid

A posted bill is recomputed whether or not the supplier has been paid on it, as long as the bill's
total withheld does not go up. A payment voucher pays the supplier the balance less the
withholdings still outstanding, so a part-paid bill's `balance` is usually the amount withheld.

- **Withholding goes down.** The rows are rewritten. The bill's balance stays, so the difference is
  now payable to the supplier and shows on the bill's payable amount, ready for the next voucher.
- **Withholding goes up.** The bill is skipped: the supplier has already been paid more than the new
  withholding would leave them.

A bill with nothing paid is recomputed either way.

The rebuild covers every withholding on the bill, not only VAT withholding. A withholding tax whose
rate, cap or minimum was edited after the bill was raised comes out at the new figure as well. The
report lists each withholding before and after, so that change is visible.

A bill whose withholdings come out the same, to the cent, is left untouched even with `--apply`:
its rows keep their ids. The report shows it as `unchanged`.

## Bills it leaves alone

With bill numbers, each one is listed with the reason, and nothing is written for it. With `--all`,
the command only reads bills that are pending, unpaid, partially paid or paid, not deleted, and not
credit notes or liability bills. It lists only the bills that change or fail, and gives a count of
the rest.

| Reason | Why |
|---|---|
| Not found / merged | Deleted, or merged into another bill (named). Recompute the bill it went into |
| Credit note / liability bill | Neither carries withholdings of its own |
| No withholdings | The bill names no withholding taxes and has no withholding rows |
| Cancelled or reversed | The bill no longer stands |
| On a pending payment voucher | The voucher holds the amount it pays the supplier, net of the withholdings. Cancel the voucher, recompute, and raise it again. A paid or cancelled voucher does not stop it |
| Withholding paid or remitted | A withholding has gone out on a liability bill |
| Withholding would rise | The bill has money paid on it, and its total withheld would go up (see above) |
| Withholding outside the bill's list | A withholding row whose tax is not in `withholding_tax_ids`. Rebuilding from the list would drop it, so the bill needs a look by hand |

Running it again on a bill that has been done reports `unchanged`.
