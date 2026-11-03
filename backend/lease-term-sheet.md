# Lease term sheet

Two console commands for correcting lease terms in bulk (row 174). The EPMAS import kept each
tenancy's own end date but counted its term as the whole months between start and end, and an
inclusive end is a month under that: a six-year lease came in as 5 years 11 months. The end dates
are right; the terms are not. There is no endpoint; both are run by hand on the server.

```bash
php artisan leases:export-terms                                   # storage/app/lease-terms-<date>.xlsx
php artisan leases:export-terms --facility=12 --path=/tmp/terms.xlsx
php artisan leases:fix-terms storage/app/lease-terms-corrected.xlsx           # preview only
php artisan leases:fix-terms storage/app/lease-terms-corrected.xlsx --apply   # write, and write an undo sheet
php artisan leases:fix-terms storage/app/lease-terms-corrected-undo-20261022-1430.xlsx --apply   # put them back
```

## The sheet

Every active, suspended and pending lease (terminated and rejected ones are left out), one row each:

| Column | |
| --- | --- |
| `lease_id`, `epmas_id`, `property`, `units`, `tenant`, `status` | Which lease. Only `lease_id` is read back |
| `start_date` | Read back only to check it is unchanged: this does not move a start |
| `period_in_years`, `period_in_months` | The term. **Correct these** |
| `end_date` | The end. Change it only where the end itself is wrong; blank keeps the lease's end |
| `matches` | `yes` when start + term, less a day, is the end date |
| `suggested_years`, `suggested_months` | The term that runs exactly to the end date. Blank when the tenancy does not run whole months (15 Jun to 13 Jun): a person decides those |

The export ends with a count: how many disagree, how many have a suggestion, how many need a person.
Dates go out as `YYYY-MM-DD`; a date Excel turned into a date cell is read back either way.

## Reading it back

`leases:fix-terms` reads the sheet by its heading row and lists, per row:

- **change**: the term, and the end if the sheet moved it, from and to. With notes when the new term
  still does not run to the end date, and when an end moved later than where billing stopped, so the
  next invoice run bills the months after the old end ("Billing picks up again: …").
- **unchanged**: the sheet agrees with the lease.
- **refused**, with why: no or unknown `lease_id`, a lease on two rows, a lease not active, suspended
  or pending, months over 11, a term of 0, a start date changed, an end before the start, an end
  earlier than what the lease is already billed up to (the day before `next_due_at`).

Nothing is written without `--apply`. With it, a sheet with any refused row is refused whole; fix or
remove those rows and run it again. Otherwise every change goes in one transaction, and each lease's
edit is in its activity log.

When an end moves, an escalation that ended on the lease's old end moves with it. One with its own
end date is left alone.

## Undo sheet

Before `--apply` writes anything, it writes an **undo sheet**: the leases it is about to change, as
they are, in this same format. It goes next to the corrected file (`<file>-undo-<date>-<time>.xlsx`),
or to `--undo=<path>`; if it cannot be written, nothing is changed. Running it through
`leases:fix-terms … --apply` puts those leases' terms and ends back, and the escalations moved with
an end move back with it. It is refused like any sheet: if the lease has been billed past the old
end since, that row says so.
