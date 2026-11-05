# Lease opening balance repair

A console command that puts one lease's statement back in line with its live opening balances. It
has no endpoint. Run it by hand on the server. It reports only, unless `--apply` is passed.

```bash
php artisan lease-opening-balances:repair 8107 --withdraw-invoice=218137 --withdraw-invoice=218138
php artisan lease-opening-balances:repair 8107 --withdraw-invoice=218137 --withdraw-invoice=218138 --apply
php artisan lease-opening-balances:repair 2579 --withdraw-credit-note=213460 --withdraw-credit-note=213461
```

## What it fixes

Before the fix that shipped with this command, deleting an opening balance always raised a reversal.
It did so even when the balance's own invoice or credit note had already been cancelled or deleted.
The tenant was then billed for the same balance twice. Each reversal also added the component's tax
rate on top of `billed_amount`, which already included tax. On lease 8107, for example:

- the opening balance credit notes were cancelled
- the rows were then deleted, which raised reversal invoices for 160,814.43 and 1,877,761.98
  (1,618,760.33 plus 16%)
- the credit note behind the re-entered opening balance was cancelled as well

That left the lease screen showing an opening balance of −1,643,633.37 while the statement carried
it as a debit of 2,038,576.41.

## Steps

The steps run in this order, all in one transaction:

| Step | What it does |
|---|---|
| 1. Withdraw | Each `--withdraw-invoice` or `--withdraw-credit-note` is cancelled and then deleted, together with its statement lines. Credit it spent goes back to its credit notes. Credit it gave goes back onto the invoices it paid |
| 2. Clear deleted documents' lines | Statement lines whose invoice or credit note no longer exists are removed. A deleted invoice already loses its lines. Until this fix, a deleted credit note kept its credit line and its `Cancelled Credit Note #` line |
| 3. Bill live opening balances | Rows that are not deleted, but whose invoice or credit note is no longer live (cancelled, rejected or deleted), are billed again. The rows in arrears go on one invoice and the rows in credit on one credit note, dated by `opening_balance_at`, exactly as when they were first entered. This step is skipped on a pending lease, which bills on approval |
| 4. Apply credit | The lease's open credit notes are applied to its open invoices, oldest first (`leases:apply-open-credit-notes --lease=`) |

The report lists every document and line each step touches. It also shows the statement balance now
and after the repair.

## Which documents can be withdrawn

The command does not decide which reversal was raised in error. Read the lease's activity log, then
pass the ids. Each id is refused unless the document:

- belongs to the lease
- is the reversal document of a **deleted** opening balance on it, which means the soft-deleted row
  links to it
- is not signed on the ETR. KRA holds a signed document, so it cannot be withdrawn
- for an invoice, has no receipt allocated to it

Any refusal stops the whole run before anything changes.

## Options

| Option | Meaning |
|---|---|
| `lease` | the lease id |
| `--withdraw-invoice=*` | a reversal invoice to withdraw (repeatable) |
| `--withdraw-credit-note=*` | a reversal credit note to withdraw (repeatable) |
| `--apply` | make the changes. Without it, nothing changes |
