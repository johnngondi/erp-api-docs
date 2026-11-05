# Team Arrears API

Domain: `Property Management > Reports > Teams`

Base route:

`/api/v1/app/{company}/property-management/reports/teams/arrears`

Team Arrears is a three-level drill-down: arrears **per property team**, then **per property of a
team**, then **per debtor of a property**. It is read-only and computed on the fly.

> This is **not** a registry report. It is not listed by `GET /reports`, has no export, and does not
> use the report envelope (`header` / `report` buckets / `summary`) described in
> [the shared reports contract](./README.md). Each endpoint returns plain JSON under `data`.

## Endpoints

| Method | URL | Route name | Returns |
|---|---|---|---|
| GET | `/reports/teams/arrears` | `app.property-management.reports.teams.arrears.index` | every team |
| GET | `/reports/teams/arrears/{propertyTeam}` | `app.property-management.reports.teams.arrears.show` | the team's properties |
| GET | `/reports/teams/arrears/{propertyTeam}/properties/{facility}` | `app.property-management.reports.teams.arrears.property` | the property's debtors |

All three require **`view-team-arrears`** (resource "Team Arrears", tag "Property Management", staff
group). Without it: `403`.

- `show` returns `404` when the team belongs to another company.
- `property` returns `404` when the property is not on the team, or is not in the caller's scope
  (not allocated to the caller, or its status is not in `properties_status`).

## Filters

Flat query parameters (not `filter[...]`). Every filter is optional.

| Param | Type | Default | Notes |
|---|---|---|---|
| `period_from` | date | first day of the current calendar quarter | |
| `period_to` | date | last day of the current calendar quarter | A reversed range is put in order. |
| `properties_status[]` | list of property `Status` values (`active`, `inactive`, `pending`, …) | `['active']` | |
| `leases_status[]` | list of `LeaseStatus` values (`pending`, `active`, `suspended`, `terminated`, `rejected`) | `['active']` | |
| `team_ids[]` | list of property team ids | every team | Teams level only. |

Lists may be sent as `key[]=a&key[]=b`, as a comma-separated string (`key=a,b`) or as a single
value. An unknown status or team id returns `422` with an error per element (for example
`properties_status.0`).

Every response echoes the **effective** filters, defaults applied:

```json
"filters": { "properties_status": ["active"], "leases_status": ["active"], "team_ids": [] }
```

## Scope

- **Properties:** allocated to the caller (the same allocation rule every report uses), of the
  company, on the team (`property_team_facilities`), and with a status in `properties_status`. A
  property belongs to at most one team, so team rows add up to the totals row.
- **Leases:** on those properties, with a status in `leases_status`.

## Figures (Billings & Collections basis)

Per lease, summed over any set of leases (a property, a team, all teams):

| Key | Meaning |
|---|---|
| `bbf` | `lease_billings.total` before `period_from`, less confirmed `lease_collections.total` before `period_from` |
| `billings` | `lease_billings.total` within the period |
| `collections` | confirmed `lease_collections.total` within the period (cancelled collections never count) |
| `expected` | `bbf + billings` |
| `arrears` | `expected - collections` |
| `collections_percent` | `collections / expected × 100`, 2 dp; `null` when `expected` is 0 or less |
| `collections_percent_color` | `success` above 80, `warning` from 50 to 80 inclusive, `danger` below 50, `none` when the percent is `null` |
| `period_collections_percent` | `collections / billings × 100`, 2 dp; `null` when `billings` is 0 or less |
| `period_collections_percent_color` | same bands as `collections_percent_color` |

These are the same figures as the [Billings & Collections](./billings-and-collections.md) balance
c/f for the same leases. They are **not** the [Debtors Listing](./debtors-listing.md) figures,
which read invoice balances.

**Totals rows recompute both percentages from their summed figures.** They never average the rows'
percentages.

**Money is a string with 2 decimals** (`"6300.00"`, no thousands separator). Format it on the
frontend with `currency`. `currency` is the one reporting currency shared by the properties in
scope (`{ id, code, name }`), or `null` when they use different currencies or there are none.

## GET `/reports/teams/arrears`

Lists **every team of the company**, or the teams in `team_ids`, ordered by
`period_collections_percent`, **highest first**. Teams with no period rate (`null`: nothing billed in
the period) come last, and ties are ordered by name, then id. A team with nothing in scope still
appears, with zeros and `properties_count: 0`.

```json
{
  "data": {
    "period": { "from": "2026-07-01", "to": "2026-09-30" },
    "filters": { "properties_status": ["active"], "leases_status": ["active"], "team_ids": [] },
    "currency": { "id": 1, "code": "KES", "name": "Kenya Shilling" },
    "teams": [
      {
        "id": 3,
        "name": "Alpha",
        "emblem_url": "https://…/uploads/…/preview",
        "leader": { "id": 41, "name": "Lena Leader" },
        "landlords_count": 1,
        "properties_count": 2,
        "tenants_count": 3,
        "sqft": 2500,
        "members": {
          "total": 3,
          "by_role": [
            { "role_id": 12, "role": "Property Manager", "count": 2 },
            { "role_id": null, "role": null, "count": 1 }
          ]
        },
        "bbf": "600.00",
        "billings": "6300.00",
        "expected": "6900.00",
        "collections": "4400.00",
        "arrears": "2500.00",
        "collections_percent": 63.77,
        "collections_percent_color": "warning",
        "period_collections_percent": 69.84,
        "period_collections_percent_color": "warning"
      }
    ],
    "totals": {
      "landlords_count": 1,
      "properties_count": 2,
      "tenants_count": 3,
      "sqft": 2500,
      "members_total": 3,
      "bbf": "600.00",
      "billings": "6300.00",
      "expected": "6900.00",
      "collections": "4400.00",
      "arrears": "2500.00",
      "collections_percent": 63.77,
      "collections_percent_color": "warning",
      "period_collections_percent": 69.84,
      "period_collections_percent_color": "warning"
    }
  }
}
```

- `landlords_count`: distinct `facilities.landlord_id` among the team's properties in scope.
- `tenants_count`: distinct tenants (`leases.user_id`) among the leases in scope.
- `sqft`: Total SqFt of the leases in scope, using the same rule as Billings & Collections (lettable
  spaces under the lease, on properties whose type is divided by size; 0 otherwise).
- `members.by_role`: the team's members grouped by the roles they hold **in this company**, ordered
  by role name. A member with two roles counts under both. Members with no role are counted under
  `role: null`, which comes last. `members.total` counts each member once.
- Totals count distinct landlords, tenants and members across all listed teams, so a member of two
  teams is counted once in `members_total`.
- `leader` is `null` when the team has none, and `emblem_url` is `null` without an emblem.

## GET `/reports/teams/arrears/{propertyTeam}`

The team's properties in scope, ordered like the teams: `period_collections_percent` highest first,
`null` last, ties by name, then id.

```json
{
  "data": {
    "team": { "id": 3, "name": "Alpha", "emblem_url": null, "leader": { "id": 41, "name": "Lena Leader" } },
    "period": { "from": "2026-07-01", "to": "2026-09-30" },
    "filters": { "properties_status": ["active"], "leases_status": ["active"], "team_ids": [] },
    "currency": { "id": 1, "code": "KES", "name": "Kenya Shilling" },
    "properties": [
      {
        "id": 18,
        "name": "Alpha One",
        "status": { "value": "active", "color": "success", "label": "Active" },
        "roles": [
          { "id": 301, "role_id": 13, "role": "Agency", "user": null },
          { "id": 300, "role_id": 12, "role": "Property Manager", "user": { "id": 52, "name": "Mo Member" } }
        ],
        "tenants_count": 3,
        "bbf": "600.00",
        "billings": "5300.00",
        "expected": "5900.00",
        "collections": "3500.00",
        "arrears": "2400.00",
        "collections_percent": 59.32,
        "collections_percent_color": "warning",
        "period_collections_percent": 66.04,
        "period_collections_percent_color": "warning"
      }
    ],
    "totals": {
      "properties_count": 2,
      "tenants_count": 3,
      "bbf": "600.00",
      "billings": "6300.00",
      "expected": "6900.00",
      "collections": "4400.00",
      "arrears": "2500.00",
      "collections_percent": 63.77,
      "collections_percent_color": "warning",
      "period_collections_percent": 69.84,
      "period_collections_percent_color": "warning"
    }
  }
}
```

- `roles` has one entry per `facility_role` row of the property (one role held by one person, as
  the property form saves them), ordered by role name, then holder name. `id` is the `facility_role`
  row id. `user` is `null` when the row names nobody. `roles` is `[]` when the property has none.

## GET `/reports/teams/arrears/{propertyTeam}/properties/{facility}`

The property's debtors: its leases in scope with **arrears greater than 0**, ordered by
`debt_beyond_deposit` (highest first), then `arrears` (highest first).

```json
{
  "data": {
    "team": { "id": 3, "name": "Alpha" },
    "property": { "id": 18, "name": "Alpha One" },
    "period": { "from": "2026-07-01", "to": "2026-09-30" },
    "filters": { "properties_status": ["active"], "leases_status": ["active"], "team_ids": [] },
    "currency": { "id": 1, "code": "KES", "name": "Kenya Shilling" },
    "debtors": [
      {
        "lease_id": 905,
        "tenant": { "id": 77, "name": "Tina Tenant" },
        "month_rent": "1392.00",
        "bbf": "600.00",
        "billings": "3000.00",
        "collections": "1500.00",
        "arrears": "2100.00",
        "deposit_held": "1000.00",
        "months_in_arrears": 1.51,
        "debt_beyond_deposit": "1100.00",
        "severity": "danger"
      },
      {
        "lease_id": 911,
        "tenant": { "id": 79, "name": "Tess Tenant" },
        "month_rent": "0.00",
        "bbf": "0.00",
        "billings": "300.00",
        "collections": "0.00",
        "arrears": "300.00",
        "deposit_held": "500.00",
        "months_in_arrears": null,
        "debt_beyond_deposit": "0.00",
        "severity": "warning"
      }
    ],
    "totals": {
      "arrears": "2400.00",
      "deposit_held": "1500.00",
      "months_in_arrears_avg": 1.51,
      "debt_beyond_deposit": "1100.00"
    }
  }
}
```

| Key | Meaning |
|---|---|
| `month_rent` | Monthly rent **including service charge and VAT**: the sum of `cost_per_month × (1 + tax)` over the lease's item components whose lease component is `is_rent` or `is_service_charge`. The tax is the item component's own `tax_id` when it has one, else the lease component's. `taxes.value` is a fraction (0.16 = 16%). |
| `deposit_held` | `lease_deposits.amount` not yet refunded (`refunded_at` and `refunded_by_exit_notice_id` both null) |
| `months_in_arrears` | `arrears / month_rent`, 2 dp; `null` when `month_rent` is 0 |
| `debt_beyond_deposit` | `max(0, arrears - deposit_held)` |
| `severity` | `danger` when `debt_beyond_deposit > 0`, else `warning` (every row has arrears) |

Totals: `arrears`, `deposit_held` and `debt_beyond_deposit` are sums over the debtor rows.
`months_in_arrears_avg` is the mean of the rows' non-null `months_in_arrears` (2 dp), or `null` when
there are none.

Leases with zero or negative arrears are not listed, so the debtors total can be higher than the
property's `arrears` on the team page when some tenants are in credit.

The tenant statement for a row is the existing
`lease-management/tenant-statements` endpoint with `filter[lease_id]` and
`filter[transaction_at][from|to]` set to the period.
