# Property onboarding import (row 162)

Onboard a property in one go from a spreadsheet: its blocks, wings and floors, its units, the people
on them (tenants, and SC tenants who pay the service charge separately), their leases, the arrears brought forward, deposits held, and the
utility meters. Used when the company takes over management of a property: create the property
first (with its type, and blocks or wings if it has them), send the onboarding sheet to the previous
manager, then upload what comes back. The sheet is shared outside the system for now; there is no
download endpoint.

Base: `/api/v1/app/{company}/property-management/facilities/{facility}/import`

| | Endpoint | |
| --- | --- | --- |
| Check a sheet | `POST import/validate` | Writes nothing |
| Import a sheet | `POST import/process` | All or nothing |

**Who:** `update-facility` (or the property's creator), `create-lease`, and allocated to the property.
Otherwise `403`, with `You are not allocated to this property.` for the allocation.

**Sync, not a job.** `process` does the whole import inside the request, in one database transaction,
and answers when it is done. There is nothing to poll. Give the request a long timeout and keep the
Import button disabled while it runs. `validate` is also synchronous.

## Requests

Both POSTs take **either** a file **or** rows:

| Body | Field | Rules |
| --- | --- | --- |
| `multipart/form-data` | `import_file` | `.xlsx`, `.xls` or `.csv`, up to 10 MB. Only the first sheet is read; its first row must be the header row, with the column names below |
| JSON | `rows` | 1 to 2,000 rows edited on the preview - see [Fixing cells on screen](#fixing-cells-on-screen) |

A request with neither, a wrong file type or a bad `rows` array gets the usual Laravel `422`
(`errors.import_file`, `errors.rows`, `errors.rows.0.row`). Problems *inside* the sheet are never a
`422` from validate: they come back as row errors in a `200`, as below.

## The sheet

Row 1 is the header row. Each column is read by its name below, after lower-casing and turning spaces
into underscores (so `Unit Size` reads as `unit_size`). Any other name, such as `Service Charge` for
`sc`, is not read. Column order does not matter. Then one row per unit:

| Column | Notes |
| --- | --- |
| `block` | Optional. Required on every row once the property has blocks - see [Blocks and wings](#blocks-and-wings) |
| `wing` | Optional. Required on every row of a block (or a property with no blocks) that has wings |
| `floor` | Required. Text, so `G`, `0`, `1` all work |
| `unit` | Required. Unique on its floor |
| `unit_size` | Square feet. Required for a new unit on a commercial (size-divided) property; ignored on residential |
| `tenant_name`, `tenant_email`, `tenant_phone`, `tenant_pin` | Who rents it, and pays the rent. Name, email and KRA PIN required when any is given. Leave blank for a vacant or owner-occupied unit |
| `sc_tenant_name`, `sc_tenant_email`, `sc_tenant_phone`, `sc_tenant_pin` | Who pays the service charge when it is not the tenant (e.g. the unit's owner). Same rules as the tenant |
| `bill_sc_separate` | `true` / `false` (also `yes`/`no`, `1`/`0`, `t`/`f`). Blank is `false`. `true` gives the SC tenant their own service-charge lease; `false` puts the service charge on the tenant's lease - see [Who pays what](#who-pays-what) |
| `lease_start_date` | `YYYY-MM-DD` or `DD/MM/YYYY`. Required when the unit has a tenant or an SC tenant |
| `lease_term_years`, `lease_term_months` | The term; months 0-11. At least one above 0 |
| `billing_cycle` | `monthly` (default), `quarterly`, `biannual`, `annually`, `biennial`, `weekly`, `daily` |
| `rent_per_month`, `rent_tax` | Required for a tenant (0 if none). Tax: a rate (`16`, `0`) or a tax name (`Exempt`); blank is 0% |
| `sc`, `sc_tax` | Monthly service charge and its tax |
| `deposit` | The deposit the tenant has already paid. Recorded as held, not billed again |
| `arrears_rent`, `arrears_sc`, `arrears_water`, `arrears_electricity` | Owed at the takeover |
| `electricity_meter_number`, `electricity_previous_reading` | The meter and its last reading |
| `water_meter_number`, `water_previous_reading` | The same for water |

Only `floor` and `unit` must be there. Any other column may be left out of the sheet altogether (a
commercial sheet often has no SC tenant columns); a missing column reads as blank. Blank rows are
skipped.

## What a row creates

- **Structure:** blocks, wings, floors and units are matched by name (case-insensitive) and created
  when missing. A block or wing can also set the property or block up - see
  [Blocks and wings](#blocks-and-wings). On a commercial (size-divided) property a new unit needs a `unit_size` (blank is a
  row error), and a new floor is sized to the units the sheet puts on it. An existing block, wing,
  floor or the property itself must have the leasable space for them, or the row is refused (the
  property's or block's declared space is never increased).
- **People:** matched by KRA PIN, else created, and given the tenant portal. Nobody is emailed or
  texted - see [Messages](#messages).
- **Leases**, all active straight away (existing tenancies, no approval). There are two kinds, and the
  summary and counts give each separately: the **rent lease** (the tenant's) and the **service charge
  lease** (an SC tenant's). Who gets which is in [Who pays what](#who-pays-what). Then:
  - billing picks up at the **first billing date after the import**, counted from the start date.
    The current period was billed by the previous manager, and anything unpaid comes across as
    arrears.
- **Arrears** become opening balances, one invoice per lease: rent on the tenant's lease, service
  charge on whoever pays it, water and electricity on the tenant's lease (the SC tenant's if there is
  no tenant).
- **Deposit:** held on the tenant's lease against its rent.
- **Meters:** created with the previous reading as their starting reading, linked to the lease that
  pays utilities (the tenant's, else the SC tenant's). A vacant unit's meter is created on no lease.

## Who pays what

The tenant is the primary person on a unit:

| Sheet | Leases |
| --- | --- |
| Tenant only | One rent lease, with rent and service charge |
| Tenant + SC tenant, `bill_sc_separate` **true** | The tenant's rent lease (rent), and the SC tenant's service charge lease (service charge) |
| Tenant + SC tenant, `bill_sc_separate` **false or blank** | One rent lease for the tenant, with rent and service charge. The SC tenant is not used and not created; the row warns `Peter Otieno (the tenant) will pay the service charge. To bill Jane Kamau instead, set bill_sc_separate to true.` |
| SC tenant only (owner-occupied) | One service charge lease, whatever `bill_sc_separate` says. Rent and deposit must be blank or 0 |
| `bill_sc_separate` **true** with no SC tenant | Row error on `bill_sc_separate` |

The same KRA PIN as tenant and SC tenant is the same person: one lease.

## Blocks and wings

Both are optional, and the sheet can set the property up for them:

- **Blocks:** a block on a property without blocks sets the property up with blocks (the preview says
  `Sets up <property> with blocks`). Then every row needs a block.
- **Wings:** a wing sets its block up with wings (`Sets up block A with wings`, or a new block is
  created with them), or - with no block - sets the property up with wings. Then every row of that
  block, or of the property, needs a wing.

Refused, as a row error:

- giving blocks to a property that already has floors directly on it, or that has its own wings (a
  property has blocks or wings, not both);
- giving wings to a property or a block that already has floors directly on it (those floors would
  sit where floors are no longer allowed);
- a row without a block (or wing) once the property (or block) has them.

Nothing is switched on until the sheet is imported, and only if the whole sheet imports.

## Uploading the same sheet again

Nothing is created twice. A unit whose rent or service charge is already held by **the same person's**
lease is `already_set_up` and left as it is: its arrears, deposit and meters are not added again. A
meter that already exists on an existing unit of this property is left as it is. A unit already let
to **someone else** is an error.

## Check a sheet

`POST import/validate` writes nothing. It answers `200` whether or not the sheet has errors:

```json
{
  "data": {
    "message": "The sheet is ready to import.",
    "has_errors": false,
    "errors": {},
    "summary": {
      "total_rows": 2,
      "ready_rows": 2,
      "already_set_up_rows": 0,
      "error_rows": 0,
      "new_blocks": 1,
      "new_wings": 0,
      "new_floors": 1,
      "new_units": 2,
      "new_people": 2,
      "matched_people": 0,
      "new_leases": 2,
      "new_rent_leases": 1,
      "new_service_charge_leases": 1,
      "new_meters": 3
    },
    "rows": [
      {
        "row": 2,
        "unit": "Z-3-07",
        "location": "Block Z / Floor 3 / Z-3-07",
        "cells": {
          "block": "Z", "wing": null, "floor": "3", "unit": "Z-3-07", "unit_size": null,
          "tenant_name": "Peter Otieno", "tenant_email": "peter.otieno@example.com",
          "tenant_phone": "0733987654", "tenant_pin": "A009876543C",
          "sc_tenant_name": "Jane Wanjiru Kamau", "sc_tenant_email": "jane.kamau@example.com",
          "sc_tenant_phone": "0722123456", "sc_tenant_pin": "A001234567B", "bill_sc_separate": "true",
          "lease_start_date": "2025-01-01", "lease_term_years": "2", "lease_term_months": "0",
          "billing_cycle": "monthly", "rent_per_month": "45000", "rent_tax": "0", "sc": "5000",
          "sc_tax": "16", "deposit": "45000", "arrears_rent": null, "arrears_sc": null,
          "arrears_water": "1200", "arrears_electricity": "850",
          "electricity_meter_number": "EM-1049823", "electricity_previous_reading": "12456",
          "water_meter_number": "WM-220071", "water_previous_reading": "3380"
        },
        "status": "ok",
        "has_error": false,
        "errors": {},
        "warnings": [],
        "actions": [
          "Creates block Z",
          "Creates floor 3",
          "Creates unit Z-3-07",
          "Creates tenant Peter Otieno (A009876543C)",
          "Creates SC tenant Jane Wanjiru Kamau (A001234567B)",
          "Lease for Peter Otieno: rent 45,000.00",
          "Service charge lease for Jane Wanjiru Kamau: 5,000.00",
          "Deposit held: 45,000.00",
          "Arrears brought forward: 2,050.00",
          "Electricity meter EM-1049823, starting reading 12,456",
          "Water meter WM-220071, starting reading 3,380"
        ]
      },
      {
        "row": 3,
        "unit": "Z-3-08",
        "location": "Block Z / Floor 3 / Z-3-08",
        "cells": { "block": "Z", "floor": "3", "unit": "Z-3-08", "water_meter_number": "WM-220072", "water_previous_reading": "10", "...": "every other column, null" },
        "status": "ok",
        "has_error": false,
        "errors": {},
        "warnings": ["Vacant: no lease will be created."],
        "actions": ["Creates unit Z-3-08", "Water meter WM-220072, starting reading 10"]
      }
    ]
  }
}
```

A row with an error:

```json
{
  "row": 2,
  "unit": "A-3-07",
  "location": "Block A / East / Floor 3 / A-3-07",
  "cells": { "block": "A", "wing": "East", "...": "every column" },
  "status": "error",
  "has_error": true,
  "errors": { "wing": "Block A already has floors without wings, so wings can't be added here. Leave the wing blank." },
  "warnings": [],
  "actions": ["Creates tenant Peter Otieno (A009876543C)", "..."]
}
```

The fields:

- `message`: `The sheet is ready to import.` or `Some rows need fixing before the sheet can be imported.`
- `has_errors`: true when any row has an error, or there is a top-level error. Import only when false.
- `errors` (top level): problems with the whole sheet rather than a cell. See
  [Sheet errors](#sheet-errors).
- `summary`: row counts, then what the import *would* create. Rows with errors are left out of the
  "new" counts. `new_leases` = `new_rent_leases` + `new_service_charge_leases`. Blocks, floors and
  people are counted once each, however many rows name them.
- `rows[]`, one per non-blank row:
  - `row`: the sheet row number (the header is row 1), or the `row` you sent back.
  - `unit`, `location`: for the row's label.
  - `cells`: the row as read, every column by name (`null` when blank).
  - `status`: `ok` (will be created), `already_set_up` (left as it is), or `error` (stops the import).
  - `errors`: keyed by **column name**, the same keys as `cells`, each a message to show next to that
    cell. So a cell error is at `rows[i].errors[column]`, and `rows[i].row` tells the user which line.
    Every key is a column, so every error has a cell to fix it in.
  - `warnings`: shown but not blocking: `Vacant: no lease will be created.`, `Already imported: this
    unit's leases are in the system, so this row will be skipped.`, `Meter X is already in the system, so
    it will be skipped.`, the bill_sc_separate one above, and on the first row of a KRA PIN or email
    clash: `Row 8 has KRA PIN A009876543C with a different email, mary@example.com. Use the same email on
    both rows.` / `Row 8 uses peter@example.com with a different KRA PIN, A007777777D. An email belongs to
    one person.` (the later row carries the error).
  - **A KRA PIN already on file under another name** is not an error: the PIN is the person, so the
    lease goes to them and the row warns `KRA PIN A001234567B is registered to Faith Achieng, so the
    lease goes to them; the sheet says Collins Barasa. Check the PIN if that is not who you meant.`
    The name on file is not changed. Only case and spacing differences (`PETER  otieno` for
    `Peter Otieno`) pass without a warning. Show these warnings prominently on the preview: they are
    the only sign of a mistyped PIN.
  - `actions`: what importing the row will do, in plain words. A new block, floor or person, and a
    property or block being set up with blocks or wings, is described on the first row that does it.

## Errors

### Cell errors (`rows[].errors`)

Every message is written to sit next to its cell, and still makes sense in a list. `<Property>` is
the property's name. Column names appear only where the user has to type one (`bill_sc_separate`).

| Key | Message |
| --- | --- |
| `block` | `Enter the block: <Property> is set up with blocks.` / `Other rows have a block, so this row needs one too.` / `<Property> already has floors without blocks, so blocks can't be added here. Leave the block blank.` / `<Property> uses wings, not blocks. Leave the block blank.` |
| `wing` | `Enter the wing: <Property> is set up with wings.` / `Enter the wing: block A is set up with wings.` / `Other rows have a wing, so this row needs one too.` / `Other rows in block A have a wing, so this row needs one too.` / `<Property> already has floors without wings, so wings can't be added here. Leave the wing blank.` / `Block A already has floors without wings, so wings can't be added here. Leave the wing blank.` |
| `floor`, `unit` | `Enter the floor.` / `Enter the unit.` |
| `unit` | `This unit appears on rows 4 and 9. Each unit goes on one row only.` / `A-3-07 is already rented to Peter Otieno.` / `A-3-07 already has a service-charge lease with Jane Kamau.` |
| `unit_size` | `Enter the unit's size in sq ft.` / `Not enough space: Block A has 1,000 sq ft free, but these units need 1,100 sq ft. Increase Block A's leasable space first.` |
| `tenant_*`, `sc_tenant_*` | `Enter the tenant's name.` (also `email`, `KRA PIN`; `SC tenant's` for the SC tenant) / `The tenant's email isn't a valid address.` / `Not a valid KRA PIN. It should look like A001234567B.` |
| `tenant_email`, `sc_tenant_email` | `KRA PIN A009876543C is on row 2 with peter@example.com. Use the same email on both rows.` / `peter@example.com is on row 2 with KRA PIN A009876543C. An email belongs to one person: use their KRA PIN, or another email.` - on the later row; the first row gets a warning naming this one, since either may be the wrong one. When the tenant and SC tenant on one row clash: `KRA PIN A009876543C is also the tenant on this row, with peter@example.com. Use the same email for both.` / `peter@example.com is also the tenant's on this row, with KRA PIN A009876543C. An email belongs to one person: use their KRA PIN, or another email.` |
| `*_email`, `*_phone` | `This email is already used by someone with a different KRA PIN. Check the email or the PIN.` / `This phone belongs to someone else, not the person with KRA PIN A001234567B.` |
| `lease_start_date` | `Enter the lease start date.` / `Enter the lease start date as a date like 31/01/2025.` |
| `lease_term_years`, `lease_term_months` | `Enter the lease term in years, months or both.` / `Months must be 0 to 11. Put whole years in the years column.` |
| `billing_cycle` | `Use monthly, quarterly, biannual, annually, biennial, weekly or daily.` |
| `rent_per_month` | `Enter the monthly rent (0 if there's none).` / `Only a tenant pays rent.` / `Only a tenant pays rent; an owner-occupied unit pays service charge only.` |
| `rent_tax`, `sc_tax` | `No tax called "VAT 8". Enter a rate (16 or 0) or a tax name.` |
| `deposit` | `Only a tenant can have a deposit.` / `The deposit can't be negative.` |
| `arrears_*` | `Arrears need a tenant or SC tenant on this row.` / `Only a tenant can owe rent arrears.` |
| `bill_sc_separate` | `Enter true or false (blank means false).` / `Add the SC tenant's details, or set this to false.` |
| any number column | `Enter a number for monthly rent.` / `Unit size can't be negative.` (named in plain words: unit size, lease term (years), lease term (months), monthly rent, service charge, deposit, rent / service charge / water / electricity arrears, previous electricity / water reading) |
| `*_meter_number` | `Enter the water meter number for this reading.` / `Meter WM-220071 is already in the system. Check the number.` / `Water meters can't be added yet: the m³ unit isn't set up. Ask an admin.` / `This meter number appears on rows 3 and 7. Each meter goes on one row only.` (on each of those rows, under the column holding it) |

### Sheet errors

The top-level `errors` object can hold:

| Key | Message | Fix |
| --- | --- | --- |
| `file` | `The sheet has no unit rows. Check that row 1 is the header row.` | The file is empty, or the header row is missing or renamed |
| `rows` | `Each row must have its own row number.` | Two rows sent back with the same `row` |
| `component_rent`, `component_service_charge`, `component_water`, `component_electricity` | `No active Rent lease component is set up.` | Company set-up: the lease component must exist and be active |

## Fixing cells on screen

A user can correct the preview instead of the spreadsheet:

1. Show `rows[].cells` in an editable grid and mark the cells named in `rows[].errors`.
2. Send the corrected rows back to `validate` as JSON (`Content-Type: application/json`). Each row is
   its `cells` plus its `row`:

   ```json
   {
     "rows": [
       { "row": 2, "block": "A", "wing": null, "floor": "3", "unit": "A-3-07", "tenant_pin": "A009876543C", "...": "every column" },
       { "row": 3, "block": "A", "wing": null, "floor": "3", "unit": "A-3-08", "...": "every column" }
     ]
   }
   ```

3. When `has_errors` is false, send the same `rows` to `process`.

Rows sent back are checked exactly as a file is, so a fixed cell is checked again and an import with
any error left is still refused. Send **all** the rows, not only the fixed ones: the request is the
whole sheet. Keep each row's `row` so its errors point at the same line. Each number may appear only
once; a repeat answers `errors.rows`. Up to 2,000 rows. Re-uploading a corrected file works too.

## Import a sheet

`POST import/process` takes the same file, or the same `rows`. It checks everything again first. If
anything has an error, nothing is imported and the answer is `422`, with the validate body under
`data`:

```json
{
  "message": "Some rows need fixing before the sheet can be imported. Nothing was imported.",
  "data": { "has_errors": true, "errors": {}, "summary": { "...": "..." }, "rows": [ "..." ] }
}
```

On success (`200`) it returns what was created, plus the preview body as it was checked:

```json
{
  "data": {
    "message": "The sheet was imported.",
    "counts": {
      "blocks": 1,
      "wings": 0,
      "floors": 1,
      "units": 2,
      "people": 2,
      "leases": 2,
      "rent_leases": 1,
      "service_charge_leases": 1,
      "opening_balances": 2,
      "deposits": 1,
      "meters": 3,
      "skipped_rows": 0
    },
    "has_errors": false,
    "errors": {},
    "summary": { "...": "as validate" },
    "rows": [ "... as validate" ]
  }
}
```

- `counts` is what was actually written. `leases` = `rent_leases` + `service_charge_leases`.
- `opening_balances` counts the arrears lines brought forward, one per component with an amount
  (rent, service charge, water, electricity). Each lease's lines go on one invoice.
- `skipped_rows` counts the `already_set_up` rows that were left as they are.
- An unexpected failure is a `500` with `Error importing the sheet. Nothing was imported.` The
  transaction is rolled back, so the sheet can simply be sent again.

## Messages

The import messages nobody: no welcome email to new people (they set a password through "forgot
password" when they are invited later), no lease notification, and opening-balance invoices are never
emailed.

## Suggested screen

1. On the property, upload the filled sheet to `validate`. Show the summary (with rent and service-charge leases
   separately) and the rows: errors next to their cells, warnings and actions under each row.
2. Let the user fix cells in place and re-check with `rows`, or upload a corrected file.
3. Enable **Import** only when `has_errors` is false. Send the same file or `rows` to `process`,
   wait for it, then show `counts`.
