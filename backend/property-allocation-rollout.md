# Property allocation enforcement — production rollout

**Audience: whoever deploys this.** Read the order section before deploying. There is one step that
must happen between the migration and the release, and skipping it locks people out.

## What changes for users

Property Management resources become visible only for the properties a user is allocated to — a row
in `facility_user`. That covers properties, spaces, leases, applications, LOOs, invoices, credit
notes, receipts, statements, exit notices, bills, expenses, remittances, budgets, contracts, LPOs,
procurement requests, tickets, assets, inventories, utilities, meters, projects, fleet, power, and
every Property Management report except Trial Balance.

There is **no bypass**. A user allocated to nothing sees nothing, whatever permissions or roles they
hold, including a company owner.

## The one thing that will generate support tickets

**On the dev database, 93 of 98 active staff in NW Realite hold no allocation at all.** If
production looks similar, those people will open the Properties page after release and see an empty
list, and get a 403 on every property they try to open.

That is the agreed behaviour — there is deliberately no backfill migration — so the allocations have
to be made deliberately, before the release. The audit command below is how you find who needs one.

## Order of operations

### 1. Deploy the code and run the migration

```bash
php artisan migrate --force
```

One migration: `2026_09_23_100000_add_unique_index_to_facility_user_table`. It deletes duplicate
`(facility_id, user_id)` rows — keeping the earliest of each, so the grant keeps its original
`created_at` — and then adds the unique index that table never had.

It is safe to run before the release is live: the index changes no behaviour on its own.

### 2. Audit who holds no allocation — **before** anyone uses the release

```bash
# every company
php artisan properties:audit-allocations

# one company, by id or slug
php artisan properties:audit-allocations 1

# include staff who already hold allocations, to see the whole picture
php artisan properties:audit-allocations --all-users
```

Read-only. It prints, per company, the property count, the active staff count, and a table of
everyone holding no allocation.

### 3. Allocate them

Two ways, both already in the app:

- **Company users screen** — `PATCH /api/v1/app/{company}/access-management/company-users/{id}`
  with a `facilities` array. This is a **sync**: an id left out is removed.
- **Naming someone to a property role** — property create/update with a `roles` payload. This is
  **additive** and also grants the allocation, so being named Property Manager on one property never
  costs someone access to another.

Re-run the audit until the unallocated count is what you intend it to be. It is fine for it to be
non-zero — that is the feature working — as long as it is a decision rather than a surprise.

### 4. Release

No cache or queue steps are needed for the scoping itself. Standard deploy hygiene still applies:

```bash
php artisan config:cache
php artisan route:cache
php artisan queue:restart
```

`queue:restart` matters because of step 5.

## 5. Two behaviour changes to brief support on

**Procurement steps now refuse to dispatch to nobody.** `User::hasAccessToFacility()` was broken —
it answered yes for anybody holding any allocation anywhere — so company-wide approval roles
dispatched to people with no connection to the property. Now:

- Actor lists shrink. This is the fix, but it is visible.
- A step whose role has no holder allocated to that property raises a **422** naming the role and
  the property, instead of silently sitting at the head of the chain with nobody able to move it.

If a company sees that 422, the fix is to name a holder to that role on that property, or allocate
someone who holds the role.

**Reads return 403 with a distinct message.** A property the user is not allocated to answers
`You are not assigned to this property.` rather than a generic permission error, so the frontend can
tell the two apart.

## What is deliberately *not* restricted

| Not restricted | Why |
|---|---|
| Payment vouchers, settlements, ledgers, payouts | No `facility_id`, and a voucher legitimately spans properties. Company-scoped only. |
| Trial Balance report | Its bank legs have no property link, so filtering only the invoice and bill legs produces a report that does not balance. |
| Suppliers, tenants, landlords | They are users, not property-owned. Always visible. |
| Company reference data | Expense categories and types, facility types, LOO templates, document templates, settings. |
| `update` and `destroy` | Still permission-only. A user with the permission but no allocation cannot *find* a record through the API, but can still act on one if they learn its id another way. Known and accepted. |

## Rolling back

The code is reversible by deploying the previous release; nothing about the scoping is persisted.

The migration's `down()` drops the unique index only — it does not recreate the duplicate rows it
removed, and there is nothing to gain from doing so.

Allocations created during step 3 are ordinary data and survive a rollback, which is what you want:
if you re-deploy later, the work is not lost.
