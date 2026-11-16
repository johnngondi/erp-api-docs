# Facility LR numbers from EPMAS

A console command that copies each property's LR (land reference) number from EPMAS
(`properties.plot_no`) into the ERP (`facilities.lr_number`). Most ERP properties have no LR number,
and EPMAS holds one for every property it had.

It has no endpoint. Run it by hand on the server. It needs the read-only `epmas` connection
(`EPMAS_DB_*` in `.env`). It only reports unless `--apply` is passed, so run it once without
`--apply`, check the list, then run it again with `--apply`.

```bash
php artisan facilities:pull-epmas-lr-numbers
php artisan facilities:pull-epmas-lr-numbers --apply
php artisan facilities:pull-epmas-lr-numbers --facility=12 --facility=40
php artisan facilities:pull-epmas-lr-numbers --overwrite
php artisan facilities:pull-epmas-lr-numbers --overwrite --apply
```

| Option | Meaning |
|---|---|
| `--apply` | Write the LR numbers. Without it the command only lists them |
| `--overwrite` | Also replace an LR number the ERP property already has when EPMAS holds a different one. Without it those are listed and kept |
| `--facility=*` | Only these ERP property ids. Repeatable. Omit for every property |

## The list

One row per ERP property that would get an LR number:

| Column | Source |
|---|---|
| ERP ID | `facilities.id` |
| ERP Name | `facilities.name` |
| EPMAS Name | `properties.name` of each EPMAS property that became this ERP property, joined with `; ` |
| LR Number | The value that will be written: the distinct `plot_no` values of those EPMAS properties, trimmed, joined with `, ` |

Properties whose ERP LR number differs from EPMAS's, when `--overwrite` is not passed, follow in a
second table with a fifth column, `Current ERP LR Number`. They are kept.

EPMAS holds placeholders where nobody typed the number in (`N/A`, `L.R NO`, `KPA G`). A value with
no digit in it is never written. Properties whose EPMAS values are all placeholders follow in their
own table (`EPMAS Value`, `Current ERP LR Number`), so their numbers can be entered on the property's
edit page. When a facility has both, only the values with a digit are used.

A summary line counts the properties to write, those already matching, those kept, the
placeholders, and the EPMAS properties that match no ERP property. Run with `-v` to list those
unmatched EPMAS properties. Most of them are properties the import left out (inactive in EPMAS).

## How an EPMAS property is matched

The import kept EPMAS ids, so most ERP properties carry their EPMAS property as
`facilities.old_system_id` = `Property:{id}`. The command also follows the other ways the import
folded EPMAS properties together:

1. An EPMAS property imported as a block: `facility_blocks.old_system_id` = `Property:{id}`. Its
   facility gets the LR number, together with the other blocks' numbers.
2. A service-charge property merged into its rent property: the pairing recorded under
   `merged-properties` in `storage/app/epmas-import/decisions/*.json`, when those files are on the
   server.
3. The facility's own `old_system_id`.
4. An `epmas_import_aliases` row for `Property:{id}` on the facility model.

An ERP property created in the ERP after the import has no EPMAS property and is never listed.
Enter its LR number on the property's edit page.

## What changes

With `--apply`, each listed property's `lr_number` is set to the LR Number shown, saved through the
model so the activity log records the change. Nothing else is written, and nothing is written to
EPMAS. Running it again with the same data finds nothing new to write.

An EPMAS property with a blank `plot_no` is ignored. Spaces inside a value are collapsed to one and
the ends trimmed, so a number that differs only in spacing counts as matching. A combined LR number longer than 255
characters is not written and is reported.
