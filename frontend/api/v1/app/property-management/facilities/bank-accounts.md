# Facility Bank Accounts API

Domain: `Property Management > Properties`

Base route:

`/api/v1/app/{company}/property-management/facilities/{facility}/bank-accounts`

## Overview

A `facility_bank_account` is the **facility-level projection** of which bank
account collects rent/charges (or pays expenses) for one property — it's what
the `payment_details` document-template block (on invoices) reads to print
"pay to" instructions. It is a sibling of `landlord_default_accounts` (the
landlord-level routing used for remittance/payout), not a replacement for it —
both tables are kept in sync at landlord-creation time (see §4).

Each row links one `facility` to one `bank_account` for one `purpose`:

| `purpose` | Meaning |
| --- | --- |
| `collection` | Collects rent/charges from tenants |
| `expense` | Pays facility expenses out |
| `both` | Both of the above |
| `remittance` | Landlord remittance payout account, projected onto this facility |

`purpose` is a plain string on the model, cast through the
`FacilityBankAccountPurpose` PHP enum — it is **not** a DB `enum()` column, so
new values can be added without a migration.

A `collection`/`both` account may optionally be scoped to specific
`lease_components` (e.g. only collects "Rent", not "Water") via the
`lease_components` array column — `null`/`[]` means it collects for **all**
components. `lease_components` is rejected on an `expense`-purpose row (a
component split is meaningless on money going out).

## Endpoints

Standard resource routes:

- `GET    /facilities/{facility}/bank-accounts` — list (filters: `purpose`, `bank_account_id`, `is_active`; sorts: `id`, `purpose`, `created_at`)
- `POST   /facilities/{facility}/bank-accounts` — create
- `GET    /facilities/{facility}/bank-accounts/{facilityBankAccount}` — show
- `PUT|PATCH /facilities/{facility}/bank-accounts/{facilityBankAccount}` — update
- `DELETE /facilities/{facility}/bank-accounts/{facilityBankAccount}` — soft delete

## Authorization

| Action | Permission |
| --- | --- |
| Index / Show | `view-facility-bank-account` |
| Store | `create-facility-bank-account` |
| Update | `update-facility-bank-account` |
| Destroy | `delete-facility-bank-account` |

## Request body (store / update)

```jsonc
{
  "bank_account_id": 14,
  "purpose": "collection", // collection | expense | both | remittance
  "lease_components": [3, 7], // optional; rejected when purpose = expense
  "is_active": true, // optional, defaults true
}
```

`facility_id` is set from the route — never send it in the body.

Unique constraint: one row per `(facility_id, bank_account_id, purpose)`. Adding
a second `collection` row for the same account on the same facility is
rejected; instead update the existing row's `lease_components`.

## Response shape

Index/show items:

```jsonc
{
  "id": 1,
  "facility": { "id": 4, "name": "Westlands Court" },
  "bank_account": {
    "id": 14,
    "account_name": "Kariuki Estates Ltd",
    "account_number": "1102938475",
    "bank_branch": { "id": 2, "name": "Westlands", "bank": { "id": 1, "name": "KCB" } },
    // … full BankAccountResource shape
  },
  "purpose": { "value": "collection", "color": "success", "label": "Collection" },
  "lease_components": [{ "id": 3, "name": "Rent" }, { "id": 7, "name": "Water" }], // null when it collects for all components
  "is_active": true,
  "created": { "raw": "…", "formatted": "01 Sep, 2026", "diff": "…" },
  "permissions": { "view": true, "update": true, "delete": true },
}
```

Store/update responses use the `DataResource` envelope:
`{ "data": { "message": "…", "facility_bank_account": { …above shape… } } }`.

## Landlord-creation fan-out

The landlord-creation wizard (`POST /users/landlords`) already accepts a
top-level `accounts[]` array for the landlord's own default accounts
(`landlord_default_accounts`). That same `accounts[]` now **also fans out** one
`facility_bank_account` row per property created in the same submission:

```jsonc
{
  // … landlord fields …
  "contracts": [
    {
      "property": { "name": "Westlands Court", /* … */ },
      "contract": { /* … */ },
      // optional — overrides the top-level accounts[] for THIS property only
      "accounts": [
        { "bank_account_id": 14, "component": "collection" }
      ]
    },
    { "property": { "name": "Kilimani Heights", /* … */ }, "contract": { /* … */ } }
  ],
  "accounts": [
    { "bank_account_id": 14, "component": "collection", "created": false }
  ]
}
```

- Every `contracts[]` entry that produces a facility gets the **top-level**
  `accounts[]` fanned onto it, **unless that contract supplies its own
  `accounts[]`**, in which case the contract's list wins for that property only
  (Kilimani Heights above gets the top-level list; Westlands Court gets only
  its own override).
- `component` normalization matches `landlord_default_accounts`' existing rule:
  a string (`collection`/`expense`/`remittance`) maps straight to that
  `purpose`; a numeric value is a `lease_components` id and always implies
  `purpose: collection`, with that id appended into the row's
  `lease_components` array (multiple numeric-component entries for the same
  `bank_account_id` merge into **one** row's array, they don't create separate
  rows — the unique constraint above wouldn't allow that anyway).
- This fan-out is additive to (not a replacement for) the existing
  `landlord_default_accounts` sync — both happen from the same `accounts[]`
  payload.

## Document-template integration

The invoice `payment_details` block reads a lease's facility's `collection`/
`both` accounts and expands each into one row per lease component it covers
(an account with `lease_components: [3, 7]` prints two rows — one for Rent, one
for Water; an account with `lease_components: null` prints a single "All
charges" row). See `docs/document-templates.md` §4 for the full payload shape.
