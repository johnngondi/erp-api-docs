# EPMAS history (reference only)

The EPMAS import brought each lease in with one opening balance dated at the cutover
(30 Sep 2026), and supplier bills only where they were still owed. Everything before that date was
collapsed into those balances. This command copies the EPMAS tenant ledger, the property expense
ledger and the landlord remittances (2017 to the cutover) into **reference-only** tables so trend
reports, the landlord Income & Expenditure report, analytics and AI have the history to work from.

```bash
php artisan history:import-epmas --dry-run             # compute and print everything, write nothing
php artisan history:import-epmas                       # (re)build tenants, expenses and remittances
php artisan history:import-epmas --only=expenses --only=remittances  # only these parts
php artisan history:import-epmas --only=tenants --lease=12 --lease=40 # only these leases
php artisan history:import-epmas --as-of=2026-09-30    # the cutover date
```

| Option | Meaning |
|---|---|
| `--only=` | `tenants`, `expenses` or `remittances`; repeatable. Default: all three. |
| `--lease=` | Narrows the `tenants` part to these leases. Expenses and remittances are always rebuilt per property, whole. |
| `--as-of=` | The cutover date; rows dated after the end of this day (app timezone) are left out. Default: `config('history.epmas_cutover')`, env `EPMAS_HISTORY_CUTOVER`, `2026-09-30`. Reports read the same setting. |
| `--dry-run` | Compute and print everything, write nothing. |

## The rule: history never touches a current figure

- History is written **only** to `lease_history_billings`, `lease_history_collections`,
  `facility_history_expenses` and `facility_history_remittances`. No existing table gets a row, a
  column or an index from it.
- **Nothing that computes a balance reads these tables**: not the tenant statement, ageing,
  debtors, arrears, remittances, management fees, budgets, dashboards or any existing report. Only
  the reports built for them do (the trend reports, and the Income & Expenditure report for the
  periods up to the cutover).
- No existing model has a relation to them. `LeaseHistoryBilling`, `LeaseHistoryCollection`,
  `FacilityHistoryExpense` and `FacilityHistoryRemittance` are read-only models: saving or deleting
  one through Eloquent throws.
- The importer writes with the query builder, so no model events, activity log rows or
  company-scope stamping happen. `company_id` is copied from the lease (tenants) or the facility
  (expenses, remittances).
- Deleting a lease deletes its history (`lease_id` cascades), so `DeleteLeaseAction` is unaffected.

## Which leases

A lease is in scope when

- its id is at most the highest EPMAS `tenants.id` (EPMAS `tenant_id` = ERP `leases.id`),
- it was created on or before the cutover (end of day, Africa/Nairobi),
- its `old_system_id` is `Tenant:{id}` (it really was imported from that tenant), and
- the EPMAS tenant's property resolves to the lease's facility, the way the importer resolved it:
  a `facility_blocks` row stamped `Property:{id}`, else a `merged-properties` decision in
  `storage/app/epmas-import/decisions`, else the `facilities` row stamped `Property:{id}`
  (`App\Services\History\EpmasPropertyFacilities`, shared by every part of the command).

A lease failing the last two checks is **skipped and listed**, never guessed. Leases created after
the cutover start above the highest EPMAS tenant id, so no EPMAS row can attach to them.

## Tables

### `lease_history_billings`

One row per EPMAS `tenant_billings` row per non-zero charge, plus the cutover reconciliation rows.

| Column | Meaning |
|---|---|
| `company_id`, `lease_id`, `facility_id` | from the ERP lease |
| `lease_component_id` | the charge (map below) |
| `entry_type` | `invoice`, `credit_note` (from the ledger row's `item_type`) or `cutover_reconciliation` |
| `transaction_date` | the ledger row's `txn_date` |
| `reference` | the ledger row's `ref` |
| `document_number` | EPMAS `invoices.invoice_no` or `credit_notes.ref` |
| `currency_id` | the invoice's / credit note's currency, else the lease currency |
| `amount`, `tax`, `total` | net charge, its prorated VAT, `amount + tax`; signs as in the ledger (credit notes negative) |
| `source_table`, `source_id` | `tenant_billings` and its id; unique with `lease_component_id` |

### `lease_history_collections`

One row per EPMAS `tenant_collections` row per non-zero charge. The same columns, with
`entry_type` always `payment`, and:

| Column | Meaning |
|---|---|
| `lease_component_id` | the charge; **NULL** for the row's `unallocated` money |
| `status` | `cancelled` when the EPMAS payment has `cancelled_at`, else `confirmed` |
| `receipt_number`, `document_number` | `payments.rct_no` |
| `payment_mode`, `payment_channel` | `payments.mode`, `payments.channel` |
| `is_withholding` | the payment settled the account with tax the tenant withheld (below) |
| `paid_at` | `payments.date_paid`, else the payment's `txn_date` |
| `currency_id` | the payment's currency, else the lease currency |

Count `confirmed` rows only, as for live `lease_collections`. EPMAS writes a reversal of a
cancelled payment as a negative row on a payment that also carries `cancelled_at`, so both halves
are `cancelled` and drop out together.

**Withholding.** EPMAS marks a receipt of tax the tenant withheld by its channel:
`withholding_vat` or `withholding_rental` (`payments.withholding` and `withholding_rate` exist but
are empty). `is_withholding` is true when the payment's `channel` or `mode` starts with
`withholding`, or `withholding` holds a value other than `0`, or `withholding_rate` is above 0. It
is the history counterpart of the live `payment_methods.is_withholding` (the seeded "VAT
Withholding" and "Rental Withholding" methods), and is what EPMAS deducted as `withheld` on a
remittance: for property 157, August 2026, the `withholding_vat` collections add up to the
remittance's `withheld` (305,744) to the shilling.

### Charge map

| EPMAS column | Lease component |
|---|---|
| `rent` | Rent |
| `service_charge` | Service Charge |
| `service` | Service |
| `electricity` | Electricity |
| `water` | Water |
| `parking` | Parking |
| `signage` | Signage |
| `promotion_fee` | Promotion Fee |
| `deposit` | Deposit |
| `excess_service_charge` | Excess Service Charge |

Components are looked up by name when the command starts; a missing one stops it.

### VAT

EPMAS keeps one `vat` column per row. The same rule as the opening balances spreads it over the
row's vatable charges (Rent, Service Charge, Parking, Signage, Promotion Fee, Excess Service
Charge, Electricity; never Deposit, Water or Service):

    tax[c] = vat × |amount[c]| / Σ |amount[vatable]|

rounded to the cent, with the rounding remainder on the largest line, so the row's taxes add up
to its `vat` (itself rounded to the cent). The tax carries the sign of `vat`. A row with VAT and
no vatable charge gets a tax-only Rent line (`amount` 0).

## Cutover reconciliation

History alone does not add up to the opening balances: those were rounded to the cent, counted
cancelled payments as collected, and prorated VAT over the whole balance rather than row by row.
So that a series built from history and live data ends at **exactly** today's balance, the command
writes one `cutover_reconciliation` row into `lease_history_billings` per lease and component that
has history or an opening balance:

    reconciliation = OB_ledger − (Σ history billings − Σ confirmed history collections)

for `amount`, `tax` and `total` each, dated at the cutover 23:59:59, `source_table`
`cutover_reconciliation`, `source_id` the lease id, in the lease currency. A row that is zero in
all three is not written.

- `OB_ledger` is the sum of the live `lease_billings` rows of that lease and component whose
  `invoice_id` or `credit_note_id` appears in `lease_opening_balances` (whether or not the
  opening-balance row is soft deleted): the opening-balance documents as they sit in the ledger.
- Unallocated collections (NULL component) count against Rent, as the opening-balance import did.

**Invariant**, for every lease and component, to 5 decimals:

    Σ history billings (incl. reconciliation) − Σ confirmed history collections = OB_ledger

The command checks it against the database after writing (in memory on `--dry-run`) and fails,
naming the offenders, if it does not hold.

### Combining history with live data

A combined billed/collected series is

- history rows (`lease_history_billings`, including the reconciliation, and `confirmed`
  `lease_history_collections`), plus
- live `lease_billings` **excluding the opening-balance documents** (`invoice_id` /
  `credit_note_id` in `lease_opening_balances`), plus live `confirmed` `lease_collections`.

The opening-balance documents are left out because history plus reconciliation already is them.
Any balance rebuilt from the combined series therefore equals today's balance to the cent. The
reconciliation rows are shown as their own line ("Cutover reconciliation"), never hidden inside
billed amounts. The SQL is in the database guide (section 3, `lease_history_billings`).

## Expenses (`--only=expenses`)

EPMAS recognised a property's expenses in `property_expenses`, one row per posted supplier bill
(`item_type` `App\Models\Bill`, `item_id` the bill), split by the side of the landlord's account it
falls on. That is the ledger copied into `facility_history_expenses`. The ERP imported only the
bills still owed at the cutover, into `facility_bills` with their EPMAS ids and, by design, **no**
`facility_expenses` row ("EPMAS recognised the expense"), so history and the live expenses do not
overlap.

**Which rows.** Every `property_expenses` row dated on or before the cutover whose `property_id`
resolves to an ERP facility (blocks, merged-properties decisions, facilities, as for leases). The
rest are listed by property with their row count and total, never guessed. Properties EPMAS
retired before the migration (not imported as facilities) are most of them.

### `facility_history_expenses`

One row per `property_expenses` row per non-zero category column.

| Column | Meaning |
|---|---|
| `company_id`, `facility_id` | the resolved facility and its company |
| `vendor_id` | the ERP user of the EPMAS contractor (`users.old_system_id = 'Merchant:{id}'`, else an `epmas_import_aliases` row); NULL when the supplier was not imported. No foreign key |
| `bill_id` | the EPMAS bill id (`item_id`). Equal to `facility_bills.id` for a bill the migration imported; no foreign key, most EPMAS bills were never imported |
| `expense_type_id`, `expense_sub_type_id` | the bill's contract's `administrative_expense_id` / `administrative_expense_subcategory_id` (ids kept by the migration), NULL when the ERP has no such id or the bill has no contract |
| `expense_category_id` | from the column (below) |
| `transaction_date` | `txn_date` |
| `reference`, `notes` | `ref`, `notes` |
| `document_number` | the bill's `invoice_number`, else `bill_no` |
| `currency_id` | the bill's `currency` by code, else the facility's reporting currency |
| `amount`, `tax`, `total` | net, VAT, gross; signs as in EPMAS (credit bills negative) |
| `source_table`, `source_id` | `property_expenses` and its id; unique with `expense_category_id` |

**Category map.** `rent` → Rent (1), `service_charge` → Service Charge (2), `services` and
`service` → Service (3), the migration's rule for expense categories. On dev the column always
equals the bill's own `category` and every row carries exactly one of them. It is not the expense
type's category: a "rent" type such as Letting Fees is often charged to service charge and the
other way round, so classify by `expense_category_id`, never through the type.

**VAT.** `billed` (a generated column, `rent + service_charge + services + service + vat`, to 4
decimals) is the row's gross. `amount` is the category column and `tax = billed − Σ amounts`
(EPMAS's `vat`, to the 4 decimals `billed` keeps). On a row with several columns the tax is spread
over them by size, the remainder on the largest; a row with tax and no amount puts it on the bill's
`category`, else Service Charge. So `Σ total` is `Σ billed` exactly.

**Double counting guard.** A history row whose bill also has a live `facility_expenses` row
(`bill_id`, not soft deleted) is **skipped and listed**: the live expense already counts it. None
on dev.

**What is kept as EPMAS has it.** Rows of bills EPMAS later cancelled, and the few bills posted to
the ledger twice, are copied like any other row: the ledger is the record of what EPMAS deducted.
The command prints both so an analyst can exclude them (`facility_bills` is not needed:
`bill_id` joins straight to EPMAS when it is reachable).

**Tie-out**, printed, for the properties in scope:

    Σ facility_history_expenses.total = Σ property_expenses.billed − Σ skipped (double counted)

The EPMAS side is summed on the EPMAS connection (`CAST(billed AS DECIMAL(20,4))`), the history
side from the database after writing (in memory on `--dry-run`). The command fails if they differ.

On dev (2026-10-23): 28,649 rows on 98 facilities, 2018-12 to 2026-09-30, totalling
4,054,175,579.8763, which is EPMAS `billed` of those properties exactly (Service Charge 3,087.6
million, Rent 751.9 million, Service 214.7 million); none skipped as double counted; 582 rows
(117.4 million) on cancelled bills and 36 bills posted twice, kept; 49 rows whose bill EPMAS no
longer has (no expense type); 1,012 rows with no ERP supplier; 70 retired properties not resolved
(14,064 rows, 1,289.6 million).

## Remittances (`--only=remittances`)

EPMAS `remittances` are copied into `facility_history_remittances`, every row that is not
`cancelled` and whose `period_from` is on or before the cutover, on a resolved property. The
regular monthly remittance, its collections, expenses, tax withheld and the amount paid to the
landlord, and the advances paid before a period closed.

| Column | Meaning |
|---|---|
| `company_id`, `facility_id` | the resolved facility and its company |
| `is_advance` | EPMAS `is_advance` |
| `status` | EPMAS `status` as it is, lower-cased: `submitted` (approved and paid out, `approved_by` and `submitted_at` set) or `pending` (a draft EPMAS opened for the month and never submitted). `cancelled` rows are not copied |
| `period_from`, `period_to` | as in EPMAS; an advance's are both the moment it was made |
| `processed_at` | `date_processed`, else `submitted_at` (`date_processed` is never filled) |
| `collected`, `expenses`, `withheld` | as in EPMAS: the period's collections, the expenses deducted (the management fee bill included) and the tax tenants withheld |
| `remittable_amount` | EPMAS `amount`: what the landlord is paid |
| `currency_id` | the property's `default_currency` by code, else the facility's reporting currency |
| `reference` | for an advance, the EPMAS id of the regular remittance it was deducted on (`remittance_id`); else NULL |
| `source_table`, `source_id` | `remittances` and its id; unique together |

**`remittable_amount` = EPMAS `amount`.** On a submitted regular remittance
`amount = collected − expenses − withheld − balance + balance_bf`, where `balance` is the advances
deducted (4,135 of 4,139 rows to the shilling), the same thing the ERP's
`ComputeRemittableAmountService` computes (remittable collections − management fee − expenses −
advances). On an advance it is the amount advanced, as the ERP's typed-in `remittable_amount`.
Pending drafts often hold stale figures.

**Accepted statuses.** An approved or paid EPMAS remittance is `submitted`; `pending` is not. The
ERP's own rule for advances, "status not pending, cancelled or rejected", therefore keeps
`submitted` and drops `pending` when applied to the text as stored.

**Tie-out**, printed: `Σ remittable_amount` written = `Σ amount` of the EPMAS rows in scope.

On dev (2026-10-23): 5,489 rows on 101 facilities: 2,828 submitted and 2,479 pending regular
remittances, 181 submitted advances (449.9 million) and 1 pending; submitted remittable amount
3,580.8 million; tie-out exact at 3,615,567,697.55378; 109 cancelled rows left out; 69 retired
properties not resolved (2,155 rows).

## Reports that read history

History is read only by these reports, and only for dates on or before the cutover (the end of
`config('history.epmas_cutover')`, default `2026-09-30`, in Africa/Nairobi). None of them changes a
figure for a period after the cutover.

- **Billing & Collection Trend** and **Occupancy Trend** (tenant reports), through
  `Reports\Support\CombinedTenantLedger`.
- **Income & Expenditure** (landlord report), through `Reports\Support\IncomeExpenditureHistory`,
  unless the request sends `include_history=false`. It adds `confirmed` history collections to
  income (an unallocated payment under Rent, `is_withholding` feeding Tax Withheld),
  `facility_history_expenses` to expenses and advance `facility_history_remittances` in EPMAS status
  `submitted` to Less Advance Remittance. Each history row is converted from its own currency into the
  property's reporting currency first. See the report's API page for the details.

## Re-running

The command rebuilds. Tenants: per lease, in one transaction, it deletes that lease's rows from
both lease history tables and inserts them again; a lease that is skipped on a run has its history
cleared. Expenses and remittances: per facility, in one transaction, it deletes the facility's rows
and inserts them again, and after a full pass deletes the rows of any facility no longer in scope.
A second run writes the same rows.

## Output

Tenants:

- rows written per table, and the EPMAS rows dated after the cutover that were left out;
- totals: history billings, confirmed and cancelled collections (and the confirmed withholding
  collections among them), history net, the live opening-balance ledger total, the reconciliation
  total and the reconciliation per component;
- the 20 leases with the largest reconciliation (by the sum of its absolute component totals);
- counts and the first 20 ids of: leases skipped (no EPMAS tenant, not imported from it, facility
  mismatch or unresolved), unknown currency codes, and opening-balance components EPMAS has no
  column for (they reconcile like any other).

Expenses: rows written, facilities, rows after the cutover; totals by category and by year; the
tie-out; rows on cancelled bills and bills in the ledger more than once (kept, counted); and the
lists of properties not resolved (row count and total), bills skipped as double counted, rows with
no EPMAS bill, expense types and sub types the ERP does not have, and unknown currency codes.

Remittances: rows written by status, regular and advance; advances count and total (all, and
`submitted`); remittable amount by year; the tie-out; cancelled rows left out; and properties not
resolved.

`--dry-run` computes and prints all of it and writes nothing.

## Production run

1. Take a database backup.
2. `php artisan migrate` (creates the four history tables and the `is_withholding` column on
   `lease_history_collections`, and nothing else).
3. The EPMAS database must be reachable from the server through the `epmas` connection
   (`EPMAS_DB_*` in `.env`), or be restored there as `epmas_to_import`. The command only reads it.
4. Run `php artisan history:import-epmas --dry-run` and check the opening-balance ledger total, the
   reconciliation and the two tie-outs, then run it without `--dry-run`. Run it out of hours: it
   reads every lease's ledger, but only reads live tables, so no live table is locked for writing.
   On dev the tenants part (about 360,000 rows) takes 30 seconds to 2 minutes, expenses and
   remittances (about 34,000 rows) about 10 seconds; the command stops Telescope recording, which
   would otherwise format the bindings of every 1,000-row insert and take many minutes.
5. Rollback is rolling back the two migrations, newest first:
   `php artisan migrate:rollback --path=database/migrations/2026_10_23_110000_create_facility_history_tables.php`
   (drops the expense and remittance history and the `is_withholding` column) and
   `php artisan migrate:rollback --path=database/migrations/2026_10_23_100000_create_lease_history_tables.php`
   (drops the lease history). Nothing else changes.
