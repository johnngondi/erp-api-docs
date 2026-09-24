# Tenants

A tenant is a `users` row holding the `tenant` user group. There is no `tenants`
table, so the whole surface is group membership over `User`.

Base prefix: `/api/v1/app/{company}`

| Method | URI | Name | Controller |
|---|---|---|---|
| `GET` | `.../users/tenants` | `app.users.tenants.index` | `TenantController@index` |
| `POST` | `.../users/tenants` | `app.users.tenants.store` | `TenantController@store` |
| `GET` | `.../users/tenants/{tenant}` | `app.users.tenants.show` | `TenantController@show` |
| `PUT\|PATCH` | `.../users/tenants/{tenant}` | `app.users.tenants.update` | `TenantController@update` |
| `DELETE` | `.../users/tenants/{tenant}` | `app.users.tenants.destroy` | `TenantController@destroy` |

Registered as a plain `Route::apiResource('tenants', TenantController::class)` in
`routes/Api/v1/app.php`, under the `users.` group.

Frontend spec: `docs/frontend/api/v1/app/users/tenants.md`.

## Authorization

Tenants have no permission set of their own. `TenantController` authorizes
against `LeasePolicy`, because holding the tenant group is only meaningful to
whoever administers leases:

| Action | Gate | Permission |
|---|---|---|
| `index`, `show` | `viewAny` on `Lease::class` | `view-lease` |
| `store` | `create` on `Lease::class` | `create-lease` |
| `update` | `update` on `Lease::class` | `update-lease` |
| `destroy` | `delete` on `Lease::class` | `delete-lease` |

`LeasePolicy::update()` and `::delete()` take a nullable `Lease`, so passing the
class string is safe: `$lease` stays null, `request()->route('lease')` is null on
these routes, and the check collapses to the permission alone. The
creator-fallback branch in those policy methods is unreachable from here.

## Record scoping

`show`, `update` and `destroy` all begin with:

```php
if (!$tenant->hasUserGroup('tenant'))
    abort(404, __('Tenant not found'));
```

`{tenant}` is an implicitly bound `User`, so without this a landlord or vendor id
would resolve and be editable through the tenant surface. The check runs before
the gate so a non-tenant id is a `404` rather than a `403`.

## index

`UsersHelper::tenants()` builds a `QueryBuilder` over the tenant group's `users()`
relation:

- `allowedSorts`: `id`, `name`, `created_at`
- `allowedFilters`: `search` (custom `UserFilter`), `id`, `name`, `email`,
  `phone`, `created_at`
- pagination: `per_page`, default `config('app.query.default_per_page')`

`UserFilter` runs a Scout search across the `toSearchableArray()` keys, and
short-circuits to `whereIn('users.id', …)` when the value is a CSV of digits.

Deliberately no `status` filter, unlike `UsersHelper::vendors()`. Deactivation
removes group membership, so a deactivated tenant leaves the relation entirely
and there is no inactive row to select.

## store

Elevation, not plain creation. `CheckIfUserExistsAction` looks up the email:

- miss — `CreateNewUser` creates the user with a server-generated
  `Str::password(8)` and `is_visible_to_staff = true`
- hit — the existing user is reused untouched

Either way `AttachUserGroupsToUserAction` and `AssignUserGroupRolesAction` then
apply the `tenant` group and its roles. The two paths are distinguished only by
the returned `message`.

Validation lives in `CreateNewUser`, not in a Data class, so the conditional
fields (`name`, `phone`, `terms`, `policy`) are only enforced on the create path.

`portal` is required by `CreateNewUser` but is never a caller input here. The
controller injects it after `$request->all()`, so a spoofed `portal` in the body
is overwritten rather than honoured:

```php
private function tenantPortalId(Company $company): ?int
{
    return UserGroupSettings::tenantGroupId($company)
        ?? UserGroup::query()->where('title', 'tenant')->value('id');
}
```

`UserGroupSettings::tenantGroupId()` reads the company-scoped
`tenant_user_group_id` setting (`SettingSeeder`). The global `settings()` helper
is deliberately not used — it caches by key alone and would leak one company's
group id into another's. The fallback to the `tenant` group covers a company
whose setting was never seeded.

## update

`UpdateTenantData` (Spatie Data v4) over `UpdateTenantAction`. Every field is
`sometimes`, so a partial `PATCH` is a first-class request and an absent key
leaves the column alone.

The controller calls `UpdateTenantData::validateAndCreate()`, not `::from()`.
`from()` only runs the rules when it is handed a `Request`, and the payload has
to be an array here so `tenant_id` can be injected — `from()` would skip
validation entirely and let a duplicate email reach the database as a `500`.

Uniqueness on `email`, `phone` and `tax_pin` has to ignore the tenant's own row.
`rules(ValidationContext $context)` reads `tenant_id` off the payload, which
`TenantController@update` injects from the bound model:

```php
UpdateTenantData::from(array_merge($request->all(), ['tenant_id' => $tenant->id]));
```

`tenant_id` is a validation input only — `UpdateTenantAction` never writes it.

`tax_pin` is `required_if:has_vat,true`, matching `CreateNewUser`.

The action writes only the keys that were present, resolved through
`Optional`, so sending `{"address": null}` clears the column while omitting
`address` preserves it. That distinction is the reason the properties are typed
`string|Optional|null` rather than `?string`.

## destroy

Deactivation, not deletion. `DeactivateTenantAction` mirrors
`DeleteVendorAction`:

```php
$tenant->removeUserGroups(UserGroup::where('title', 'tenant')->get());

if ($tenant->userGroups()->count() == 0) {
    (new DeleteUser())->delete($tenant);
}
```

For a user who holds another group, leases, invoices, receipts and statements are
untouched and keep resolving `user_id` to a live row.

A user left with no groups at all is deleted, on the grounds that they can no
longer sign in as anything — and **that delete is permanent**. `users` has a
`deleted_at` column from its create migration, but the `User` model does not use
the `SoftDeletes` trait, so `DeleteUser::delete()` removes the row. A tenant who
is only ever a tenant holds no other group, so this is the ordinary path rather
than an edge case, which sits awkwardly against the endpoint's "deactivate"
intent. Pinned by `it permanently deletes a tenant left with no other group` in
`tests/Feature/TenantCrudTest.php` so a later change to either the trait or this
branch is a visible test change.

`removeUserGroups()` calls `forgetUserGroupMembership()`, so a `hasUserGroup()`
answer memoized earlier in the same request is not served stale.

Re-activation is `store` with the same email, which takes the elevate path. There
is deliberately no restore route.

## Transactions and errors

All three writes wrap in `DB::beginTransaction()` with a `catch` that rolls back,
calls `log_exception()` and rethrows a generic `Exception` with a `500`:

| Action | Log context | Message |
|---|---|---|
| `store` | `"Error creating tenant"` | `Error creating tenant` |
| `update` | `"Error updating tenant"` | `Error updating tenant` |
| `destroy` | `"Error deactivating tenant."` | `Error deactivating tenant` |

`store` and `update` rethrow `ValidationException` ahead of the generic catch so
a `422` is not flattened into a `500`.

## Resource

`TenantResource` (`App\Http\Resources\V1\App`), mirroring `VendorResource`:
`id`, `name`, `email`, `phone` always; `has_vat`, `tax_pin`, `address`,
`withholds`, `created_at` under `whenHas`; `status` as `Status::get()`; and a
`permissions` block carrying `view-lease` / `update-lease` / `delete-lease` so
the UI can hide row actions.

`store`, `update` and `destroy` return it nested in a `DataResource` alongside
`message`; `index` and `show` return it directly.
