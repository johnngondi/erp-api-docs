# Property allocation scoping — the backend contract

**Audience: backend developers.** This describes how a Property Management record is restricted to
the people allocated to its property. Nothing here is a new API endpoint; it changes what existing
list and read endpoints return.

## The rule

A row in `facility_user` is the **only** key to Property Management visibility.

There is no super admin, company owner or permission bypass. A user allocated to no properties sees
no properties, and no invoices, bills, contracts, receipts, LPOs or credit notes either. This falls
out of the query itself — `whereIn('facility_id', [])` is a false predicate — rather than being a
special case someone has to remember to write.

Allocation is granted two ways, both writing the same pivot:

| How | Where |
|---|---|
| Directly, on a company user | `AssignFacilitiesToUserAction`, from the company-user endpoints |
| Implicitly, by naming someone to a property role | `AssignFacilityRolesAction`, additively |

Because naming a role holder also writes `facility_user`, reading that one table covers role
holders too. Nothing needs to consult `facility_role` to answer "can this user see it".

## What is scoped, and what is not

Scoped: properties, spaces, leases, lease applications, LOOs, invoices, credit notes, receipts,
tenant statements, exit notices, bills, expenses, remittances, budgets, contracts, LPOs,
procurement requests, tickets, assets, inventories, utilities, meters, projects and the facility
operations models.

**Not scoped, deliberately:**

| Not scoped | Why |
|---|---|
| `FacilityPaymentVoucher`, `FacilitySettlement` | No `facility_id`, and one voucher legitimately spans properties. Company-scoped only. |
| `FacilityLedger`, `FacilityLedgerItem`, `FacilityPayout` | No path to a property at all. |
| Trial Balance report | Its bank legs have no property link, so filtering only the invoice and bill legs yields a trial balance that does not balance. |
| Suppliers, tenants, landlords | They are `User` rows, not owned by a property. Always visible. |
| Company reference data | Expense categories and types, facility types, LOO templates, document templates, settings. |

## How a model becomes property-scoped

Add the trait. If its property is a `facility_id` column on its own table, that is the whole change:

```php
use App\Concerns\HasAccessibleToUserScope;

class FacilityBill extends Model
{
    use HasAccessibleToUserScope;
}
```

If the property is somewhere else, say where with `facilityScopePath()`:

```php
class FacilityInvoice extends Model
{
    use HasAccessibleToUserScope;

    public function facilityScopePath(): string
    {
        return 'lease.facility_id';
    }
}
```

### The path forms

| Path | Meaning | Cost |
|---|---|---|
| `facility_id` | A column on the model's own table. The default. | one `IN` |
| `id` | The model **is** the property. Only `Facility`. | one `IN` |
| `facility` | A `Facility` relation on the model itself. | one `EXISTS` |
| `lease.facility_id` | Relation hop, then a column on the related table. | one `EXISTS` |
| `lease.facility` | Relation hop ending on a `Facility` relation. | two `EXISTS` |

The last segment is read as a column when it ends in `_id`, otherwise as a relation. **Prefer the
column form.** Every chained model in this domain reaches a table that already carries
`facility_id`, so there is no reason to pay for the second hop.

The paths in use: `lease.facility_id` (invoices, credit notes, tenant statements, exit notices and
the six lease children), `procurementRequest.facility_id` (LPOs, site visit reports),
`facilityContract.facility_id` (vendor statements), `bill.facility_id` (bill withholdings),
`expense.facility_id` (project expenses), and `id` for `Facility` itself.

### When no path can describe it

Override `scopeAccessibleToUser` directly. `FacilitySpace` is the one case: its property is reachable
only through the `facility_floors.floorable` morph, which resolves
`Facility -> Floor`, `Facility -> Block -> Floor`, `Facility -> Wing -> Floor` and
`Facility -> Block -> Wing -> Floor` to the same property. It delegates to its own
`scopeInFacilities()` rather than growing a second copy of that morph logic.

## Applying it

**Lists** — in the controller's `index`, exactly as the procurement endpoints already did:

```php
$bills = QueryBuilder::for(FacilityBill::query()->accessibleToUser(user()))
    ->allowedFilters([...])
    ->paginate(...);
```

**Reads** — in the model's policy, through the shared trait:

```php
use App\Policies\Concerns\ChecksPropertyAllocation;

class FacilityBillPolicy
{
    use ChecksPropertyAllocation;

    public function view(User $user, FacilityBill $bill): bool
    {
        return $user->hasPermissionTo('view-facility-bill')
            && $this->allocatedTo($user, $bill);
    }
}
```

An unallocated read returns **403**, with a message distinct from a permission failure so the UI can
say "you are not assigned to this property" rather than "you lack permission".

Nested routes come free. Children under `facilities/{facility}` and `leases/{lease}` use Laravel's
`->scoped()` binding, so scoping the parent covers blocks, wings, floors, bank accounts, lease
items, billings, collections, deposits, escalations and opening balances.

## Ownership fallbacks: gate the staff branch, not the whole policy

Several policies let someone read a record because it is *theirs*, not because they hold a staff
permission — a tenant reading their own lease, LOO or ticket, an applicant their own lease
application, a vendor their own bill, a project's own manager. **None of those people hold a
`facility_user` row**, so gating the whole policy on allocation locks every one of them out of
their own data.

The allocation check goes on the permission branch only:

```php
// Staff are allocation bound; the tenant path below is not.
if ($user->hasPermissionTo('view-lease') && $this->allocatedTo($user, $lease)) {
    return true;
}

return $lease instanceof Lease && (int) $lease->user_id === (int) $user->id;
```

The policies carrying such a fallback today: `LeasePolicy`, `LooPolicy`, `LeaseApplicationPolicy`,
`FacilityBillPolicy`, `FacilityTicketPolicy` and `ProjectPolicy`. `FacilityTicketPolicy::view`
gained one in this work — `update()` and `delete()` already had it, and `view()` had never needed
it because it was permission-only.

The same rule applies outside policies. `BuildsVendorStatements` is shared by the app and the
vendor portal, so it takes an optional user and only the app surfaces pass it.

## Pass the record, not the class

`authorize('view', FacilityInvoice::class)` type-checks and reads naturally, but it hands the
policy a `null` record, so **no per-record check runs at all**. Nine `show` methods were written
that way, which is also why several of these policies take a nullable model. Always
`authorize('view', $invoice)`.

## Two things to know before you rely on this

**Writes are not covered.** `update` and `destroy` keep their permission-only checks. A user holding
the permission but no allocation cannot *find* a record through the API, but can still act on one if
they learn its id another way. This is a known and accepted gap, not an oversight — do not read the
`view()` check as covering writes.

**It is opt-in, not a global scope.** A global scope was rejected: `CompanyScoped` early-returns
under `app()->runningInConsole()`, so anything copying that pattern is inert in queue workers,
commands and seeders; a global scope also applies to writes, and would blank the landlord, tenant
and vendor portals, which query these same models with users who hold no allocations. Opt-in means
`grep -rn accessibleToUser app/Http` enumerates exactly which endpoints enforce allocation — the
diff is the audit.

## Reports

Reports do not each filter their own figures. 20 of them resolve their property universe with the
same query and fold every figure from `$facilities->pluck('id')`, so
`App\Services\PropertyManagement\Reports\Support\ReportFacilities` narrows that universe once and
every figure, total and bucket follows. A report run with no authenticated user throws rather than
silently returning the whole portfolio.

The Property Management dashboard (`App\Services\PropertyManagement\Dashboard\PropertyManagementDashboardService`)
is built the same way: it resolves its property universe through `ReportFacilities::scoped()`, then
applies its own Landlord, Property and Team filters, and every card, series and list folds from
that one set of ids. Its "Team" filter is a company department, and a property is on the team
when a member of that department is allocated to it — the same `facility_user` row that decides
visibility, so the filter can never widen what the actor may see.

## Tests

A facility built with `Facility::factory()` bypasses `CreateFacilityAction`, and therefore all
auto-assignment, so it is visible to nobody. Opt in explicitly:

```php
$facility = Facility::factory()->create();

allocate($user, $facility);              // or: Facility::factory()->allocatedTo($user)->create()
```

Both write `facility_user` and refresh the user, because `User::accessible_facilities` is memoized
per instance — a grant made after the accessor was first read is otherwise not seen.

Every enforcement change carries the same three-user test: allocated to one property sees only that
one, allocated to both sees both, allocated to nothing sees nothing and gets 403 on a read. Plus one
asserting a user allocated *only* via a property role still sees the property, which is what proves
the additive grant path works through `facility_user` alone.
