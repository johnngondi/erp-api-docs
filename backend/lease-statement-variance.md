# Lease statement variance

A console check that every lease's tenant statement agrees with its ledger. There is no
endpoint; it is run by hand on the server and changes nothing.

```bash
php artisan leases:statement-variance                       # every lease with a variance
php artisan leases:statement-variance --company=3 --facility=12 --tolerance=1
php artisan leases:statement-variance --csv=storage/app/variance.csv
php artisan leases:statement-variance --lease=4141 --detail # line up one lease document by document
```

## What it compares

| | Source | Rule |
|---|---|---|
| **S**, statement | `tenant_statements` of the lease | `SUM(debit - credit)`, every status, exactly as the tenant statement screen (`GetTenantStatementService`) |
| **B**, billings | `lease_billings.total` of the lease | invoices positive, credit notes negative, no status filter |
| **C**, collections | `lease_collections.total` of the lease, status `confirmed` | the rule the landlord reports use (`FacilityLedgerTotals`) |

`variance = S - (B - C)`. `B - C` is the lease's `current_arrears`. A lease is listed when
`|variance|` is above `--tolerance` (default `0.01`), largest first.

All figures are in the **lease currency**. Billings and statement lines are kept in it.
Collections are kept in the property's reporting currency, so when that differs they are
converted to the lease currency at each collection's transaction-date rate
(`currency_rate_histories`, else the live rate). This is an approximation: the rate actually used
when the receipt was posted is not stored per collection. Such leases are marked `~` in the
`Currency` column.

Statement lines without a lease (a receipt's "Unallocated Receipt #" line) belong to no lease and
have no collection, so they are outside the comparison on both sides.

## Options

| Option | Meaning |
|---|---|
| `--company=` | only leases of this company |
| `--facility=` | only leases of this property |
| `--lease=*` | only these leases (repeatable) |
| `--tolerance=` | smallest variance listed, default `0.01` |
| `--csv=` | also write every listed lease to this CSV path |
| `--detail` | with exactly one `--lease`, show the document-by-document breakdown below |
| `--all` | with `--detail`, also show documents that carry no variance |

## `--detail`

One row per document on the lease, each with its share of S, B and C and its own variance, so
the document that breaks the balance stands out:

| Row | S | B | C |
|---|---|---|---|
| `Invoice #n` | statement lines of the invoice | billings with `invoice_id = n` | – |
| `Credit note #n` | statement lines of the credit note | billings with `credit_note_id = n` | – |
| `Receipt #n` | the receipt's statement lines on this lease (credits less cancellation debits) | collections of the receipt's allocations on this lease | ✓ |
| `Unlinked statement lines` | lines with no invoice, credit note or receipt behind them (imports, manual) | – | – |
| `Unlinked billings` | – | billings with neither invoice nor credit note | – |
| `Unlinked collections` | – | – | collections with no receipt allocation |

Each row's variance is `S - (B - C)`, and the rows add up to the lease's variance. Rows with no
variance are hidden unless `--all` is given.

## Reading a variance

- **Receipt row** with S but no C: the statement was credited but no collection was written
  (often an imported receipt). With C but no S: the opposite.
- **Invoice row**: billing lines and the statement debit disagree, e.g. an invoice edited after
  posting, or a cancelled invoice whose billing was removed but whose statement line stayed.
- **Unlinked rows**: imported or hand-made entries; compare them by date with the other side.
