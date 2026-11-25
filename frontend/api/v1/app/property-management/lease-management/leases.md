# Leases API

Domain: `Property Management > Lease Management`

Base route:

`/api/v1/app/{company}/property-management/lease-management/leases`

## Main Endpoints

- `GET /leases` (list)
- `POST /leases` (create)
- `POST /leases/activation`: create an activation lease with its first invoice. See
  [Activation leases](#activation-leases)
- `POST /leases/merge/preview`, `POST /leases/merge`: merge one tenant's leases into one lease. See
  [Merge leases](#merge-leases)
- `GET /leases/{lease}` (show)
- `PUT/PATCH /leases/{lease}` (update)
- `DELETE /leases/{lease}` (delete)
- `PATCH /leases/{lease}/activate`
- `PATCH /leases/{lease}/suspend`
- `PATCH /leases/{lease}/terminate`
- `POST /leases/{lease}/renew`: start a renewal of a lease that is ending. See
  [Renew a lease](#renew-a-lease)
- `POST /leases/{lease}/generate-invoice-for-next-period`
- `POST /leases/{lease}/letting-fee`: bill the lease's letting (or re-letting) fee. See
  [Generate the letting fee](#generate-the-letting-fee)
- `GET /leases/{lease}/invoice-components`: what a manual invoice on the lease can bill. See
  [Invoices](invoices.md#components-a-manual-invoice-can-bill)

## List Leases

`GET /api/v1/app/{company}/property-management/lease-management/leases`

Supported query params:

- Filters:
  - `filter[facility_id]` — **exact**; comma-separated ids match any. Lease pickers narrow to one
    property with it
  - `filter[user_id]` — **exact**; comma-separated ids match any (several tenants at once)
  - `filter[mergeable]=1` — leases that can be merged **and** have a partner: another lease of the
    same tenant on the same property that can be merged too. See
    [Finding leases to merge](#finding-leases-to-merge)
  - `filter[status]`
  - `filter[lease_type]` — **exact**: `standard` or `activation`
  - `filter[landlord_identifier]` — **exact**. `filter[search]` matches it too
  - `filter[created_at]`
  - `filter[start_at]`
  - `filter[end_at]`
  - `filter[next_due_at]`
  - `filter[billing_cycle]`
- Sort:
  - `sort=user_id,status,created_at,start_at,end_at,next_due_at,billing_cycle`
  - Prefix with `-` for descending (example: `sort=-start_at`)
- Include: not supported
- Select fields: not supported
- Pagination:
  - `per_page` (optional)
  - `page` (optional)

Examples:

- `GET /api/v1/app/12/property-management/lease-management/leases?filter[status]=active&sort=-start_at`
- `GET /api/v1/app/12/property-management/lease-management/leases?filter[user_id]=77&per_page=10&page=2`

## Dates and the term

`start_at`, `end_at` and `next_due_at` come back with `raw` as a **plain date**, `"2026-08-01"`
(row 142). They used to be UTC timestamps (`"2026-07-31T21:00:00.000000Z"` for 1 Aug in Nairobi),
which made two things go wrong:

- **The term showed a month short.** Worked out from those two timestamps, a 6-year lease from
  1 Aug 2026 to 31 Jul 2032 reads as 5 years 11 months 29 days. Show the term from
  `period_in_years` / `period_in_months` - what was entered - not from the dates. `end_at` is the
  term's **last day** (inclusive), so a date difference is a day short of the term anyway.
- **Saving moved the dates.** Sending the old `raw` back as `start_at` was read as 31 July, so the
  end landed on 30 July. The server now reads any date it is sent as the day it means in Nairobi,
  so a timestamp no longer shifts it - but send `YYYY-MM-DD`.

`end_at` is never accepted: it is start + term - 1 day. When the month the term reaches has no
such day, it is that month's last day instead: a month from 31 Jan ends 28 Feb (29 Feb in a leap
year), a year from 29 Feb ends 28 Feb, a month from 31 Mar ends 30 Apr. A lease application's
`proposed_end_date` follows the same rule. On an **update** it is worked out again
only when `start_at`, `period_in_years` or `period_in_months` actually changes; any other edit keeps
the end the lease has. Leases imported from EPMAS carry EPMAS's end date and a term counted a month
short (6 years stored as 5 years 11 months), or run part-months (15 Jun to 13 Jun), so re-deriving
the end on every save shortened them.

The same applies to management contracts (`start_at`, `end_at`) and supplier contracts
(`start_at`, `end_at`).

## Create Lease

`POST /api/v1/app/{company}/property-management/lease-management/leases`

Request body:

| Field | Required | Type | Allowed Values / Notes |
|---|---|---|---|
| `user_id` | Yes | integer | Must exist in `users.id` |
| `start_at` | Yes | date (`YYYY-MM-DD`) | - |
| `period_in_years` | Yes | integer | - |
| `facility_id` | Yes | integer | Must exist in `facilities.id` |
| `period_in_months` | Yes | integer | - |
| `billing_cycle` | Yes | string | `biennial`, `annually`, `biannual`, `quarterly`, `monthly`, `weekly`, `daily`, `hourly` |
| `next_due_at` | Yes | date (`YYYY-MM-DD`) | - |
| `status` | No | string | `pending`, `active`, `suspended`, `terminated`, `rejected`. Not worth sending on create — see approval below |
| `currency_id` | No | integer | Must exist in `currencies.id` (defaults to `1` when omitted) |
| `landlord_identifier` | No | string or number, nullable | Up to 255 characters. What the landlord calls this tenant in their own system. **Not unique**: several leases may share one. A number is stored as text (`4512` comes back as `"4512"`); blank is `null`. On an update, omitting it leaves the stored value alone and `null` clears it |

Example request:

```json
{
  "user_id": 77,
  "start_at": "2026-01-01",
  "period_in_years": 1,
  "facility_id": 14,
  "period_in_months": 0,
  "billing_cycle": "monthly",
  "next_due_at": "2026-03-01",
  "status": "active",
  "currency_id": 1,
  "landlord_identifier": "TNT-0451"
}
```

Example response:

```json
{
  "message": "Lease created successfully",
  "lease": {
    "id": 101,
    "user": { "id": 77, "name": "Jane Tenant" },
    "facility": { "id": 14, "name": "Parkview Residency" },
    "billing_cycle": "monthly",
    "landlord_identifier": "TNT-0451",
    "status": { "value": "active", "color": "success" },
    "start_at": { "raw": "2026-01-01", "formatted": "01 Jan, 2026" },
    "next_due_at": { "raw": "2026-03-01", "formatted": "01 Mar, 2026" }
  }
}
```


### Landlord identifier

`landlord_identifier` is returned on the lease list rows and on `GET /leases/{lease}` (and on the
landlord portal's leases). It is printed on the tenant statement: on the PDF cover as a
**Landlord ID** line under **Lease ID**, and in the Excel export beside the lease id in the sheet
heading (`Tenant Statement - Lease ID 181, Landlord ID TNT-0451`). The tenant reports that have a
Lease column carry a **Landlord ID** column (`landlord_identifier`) right after it.

A lease promoted from a Letter of Offer starts with none; set it with an edit.

## Creation approval

A new lease goes through an approval chain before it becomes active, when the company has a
template configured for it.

| Status | Meaning |
| --- | --- |
| `pending` | Raised, waiting on its approval step. Not yet a working lease. |
| `active` | Approved, or created when no approval template is configured. |
| `rejected` | Refused. Terminal. |
| `suspended`, `terminated` | As before, set by hand later in the lease's life. |

**Do not send a status when creating.** The server decides: it writes `pending` and opens the
chain when a template applies, and leaves the lease `active` when none does. A status in the
payload is ignored for this purpose.

**A pending lease is deliberately not a working lease.** It is not invoiced, its space is not
marked occupied, and it does not appear in the dashboard counts or the tenant reports until it is
approved. If a lease seems to have vanished after creation, check whether it is awaiting approval
rather than assuming it failed to save.

**Approval and rejection run through the usual endpoints** — see
[approvals](../../access-management/approvals.md). The lease exposes `approval_steps` like any
other approvable record, so the same screens work for it.

**A lease promoted from a Letter of Offer skips this.** The Loo was approved on the way in, and
asking for a second approval of the same agreement would stall every promotion. Those leases are
created `active` with no chain.

**Nothing changes until a template exists.** With no active template for `Lease`, leases are
created `active` exactly as before, so this can be deployed ahead of the configuration.

## Activation leases

An activation lease is a short brand activation station, usually in a parking area. It has **no
spaces, no term and no recurring billing**. Its charges are invoices raised by hand, and it has
invoices, credit notes, receipts and a statement like any other lease. It is started from the
Leases page ("Create Activation Lease") by anyone with `create-lease`.

Every lease has a `lease_type`:

| `lease_type.value` | Meaning |
|---|---|
| `standard` | A lease of spaces on a term, billed on its cycle. Every lease created through `POST /leases` |
| `activation` | No spaces, term or billing cycle. Created through `POST /leases/activation` only |

On an activation lease:

- `status` is `active` straight away. There is no lease approval; its invoices still go through
  invoice approval like any invoice raised by hand.
- `start_at` is the day it was created, and `end_at`, `next_due_at` and `billing_cycle` are `null`.
- **It ends when it is terminated** (`PATCH /leases/{lease}/terminate`, `terminate-lease`): status
  `terminated`, `terminated_at` set. Nothing ends it on its own. A terminated lease still takes
  invoices, credit notes and receipts, as any terminated lease does.
- **It cannot be suspended:** `permissions.suspend` is `false`, so hide **Suspend**, and `PATCH
  /leases/{lease}/suspend` answers **403** ("An activation lease cannot be suspended. Terminate it
  instead."). It has no spaces or billing to pause.
- **Money on the lease page:** `total_billed` (everything billed to date), `total_collected`
  (everything collected) and `current_arrears` (the difference) come back on `GET /leases/{lease}`
  and on the list rows, for every lease type. The line-by-line history is the tenant statement.
- `description` says what and where it is (e.g. "Activation station, basement parking B2").
- No letting fee, no deposits, no escalations. It is never billed by the billing cycle and never
  appears on the Unbilled Leases report.
- `lease_items` holds one item with `facility_space` `null`: the lines its invoices bill, all at a
  cost of 0. It is bookkeeping, not something to show; hide spaces and items for this type.

### Create Activation Lease

`POST /api/v1/app/{company}/property-management/lease-management/leases/activation`

Creates the lease and its first invoice in one request. Needs `create-lease` and
`create-facility-invoice`.

| Field | Required | Type | Notes |
|---|---|---|---|
| `user_id` | Yes, unless `user` is sent | integer | The tenant (`users.id`) |
| `user` | No | object | A tenant added inline, exactly as on Create Lease: `name`, `email`, `phone` (required), `has_vat`, `id_tax_pin` (required when `has_vat` is true). The new id is used as `user_id` |
| `facility_id` | Yes | integer | The property |
| `description` | No | string, up to 2000 | Printed as the first invoice's notes; `"Activation lease"` when empty |
| `currency_id` | No | integer | Defaults to the property's reporting currency |
| `due_at` | No | date (`YYYY-MM-DD`) | The first invoice's due date; today when omitted |
| `items` | Yes, at least one | array | The first invoice's lines |
| `items[].lease_component_id` | Yes | integer | Any active component, Rent, Service Charge and Deposit included (the same list as `GET /leases/{lease}/invoice-components`) |
| `items[].amount` | Yes | number, > 0 | Excluding tax |
| `items[].tax_id` | Yes | integer | |
| `items[].notes` | No | string, up to 255 | Printed on the line; the component's name when empty |

**Refused with a 422**, each under its own key with "An activation lease has no spaces, term or
billing cycle.": `spaces`, `lease_items`, `facility_space_id`, `start_at`, `end_at`,
`period_in_years`, `period_in_months`, `billing_cycle`, `next_due_at`.

Example request:

```json
{
  "user_id": 77,
  "facility_id": 14,
  "description": "Activation station, basement parking B2",
  "items": [
    { "lease_component_id": 1, "amount": 85000, "tax_id": 1, "notes": "November 2026" },
    { "lease_component_id": 2, "amount": 12500, "tax_id": 1, "notes": "November 2026" },
    { "lease_component_id": 11, "amount": 170000, "tax_id": 3, "notes": "2 months' rent" }
  ]
}
```

Example response (abridged):

```json
{
  "message": "Activation lease created successfully",
  "lease": {
    "id": 412,
    "user": { "id": 77, "name": "Wanjiru Kamau" },
    "facility": { "id": 14, "name": "ACK Gardens" },
    "lease_type": { "value": "activation", "color": "info" },
    "description": "Activation station, basement parking B2",
    "billing_cycle": null,
    "status": { "value": "active", "color": "success" },
    "start_at": { "raw": "2026-11-11", "formatted": "11 Nov, 2026" },
    "end_at": { "raw": null, "formatted": null },
    "next_due_at": { "raw": null, "formatted": null }
  },
  "invoice": {
    "id": 9031,
    "lease_id": 412,
    "notes": "Activation station, basement parking B2",
    "invoice_items": [ "...one per line..." ]
  }
}
```

### After it is created

- **More invoices:** the normal Create Invoice screen (`POST /invoices` with
  `items[].lease_component_id`). Any active component can be billed on an activation lease; see
  [Invoices](invoices.md#components-a-manual-invoice-can-bill).
- **Credit notes, receipts, statement:** as on any lease.
- **Refused on an activation lease** (422): adding a space or lease item (`POST
  /leases/{lease}/items`, error on `facility_space_id`), adding or pricing a lease item component
  (error on `lease_component_id`), a deposit (`POST /leases/{lease}/deposits`) or an escalation
  (`POST /leases/{lease}/escalations`), both with the error on `lease_id`, and the lease form
  (`PUT/PATCH /leases/{lease}`, error on `lease_type`).
- **Billing and Collections** lists it only for a period it billed, collected or owes in. See
  [Billings and Collections](../reports/billings-and-collections.md).

## Merge leases

Merges other leases of one tenant on one property **into a chosen lease** (the **host**), which
keeps its id, history and escalations. Typical case: an
import raised one lease per unit, so a tenant with ten units has ten leases.

Flow on the Leases list:

1. Select the leases, then **Merge**. Opens a full-screen side panel, like Pay Bills.
2. Choose the lease to merge the others into (the **host**).
3. The form shows the term and billing, prefilled from the host. Preview as it changes.
4. Confirm. The host takes that term and billing, goes back to **pending** and through lease
   approval again, the way a renewal does.

Permission: `merge-lease`, and the caller must be allocated to the property. It is listed in
`me/permissions`, so gate the bulk **Merge** button on it. `permissions.merge` on a lease's detail
says whether that lease can be selected (list rows do not carry `permissions`).

### Finding leases to merge

Two additions to the lease list make the picker one step:

- **`filter[mergeable]=1`**: only leases that can go into a merge (active, standard, no exit
  notice, no renewal in progress, not already being merged) **and** have something to merge with:
  another such lease of the same tenant on the same property, in a compatible currency. The partner
  is checked across all leases, whatever else the list is filtered to or paged at.
- **`sort=user_id`**: by tenant, then property, then lease, so each tenant's leases on a property
  sit together (`sort=-user_id` reverses the tenant order only).

It combines with the other filters (all are ANDed):

| Filters | Shows |
|---|---|
| `mergeable` alone | Every tenant's duplicate leases in the company |
| `mergeable` + `facility_id` | Every tenant with duplicates on that property |
| `mergeable` + `user_id` (one or several, comma-separated) | Those tenants' mergeable leases, on every property. Group the rows by property: a selection across properties is refused (`different_property`) |
| `mergeable` + `facility_id` + `user_id` | That tenant's mergeable leases on that property, when there are at least two |
| `mergeable` + `status` other than `active`, or `lease_type=activation` | Nothing |

A tenant with one active and one suspended lease on a property shows nothing: the suspended lease
cannot be merged, so the active one has no partner. Use a large `per_page` in the picker so a
tenant's set is not split across pages. The preview and the merge still check everything, so a
lease that changed in between is refused with its `code`.

Show the **Merge** bulk action when `merge-lease` is in `GET /api/v1/app/{company}/me/permissions`.

### What the merge does

- The host keeps its id, tenant, property, tax, landlord identifier, history and escalations. It
  takes the term (`start_at`, `period_in_years`, `period_in_months`) and billing (`billing_cycle`,
  `next_due_at`) from the form, and the form's `description` when one is sent. A host with no
  currency set takes the others'.
- **While it is pending, nothing moves.** The host is `pending`, so it is not billed until approved
  (a due date that passes meanwhile is billed once it is approved), like a lease being renewed. The
  other leases keep running: they are billed, take receipts and show on reports as before. The
  host's page shows the live preview under `merge.preview`, and what it was under
  `merge.previous`.
- **On approval**, everything on the other leases moves into the host: spaces and their
  components, invoices, credit notes, bills, billings, collections, receipt allocations, deposits,
  opening balances, statement lines, utility meters and their utility bills, exit notices,
  documents, and notifications (their links open the host). The host's "utility billed" date is
  kept only when every lease was billed for utilities this month; otherwise it is cleared, so the
  meters not billed yet are billed on the host. Meter readings already billed are not billed again.
  Then each of the other leases is **deleted**, one by one, so each deletion shows in the audit
  log. The host stays. The other leases' own audit history and approval records stay with them.
- **Escalations:** the host's stay, and can be edited like on any lease. The other leases'
  escalations are deleted.
- **On rejection**, the host goes back to exactly what it was (active, with its old term and
  billing), and the other leases are untouched. They can be merged again.
- Where no approval applies (no lease approval set up, or the caller's role bypasses it), the
  merge happens straight away: the host is active again and already holds everything.
- The tenant is **not** sent the "your lease is active" email for a merge, and no letting fee is
  billed: the spaces are not a new letting.

### Which leases can be merged

Refused with a **422**. Every refusal below carries a stable **`code`** beside the message, so it
can be shown in any language, and **`lease_id`** when one lease is the reason (to highlight its row):

```json
{
  "message": "Lease #92 has a renewal in progress.",
  "code": "renewal_in_progress",
  "lease_id": 92,
  "errors": { "lease_ids": ["Lease #92 has a renewal in progress."] }
}
```

| `code` | Key | `lease_id` | When |
|---|---|---|---|
| `not_found` | `lease_ids` | | A lease is not found, or is in another company |
| `not_allocated` | `lease_ids` | Yes | The caller is not assigned to the lease's property |
| `not_active` | `lease_ids` | Yes | The lease is pending, suspended, terminated or rejected |
| `activation_lease` | `lease_ids` | Yes | An activation lease |
| `renewal_in_progress` | `lease_ids` | Yes | The lease has a renewal in progress |
| `exit_notice` | `lease_ids` | Yes | The lease has an exit notice that was not rejected |
| `already_merging` | `lease_ids` | Yes | The lease is already being merged into another, or is a host with a merge waiting for approval |
| `different_tenant` | `lease_ids` | | The leases belong to different tenants |
| `different_property` | `lease_ids` | | The leases are on different properties |
| `different_currency` | `lease_ids` | | The leases are in different currencies. A lease with no currency set counts as the others' |
| `host_not_selected` | `host_lease_id` | | The host is not one of the selected leases |
| `term_too_short` | `period_in_months` | | The term is 0 years 0 months (merge only) |
| `billing_warnings` | `next_due_at` | | A lease is billed past `next_due_at` and `accept_billing_warnings` was not sent (merge only). The body carries `warnings`, see [Merge](#merge) |

Plain form errors (fewer than 2 or more than 100 leases, a missing or malformed field) are the usual
422 with `errors` and **no `code`**. A caller without `merge-lease` gets **403**.

The same checks run again when the host is approved, since the leases keep running in
between. If one no longer qualifies (say it was terminated meanwhile), **the approval is refused
with a 422**, with the same `code` and `lease_id`, and nothing moves. A lease that was deleted
meanwhile answers `lease_missing`. The message ends "This merge cannot go ahead. Reject it and
merge again."

`permissions.merge` on a lease is `false` for the same reasons. The preview's `warnings` carry
their own codes (`billed_ahead`, `unbilled_gap`), see below.

### Preview

`POST /api/v1/app/{company}/property-management/lease-management/leases/merge/preview`

Writes nothing. Call it when the host or the next due date changes.

| Field | Required | Type | Notes |
|---|---|---|---|
| `lease_ids` | Yes | integer[] | 2 to 100 |
| `host_lease_id` | Yes | integer | One of `lease_ids` |
| `next_due_at` | No | date | The form's next due date. The host's when not sent |

Response:

```json
{
  "data": {
    "preview": {
      "host_lease_id": 75,
      "tenant": { "id": 130, "name": "Wanjiru Kamau" },
      "facility": { "id": 61, "name": "ACK Gardens" },
      "currency": { "id": 1, "code": "KES", "name": "Kenyan Shilling" },
      "next_due_at": "2026-12-01",
      "suggested_next_due_at": "2027-03-01",
      "leases": [
        { "id": 75, "is_host": true, "start_at": "2026-01-01", "end_at": "2028-12-31", "billing_cycle": "monthly",
          "next_due_at": "2026-12-01", "per_month": 27500, "deposits": 0, "opening_balance": 0, "arrears": 0 },
        { "id": 78, "is_host": false, "start_at": "2026-01-01", "end_at": "2028-12-31", "billing_cycle": "monthly",
          "next_due_at": "2027-03-01", "per_month": 32500, "deposits": 60000, "opening_balance": 0, "arrears": 0 }
      ],
      "spaces": [
        { "lease_id": 75, "lease_item_id": 301, "facility_space": { "id": 9001, "name": "A1" }, "per_month": 27500,
          "components": [
            { "id": 811, "lease_component": { "id": 1, "name": "Rent" }, "cost_per_month": 25000 },
            { "id": 812, "lease_component": { "id": 2, "name": "Service Charge" }, "cost_per_month": 2500 }
          ] }
      ],
      "per_month": {
        "components": [
          { "lease_component": { "id": 1, "name": "Rent" }, "amount": 55000 },
          { "lease_component": { "id": 2, "name": "Service Charge" }, "amount": 5000 }
        ],
        "total": 60000
      },
      "deposits": {
        "items": [ { "id": 41, "lease_id": 78, "lease_component": { "id": 1, "name": "Rent" }, "amount": 60000, "billed": true, "refunded": false } ],
        "total": 60000
      },
      "opening_balances": { "items": [], "total": 0 },
      "escalations": [
        { "id": 17, "lease_component": { "id": 1, "name": "Rent" }, "start_at": "2026-01-01", "end_at": null,
          "cycle": "years", "period": 1, "rate": 5, "next_due": "2027-01-01" }
      ],
      "escalations_dropped": 1,
      "meters": {
        "items": [
          { "id": 12, "meter_number": "KPLC-0042", "name": "A1 electricity", "utility": { "id": 5, "name": "Electricity" },
            "status": { "value": "active", "color": "success", "label": "Active" }, "lease_ids": [75, 78], "on_host": true, "shared_with_lease_ids": [] },
          { "id": 13, "meter_number": "NCWSC-0107", "name": "A2 water", "utility": { "id": 6, "name": "Water" },
            "status": { "value": "active", "color": "success", "label": "Active" }, "lease_ids": [78], "on_host": false, "shared_with_lease_ids": [91] }
        ],
        "total": 2
      },
      "utility_billing": {
        "billed_this_month": false,
        "unbilled_lease_ids": [78],
        "message": "Not billed for utilities this month: #78. After the merge lease #75 is billed for utilities this month, charging only the readings not billed yet."
      },
      "arrears": 0,
      "warnings": [
        { "code": "billed_ahead", "lease_id": 78,
          "message": "Lease #78 is billed up to 28 Feb, 2027. Billing from 01 Dec, 2026 after the merge bills its spaces again for that period." }
      ]
    }
  }
}
```

- `spaces` is every space on the selected leases, with the lease it comes from. `per_month` sums
  them by component and in total: what the tenant will pay a month on the host after the merge.
- `deposits.total` and `leases[].deposits` count deposits still held (not refunded).
- `escalations` are the host's, which it keeps; `escalations_dropped` is how many
  the other leases have, which are deleted on approval.
- `meters` is every utility meter the leases read, listed once. The host reads all of them after
  the merge. `lease_ids` are the selected leases that read the meter, `on_host` says whether the
  host already reads it, `status` is the meter's badge as on the meter list, and `shared_with_lease_ids` are leases outside the merge that also read it
  and keep reading it. Show them as a table (meter number, utility, from lease, shared with).
- `utility_billing` says how this month's utility billing carries over. `billed_this_month` is
  `true` only when every lease was billed for utilities this month; the host then counts as billed.
  Otherwise the host is billed for utilities this month, charging only readings not billed yet, and
  `unbilled_lease_ids` lists the leases not billed. Show `message` under the meters table. It is
  `null` when no lease has a meter.
- `suggested_next_due_at` is the latest next due date among the leases. **Prefill the form's
  next due date with it** (not the host's): from it no lease's spaces are billed twice. Leases
  behind it show `unbilled_gap`; fix those with **Generate next period** on that lease.
- `warnings` compare each lease's next due date with the one in the form (the host's after the
  merge), the host's own included. Show them before
  confirming; they do not block:

| `code` | Meaning |
|---|---|
| `billed_ahead` | The lease is billed past the form's next due date. Billing from that date bills its spaces again for the overlap |
| `unbilled_gap` | The lease is billed only up to before the form's next due date. The gap in between is never billed. Fix: **Generate next period** on that lease until it reaches the date, then merge |

### Merge

`POST /api/v1/app/{company}/property-management/lease-management/leases/merge`

| Field | Required | Type | Notes |
|---|---|---|---|
| `lease_ids` | Yes | integer[] | 2 to 100 |
| `host_lease_id` | Yes | integer | One of `lease_ids` |
| `start_at` | Yes | date | Prefill with the host's |
| `period_in_years` | Yes | integer, 0–99 | Prefill with the host's |
| `period_in_months` | Yes | integer, 0–11 | Prefill with the host's. The term must be at least a month |
| `billing_cycle` | Yes | string | As on Create Lease. Prefill with the host's |
| `next_due_at` | Yes | date | Prefill with the host's. See the preview's `warnings` |
| `description` | No | string, up to 2000 | The host's when not sent |
| `accept_billing_warnings` | No | boolean | `true` to go ahead although a lease is billed past `next_due_at` (`billed_ahead`). Default `false` |

`end_at` is worked out from the start and term, as on Create Lease.

**Billed twice is refused unless accepted.** When a lease is billed past `next_due_at`, its spaces
would be billed again for the overlap. Without `accept_billing_warnings: true` that answers **422**:

```json
{
  "message": "Some spaces would be billed twice from this next due date. Choose a later date, or confirm to go ahead.",
  "code": "billing_warnings",
  "warnings": [
    { "code": "billed_ahead", "lease_id": 78, "message": "Lease #78 is billed up to 28 Feb, 2027. …" }
  ],
  "errors": { "next_due_at": ["Some spaces would be billed twice from this next due date. …"] }
}
```

Show the warnings with a checkbox ("I understand these spaces will be billed again for the
overlap") and resend with `accept_billing_warnings: true`, or move the date to
`suggested_next_due_at`. `unbilled_gap` alone needs no confirmation: nothing is billed twice.

Response: the **host**, `pending` (`"message": "Leases merged."` and `status: active` with
`merge.status: completed` when no approval applied):

```json
{
  "data": {
    "message": "Merge raised for approval.",
    "lease": {
      "id": 75,
      "status": { "value": "pending", "color": "warning" },
      "merge": {
        "id": 3,
        "status": { "value": "pending", "color": "warning" },
        "lease_ids": [78],
        "previous": { "status": "active", "start_at": "2026-01-01", "period_in_years": 3, "period_in_months": 0,
          "end_at": "2028-12-31", "billing_cycle": "monthly", "next_due_at": "2026-12-01", "currency_id": 1, "description": null },
        "completed_at": null,
        "preview": { "…": "same shape as the preview above" }
      }
    }
  }
}
```

### On the lease page

`GET /leases/{lease}` carries, on the detail only:

- `merge`: on a host, its merge waiting for approval, or else its latest one: `{ id, status,
  lease_ids, previous, completed_at, preview }`. `status` is `pending`, `completed` or `rejected`.
  `lease_ids` are the leases being merged in (not the host). While pending, `preview` shows
  everything the host will hold after approval (its own spaces plus the others'), and `previous`
  what the host was before the merge changed its term and billing, which is what it goes back to on
  rejection. Both are `null` once the merge is no longer pending. `merge` is `null` on a lease that
  never took in a merge.
- `merging_into`: on a lease being merged into another, `{ id, lease_id }`, the merge and the host.
  Show "Being merged into lease #75" with a link. `null` otherwise.
- `permissions.merge`: whether this lease can be selected for a merge.

## Update Lease

`PUT/PATCH /api/v1/app/{company}/property-management/lease-management/leases/{lease}`

Use the same payload shape as create.

**A `pending` or `active` lease is edited** (row 138; an active lease is edited the way an addendum
would change it, and the edit applies straight away, with no approval). A `suspended`, `terminated`
or `rejected` lease is refused with `403` `"Only a pending or active lease can be edited."`, whatever
the caller's permissions. A lease returned for changes is still `pending`, so its creator can answer
it, and the edit sends it round its chain again.

- Gate the **Edit** button on `permissions.edit`, not `permissions.update`. `update` still means
  "may work on this lease" — deposits, escalations, opening balances, documents, the exit notice,
  activate/suspend/terminate, generating the next invoice — all of which go on after the lease is
  live, so it stays `true` on a lease that can no longer be edited.
- Who may edit a lease: `update-lease` and the lease's creator, on a pending or active lease; on a
  pending one also whoever it was returned to for changes, and an approver whose current step lends
  editing rights. When that step limits them to
  certain fields, a body carrying any other field is refused with `403`, naming the fields allowed
  and the ones refused.
  Such an approver saves their fields through `PATCH leases/{lease}/step-fields` - see
  [approvals](../../access-management/approvals.md#saving-only-the-steps-fields-step-fields-row-138).
- **Tenant and property** (row 138). Sending the lease's own values is always fine.
  - **Pending lease:** a wrong tenant is corrected in place: nothing is billed yet. The property can be
    changed only once the lease has **no units**, because its units are spaces on the old property.
    With units it answers `422` on `facility_id`: "Remove the lease's units first, then change the
    property and add units on the new one." So remove the units, change the property, then add units
    on the new one.
  - **A property change restarts the approval.** If the lease is waiting on its approval chain, the
    chain starts again from the first step, put in front of the new property's approvers. Steps
    already approved for the old property are cleared. If nobody on the new property holds the first
    step's role, the edit is refused with `422` naming the role and the property.
  - **Active lease:** both are fixed. A different `user_id` or `facility_id` answers `422`: "The tenant /
    property of an active lease cannot be changed." Its invoices, receipts, deposits and statements
    belong to that tenant and property.
- The start date and term stay editable - the end
  date is worked out from them on save - but `next_due_at` and any escalation dates are saved as sent,
  so move them in the form when the start date moves.
- **The edit never changes the status.** A `status` in the body is ignored; status moves only
  through approval and the status actions below. (Before row 138 the payload's default `active` was
  written, so editing a pending lease activated it behind its approval chain.)
- With no lease approval template, and for leases promoted from a Letter of Offer, a lease is
  created `active`, and is edited as an active lease.

## Lease Status Actions

No request body required:

- Activate: `PATCH /api/v1/app/{company}/property-management/lease-management/leases/{lease}/activate`
- Suspend: `PATCH /api/v1/app/{company}/property-management/lease-management/leases/{lease}/suspend`
- Terminate: `PATCH /api/v1/app/{company}/property-management/lease-management/leases/{lease}/terminate`

An activation lease cannot be suspended (**403**); see [Activation leases](#activation-leases).

Response shape:

```json
{
  "message": "Lease suspended successfully",
  "lease": {
    "id": 101,
    "status": { "value": "suspended", "color": "secondary" }
  }
}
```

## Renew a lease

`POST /api/v1/app/{company}/property-management/lease-management/leases/{lease}/renew`

Starts a renewal of a lease that is ending. It creates a **renewal lease application**
(`is_renewal: true`, `lease_id`) on the tenant's behalf and puts it in front of the
reviewer, exactly as if the tenant had applied from their portal. From there the ordinary
application flow applies: review → approve → generate the `renewal` offer → send → sign.
The last step, `POST loos/{loo}/promote` on the signed renewal offer, applies it to **this same
lease**: new term and rates, back to `pending` for the lease approval chain, and the deposit
difference billed (top-up) or moved to a receipt against the arrears (surplus). See
[Renewals and addenda](./loo/loo.md#renewals-and-addenda).

**Who.** Anyone who can view the lease: `view-lease` on an allocated property. There is no
separate renew permission.

**When.** The lease must be `active` with an `end_at` no more than 12 months away. A lease
whose `end_at` has already passed but is still `active` qualifies. Refused while the lease
already has a renewal in progress: a renewal application in `pending` or `review`, an
`approved` one whose offer has not been spent, or a live offer drafted straight off the
lease through `POST leases/{lease}/loos`. An offer already applied to the lease counts as
finished, so once a renewal is approved the lease can be renewed again when its new end date
comes within 12 months.

Request body. Every field is optional:

| Field | Type | Default |
|---|---|---|
| `proposed_start_date` | date | the day after `end_at` |
| `proposed_period_in_years` | integer `0`–`99` | the lease's `period_in_years` |
| `proposed_period_in_months` | integer `0`–`11` | the lease's `period_in_months` |

A term of zero years and zero months is refused with `422` on `proposed_period_in_years`.

What the application is pre-filled with:

- **From the lease:** the tenant (`user_id`), the property and its type, the proposed start
  and term above, and `billing_cycle` where the lease's cycle is one an application accepts.
- **Spaces:** every lease item becomes an allocated application space with its priced
  components (`cost_per_space_unit`, `amount`, `tax_id`) copied across. Staff may reprice or
  change them while the application is open, as on any application.
- **Applicant profile and guarantors:** copied from the tenant's *source application*. That
  is the application whose offer was promoted into this lease, else the tenant's latest
  application on this property, else their latest application anywhere. Guarantor documents
  and verification results are not copied. With no application on record at all, the profile
  falls back to the tenant's account (name, email, phone, address, postal address, tax PIN)
  and the property's city and country; anything still unknown is written blank and is left
  for staff to complete through the [staff edit](lease-applications.md#staff-edit).
- **Not copied:** lease escalations (add them on the application), documents and vetting.

The tenant is told the application was received, the same notification they get when they
apply themselves, so they can read and complete it from their portal.

Response `201`:

```json
{ "data": {
  "message": "Renewal application created successfully.",
  "application": {
    "id": 913,
    "is_renewal": true,
    "lease_id": 101,
    "status": { "value": "pending", "color": "secondary" },
    "...": "..."
  }
} }
```

`application` is the full staff application resource, the same shape as
`GET lease-applications/{application}` (see [Lease Applications](lease-applications.md)),
with its spaces, guarantors and `lease` loaded.

Errors: `403` carrying the reason when the lease is not renewable or the caller is not
allocated to its property; `422` keyed `lease` if another renewal was started in the
meantime, or if the property (and, failing that, the tenant) has no city or country on
record, since an application cannot be written without them.

The refusal reasons, word for word: *Only an active lease can be renewed.* · *This lease
has no end date to renew from.* · *A lease can be renewed only within 12 months of its end
date.* · *This lease already has a renewal application in progress.* · *This lease already
has an offer in progress.* The zero-term `422` reads *A renewal needs a term of at least one
month.*

### Renewal state on the lease

`GET leases/{lease}` carries:

```json
{
  "permissions": {
    "view": true, "update": true, "delete": false,
    "activate": false, "terminate": true, "suspend": true,
    "renew": false
  },
  "open_renewal_application": { "id": 913, "status": "pending" }
}
```

- `permissions.renew` is `true` only when the call above would succeed: the caller may view
  the lease, it is active and ending within 12 months, and nothing is renewing it already.
  Show the **Renew** button on it, and nothing else.
- `open_renewal_application` is the renewal application in progress (`{ id, status }`), or
  `null`. When `renew` is `false` because of it, link to that application instead. It is on
  the detail only, not on the list.

The other `permissions` keys are the lease's own abilities (`view`, `update`, `edit`,
`delete`, `activate`, `terminate`, `suspend`, `billLettingFee`). `edit` (row 138) is whether the lease record
itself may be edited — see [Update Lease](#update-lease); the rest were always there.

## Generate Invoice For Next Period

`POST /api/v1/app/{company}/property-management/lease-management/leases/{lease}/generate-invoice-for-next-period`

Purpose:

- Manually generate the upcoming period's invoice for a lease ahead of the scheduled due date
  (`next_due_at`), when operationally necessary. Runs the same generation as the `invoices:generate`
  scheduler.

Request body:

- None.

Eligibility (returns `422` validation error otherwise):

- Lease `status` must be `active`.
- Lease must have a billing schedule (`next_due_at` set).
- The lease must not have ended. When `next_due_at` is after `end_at`, the last period has already
  been billed: `422` with `lease`: `This lease ended on 15 Nov, 2026; there is nothing left to bill.`
- At least one auto-billed component must be priced above zero. Otherwise the lease has nothing to
  bill: `422` with `lease`: `This lease has nothing to bill: every component is priced at zero.`
  No invoice is created and `next_due_at` does not move.
- Unlike the scheduler, the "next due" date check is **skipped** — `next_due_at` may be in the future.

Behavior:

- The invoice `notes` describe the billing **period** being generated for (derived from `next_due_at` and
  the billing cycle), not the date it was generated.
- Every invoice line carries a value. A component priced at zero stays on the lease but is never
  invoiced, so the invoice holds only the priced components.
- `next_due_at` is advanced by exactly one billing cycle from the period start when the invoice is
  created, on the same day of the month, with any time of day dropped: an annual lease billed from
  1 Nov 2026 is next due 1 Nov 2027, and a monthly lease due on the 15th stays on the 15th. (Until
  2026-11-02 the date was also snapped to the start of its year, month or week, so an annual lease
  billed on 1 Nov came out due the following 1 Jan and would have been billed again ten months into
  the period it had just paid for. See [Repairing snapped due dates](#repairing-snapped-due-dates).)
- **Billing stops at the lease's end.** When `end_at` falls inside the period, the invoice covers the
  period only up to `end_at`. The monthly price stays the same, and the `quantity` is the number of
  months actually billed. Whole months count 1 each, and the month the lease ends in counts its
  days over the days in that month. For example, a quarter from 1 Oct with `end_at` 15 Nov bills
  `quantity` `1.5`, with notes `Oct 01 - Nov 15 2026`. `next_due_at` then moves to the day after
  `end_at` (16 Nov), and the lease is not billed again. An escalation inside that shorter period is
  averaged over it, as usual.
- The `invoices:generate` scheduler bills only `active` leases whose `next_due_at` is on or before
  their `end_at` (or that have no end).
- **Billing past the end (holdover, row 181).** When the company's `bill_leases_after_end` setting is
  `yes`, an `active` lease keeps billing after its end:
  - The period it ends inside goes out as **two invoices with the same due date**: one up to `end_at`
    (as above), and one for the rest of the period. A quarter from 1 Oct with `end_at` 15 Nov bills
    `Oct 01 - Nov 15 2026` (`quantity` 1.5) and `Nov 16 - Dec 31 2026` (`quantity` 1.5). `next_due_at`
    then moves to the next period (1 Jan), not the day after the end.
  - Every period after that is billed in full, at the rate in force on `end_at`: escalations dated
    after the end are not applied, on the invoice or by `escalations:process`.
  - It stops when the lease is terminated, or when `end_at` is moved later (it is then an ordinary
    running lease again).
  - Invoices already raised in holdover stay as they are if the lease is later renewed at new rates.
  - This endpoint serves a lease in holdover too, instead of the `ended` 422.

Success response:

- `message`: `Invoice for next period generated successfully.`
- `invoice`: the created `FacilityInvoice`.

### Repairing snapped due dates

Leases the biller advanced before 2026-11-02 may still carry a snapped `next_due_at`: an annual or
biennial lease billed from any month but January sits on 1 Jan, and a lease due mid-month sits on
the 1st. Until that date arrives nothing has been billed twice, and the date can simply be put
back. A console command does it; there is no endpoint:

```bash
php artisan leases:repair-snapped-due-dates                 # report only, nothing changes
php artisan leases:repair-snapped-due-dates --apply         # write the corrected dates
php artisan leases:repair-snapped-due-dates --company=3 --lease=6422 --lease=6423
php artisan leases:repair-snapped-due-dates --billed-from=2026-10-01   # only invoices raised since
```

For every active lease it takes the last invoice the biller raised (by the scheduler or the
"generate invoice for next period" button; manual invoices, deposits, opening balances and utility
bills are ignored) and works out where the due date should be: one cycle after the period that
invoice billed, counted from the lease's billing day. The figures on a biller invoice always cover
whole periods, so the day the queue got to it does not matter: a quarterly lease that started on
the 1st and was invoiced on the 2nd (its due date had drifted) was billed "Oct - Dec", three whole
months, and is next due on 1 Jan, not 2 Jan. For the month-based cycles the billing day is the day
of the month the lease started on; a lease that bills on the 1st, whatever day it started on, is
already right and is passed over.

Only a lease whose current `next_due_at` is exactly what the old snap made of the advanced due
date, and not the right date, is corrected. When the invoice was raised on neither the 1st nor the
lease's start day the date is still written, because left where it is the lease would be billed
again, and flagged so the billing day can be confirmed. A corrected date that falls after the
lease's end is written and flagged too: the period billed past the end may need a credit note.
Only a lease without a start date is listed under "review" and left alone.

The same run lists any two biller invoices on one lease that cover the same days, with the days
billed twice. Those are settled by credit note by hand; the command never raises one.

Running it again finds nothing.

## Generate the letting fee

`POST /api/v1/app/{company}/property-management/lease-management/leases/{lease}/letting-fee`

Bills the landlord the lease's letting fee (a renewal: re-letting fee) as activation would have: a
bill from the company under the property's letting fee service contract, `pending` until its
customer invoice is verified, with its customer invoice. Cancelling that customer invoice cancels the
bill, and the fee can be generated here again. How the fee is worked out, its tax and its notes are in
[Customer invoices](../finance/customer-invoices.md#where-customer-invoices-come-from).

Request body: none.

Gate the **Generate Letting Fee** button on `permissions.billLettingFee` (lease detail only). It is
`true` when the caller is allocated to the property, the lease has **no letting fee bill that is not
cancelled**, and the caller either holds `generate-customer-invoice` or holds
`verify-customer-invoice` and is one of the people who verify the property's letting fee invoices
(the Letter of Offer preparation role on it, see
[Who verifies](../finance/customer-invoices.md#who-verifies); since 2026-11-24). A lease whose
letting fee bill was cancelled can be billed again.

Responses:

| Status | When |
|---|---|
| `200` | `message`: `Letting fee billed.`; `bill`: the bill (`FacilityBillResource`, with `customer_invoice`) |
| `403` | No permission, not allocated, or the lease already has a letting fee bill: `This lease already has a letting fee bill.` |
| `422` | Nothing to bill. `lease`: `No letting fee was billed: the property's active management contract charges none, or the company vendor or letting fee expense type is not set.` |

## Lease Items and Components

### Lease items endpoints

- `GET /leases/{lease}/items`
- `POST /leases/{lease}/items`
- `GET /leases/{lease}/items/{leaseItem}`
- `PUT/PATCH /leases/{lease}/items/{leaseItem}` (currently disabled by backend and returns validation error)
- `DELETE /leases/{lease}/items/{leaseItem}`

Create lease item payload:

| Field | Required | Type | Allowed Values / Notes |
|---|---|---|---|
| `facility_space_id` | Yes | integer | Must exist in `facility_spaces.id`, must belong to lease facility, and must not be occupied |
| `components` | Yes | array | Array of lease item component objects |

Component object inside `components`:

| Field | Required | Type | Allowed Values / Notes |
|---|---|---|---|
| `lease_component_id` | Yes | integer | Must exist in `lease_components.id` |
| `tax_id` | Yes | integer | Must exist in `taxes.id` |
| `cost_per_sqft` | Yes | integer | - |
| `hs_code` | No | string | Optional |

Each lease item's `facility_space` (list, show and the create response) says what kind of space
it is:

| Field | Type | Notes |
|---|---|---|
| `space_type` | string | `leasable`, `landlord` or `common` |
| `is_parking` | boolean | A parking bay, billed its parking fee |
| `is_signage` | boolean | A signage position, billed its signage fee |
| `parking_type` | string \| null | `open` or `closed` on a parking bay, otherwise `null` |

```json
"facility_space": {
  "id": 214,
  "name": "P-12",
  "status": "unavailable",
  "size": 1,
  "space_type": "common",
  "is_parking": true,
  "is_signage": false,
  "parking_type": "closed"
}
```

### Component options for a space

- `GET /leases/{lease}/spaces/{facilitySpace}/component-options`, for editing a lease
- `GET /facilities/{facility}/spaces/{facilitySpace}/component-options`, for Create Lease, before
  there is a lease. Same answer for the same space.

The prefill for the Units / Parking / Signage editor: what a space can be billed, and what the
property's indicative rates suggest charging. Same response as the lease application's
`component-options`.

| Space | Components offered | Prefilled amount |
|---|---|---|
| Unit | Rent, service charge and any other autobilled component | Rent and service charge from the property's indicative rates; others empty |
| Parking bay | Parking fee only | The property's closed or open parking fee, by `parking_type` |
| Signage position | Signage fee only | Empty: there is no indicative signage rate |

```json
{
  "data": {
    "facility_space_id": 34,
    "size": 1,
    "is_lettable": true,
    "components": [
      {
        "lease_component_id": 8,
        "name": "Parking",
        "cost_per_space_unit": 3024.88,
        "amount": 3024.88,
        "tax_id": 2
      }
    ]
  }
}
```

- `amount` is the monthly amount (`size × cost_per_space_unit`). Send it back as
  `cost_per_month`. `null` on both means the property records no rate, and staff enter it.
- `tax_id` is the component's own tax. Use it as the default.
- A space on another property is refused with a `422` on `facility_space_id`.
- Requires `view-lease` (lease version) or `view-facility` (property version), and an allocation
  to the property.

### Lease item components endpoints

- `GET /leases/{lease}/items/{leaseItem}/components`
- `POST /leases/{lease}/items/{leaseItem}/components`
- `GET /leases/{lease}/items/{leaseItem}/components/{leaseItemComponent}`
- `PUT/PATCH /leases/{lease}/items/{leaseItem}/components/{leaseItemComponent}`
- `DELETE /leases/{lease}/items/{leaseItem}/components/{leaseItemComponent}`

`POST` accepts either **one component** at the top level, or **several** under a `components`
array. Both are validated the same way; the bulk form previously accepted anything.

| Field | Required | Type | Allowed Values / Notes |
|---|---|---|---|
| `lease_component_id` | Yes | integer | Must exist in `lease_components.id` |
| `tax_id` | Yes | integer | Must exist in `taxes.id` |
| `cost_per_sqft` | One of the two | number | Rate per space unit. `cost_per_month` becomes `size × cost_per_sqft` |
| `cost_per_month` | One of the two | number | The monthly amount, at least `0`. Stored exactly as sent, and `cost_per_sqft` becomes `cost_per_month ÷ size` |
| `hs_code` | No | string \| null | Optional |

Send **`cost_per_month`** when staff type a monthly amount. It is billed exactly as typed.
Converting it to a rate in the browser and sending `cost_per_sqft` can drift: 52,023 on an 85 sq ft
unit, sent as `612.04`, is stored as 52,023.40.

- If both are sent, `cost_per_sqft` is used and `cost_per_month` is ignored, as before. To bill a
  typed amount, send `cost_per_month` **without** `cost_per_sqft`.
- Sending neither is refused with `422` on `cost_per_sqft`.
- Both are stored to 5 decimals. Cents are kept either way.

The same fields apply to each entry in `components` on `POST /leases/{lease}/items`.

Errors on the bulk form are keyed by position, so the caller can mark the offending row:
`components.0.lease_component_id`, `components.2.tax_id`. An empty `components: []` is refused with
`components: ["At least one component is required."]`.

List component query params:

- Filters: `filter[lease_item_id]`, `filter[lease_component_id]`, `filter[cost_per_sqft]`, `filter[cost_per_month]`, `filter[tax_id]`, `filter[hs_code]`
- Sort: `sort=lease_item_id,lease_component_id,cost_per_sqft,cost_per_month,tax_id,hs_code`
- Include: not supported
- Select fields: supported
  - `fields[lease_item_components]=id,lease_item_id,lease_component_id,cost_per_sqft,cost_per_month,tax_id,hs_code`

Example list with field selection:

`GET /api/v1/app/12/property-management/lease-management/leases/101/items/55/components?fields[lease_item_components]=id,lease_component_id,cost_per_month&sort=-cost_per_month`

### Sending documents with the lease

`POST /leases` and `PUT/PATCH /leases/{lease}` accept an `uploads` array of upload ids. Upload each
file through `POST /api/v1/settings/file-management/uploads` first, then send the ids.

| Field | Required | Type | Allowed Values / Notes |
|---|---|---|---|
| `uploads` | No | array of integers | Upload ids. Each must exist in `uploads.id` |

They become **attachments** on the Documents tab below, downloadable and removable.

**This replaces `signed_agreement_upload_id`, `signed_loo_upload_id` and `other_documents`.** Those
three were never read - the API accepted them, returned success and discarded the files, so a
signed agreement uploaded through the Create Lease wizard was lost. Send `uploads` instead.

On update the list is the **complete set**: a document already on the lease and missing from it is
released. Omitting `uploads` entirely leaves the lease's documents alone, which is what an edit
form that does not show the Documents tab should do. The tab's own Add button appends through the
documents endpoint instead - do not send `uploads` from there.

### Lease documents endpoints

`GET /leases/{lease}` also returns `uploads` - the files attached to the lease itself, in the same
shape every other model with documents uses. On the list, it is opt-in with `?include=uploads`.
That key holds **only** attachments; the offer letter, the signed agreement and the rest come from
the documents endpoint below, which gathers them from the Loo and the application.


- `GET /leases/{lease}/documents`
- `POST /leases/{lease}/documents`
- `DELETE /leases/{lease}/documents/{document}`

Permissions: reading needs `view-lease`, adding and removing need `update-lease`. There are no
separate document permissions.

The list is not paginated - a lease's document count is small and the tab shows all of it.

Document object:

| Field | Type | Notes |
|---|---|---|
| `id` | integer | The **upload** id. This is what `DELETE` takes, and what identifies the file |
| `title` | string \| null | - |
| `file_name` | string | - |
| `type` | string | MIME type |
| `extension` | string | - |
| `size` | integer | Bytes |
| `source_url` | string | Where to download it. Points at the existing upload preview route |
| `source` | object | `{ value, label, color }` - where the document came from, for a chip |
| `context` | string \| null | What it belongs to, where that helps tell two apart: a Loo's reference, or an application document's type |
| `removable` | boolean | Whether `DELETE` will accept it. Only `attachment` is ever `true` |

`source.value` is one of:

| Value | Label | Comes from |
|---|---|---|
| `attachment` | Attachment | A file added to the lease through this endpoint |
| `offer-letter` | Offer letter | The Loo that was promoted into this lease |
| `signed-offer` | Signed offer | The tenant's signature on that Loo |
| `legal-fee-note` | Legal fee note | That Loo's fee note |
| `lease-agreement` | Signed lease agreement | The application the Loo was prepared from |
| `application-document` | Application document | The applicant's supporting documents. **Staff only** |

A lease owns almost none of its own paperwork, so most rows are gathered from the Loo that created
the lease, from the application behind it, and from any renewal or addendum drafted from the lease.
Those are read-only here: removing one would mean unpicking the record it actually belongs to.

#### Adding a document

Upload the file first through `POST /api/v1/settings/file-management/uploads`, then claim it:

`POST /leases/{lease}/documents`

| Field | Required | Type | Allowed Values / Notes |
|---|---|---|---|
| `uploads` | Yes | array | At least one upload id |
| `uploads.*` | Yes | integer | Must exist in `uploads.id` |

Send only the file you just uploaded - the endpoint **appends**. Documents already on the lease are
untouched, so there is no need to post the whole list back.

An upload is refused (`422`, keyed `uploads`) when it already belongs to another record, or when it
was uploaded by somebody else. Re-sending one already on this lease is accepted and changes
nothing.

The response carries the full document list back, so the tab can re-render from it.

#### Removing a document

`DELETE /leases/{lease}/documents/{document}` where `{document}` is the upload id.

The file is **released, not deleted** - it stays in `uploads` and simply stops belonging to the
lease. Only a row whose `removable` is `true` is accepted; anything else returns `422` keyed
`upload`.

## Related Docs

- `docs/frontend/app/property-management/lease-management/lease-deposits.md`
- `docs/frontend/app/property-management/lease-management/lease-escalations.md`
- `docs/frontend/app/property-management/lease-management/lease-opening-balances.md`

## Frontend Error Handling

Apply the shared rules in `docs/frontend/app/README.md`:

- Show backend `message` and field `errors` for `4xx`.
- Show generic fallback message for `5xx`.
