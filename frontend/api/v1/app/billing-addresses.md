# Billing Addresses API

Domain: `Users` and `Property Management > Properties`

A **billing address** says where a party is billed. A party is a tenant, a supplier, a landlord or a
property. Each party can keep several addresses:

- **Default.** Exactly one address per party is the default. Documents print it, and document
  emails go to its email.
- **Shipping.** Any address can be marked `is_shipping`. A property's shipping address is the
  "Deliver To" on an LPO.
- **Always CC.** Any address can be marked `always_cc`. It is copied on every document emailed to the
  party.

Issued documents keep the address they were issued with. Editing an address never changes a document
already sent (see [Locked addresses](#locked-addresses)).

This doc covers the staff app routes and the rules shared by every surface. The portals use the same
body, response and rules on their own routes:

- Supplier portal: [`docs/frontend/api/v1/vendor/billing-addresses.md`](../vendor/billing-addresses.md)
- Tenant portal: [`docs/frontend/api/v1/tenant/billing-addresses.md`](../tenant/billing-addresses.md)

## Base routes

All under `/api/v1/app/{company}`:

| Party | Base route |
| --- | --- |
| Tenant | `/users/tenants/{tenant}/billing-addresses` |
| Supplier | `/users/vendors/{vendor}/billing-addresses` |
| Landlord | `/users/landlords/{landlord}/billing-addresses` |
| Property | `/property-management/facilities/{facility}/billing-addresses` |

`{tenant}`, `{vendor}` and `{landlord}` are `users.id`. `{facility}` is `facilities.id`.

## Endpoints

Each base route is a standard API resource:

- `GET    {base}` — list
- `POST   {base}` — create
- `GET    {base}/{billingAddress}` — show
- `PUT|PATCH {base}/{billingAddress}` — update
- `DELETE {base}/{billingAddress}` — soft delete

Example: `GET /api/v1/app/12/users/tenants/481/billing-addresses`

### Not found cases (`404`)

- `{tenant}` is not a tenant, `{vendor}` is not a supplier, or `{landlord}` is not a landlord. The id
  of a real user in another role still returns `404`.
- `{billingAddress}` belongs to a different party than the one in the route.
- `{billingAddress}` was deleted, or was replaced by an edit (see [Locked addresses](#locked-addresses)).

## Authorization

| Action | Permission |
| --- | --- |
| Index / Show | `view-billing-address` |
| Store | `create-billing-address` |
| Update | `update-billing-address` |
| Destroy | `delete-billing-address` |

- **Property addresses:** these also need the property to be allocated to the user, as for every other
  property record. A user without the allocation gets `403`.
- **Portals:** the same four permissions exist for the supplier and tenant portal groups. Portal
  requests are decided by ownership: a portal user only reaches the addresses of the account they act
  for.
- **Row permissions:** each row carries a `permissions` map. Use it to show or hide the edit and delete
  actions.

## List

`GET {base}`

Paginated.

| Param | Notes |
| --- | --- |
| `filter[is_default]` | `1` / `0` |
| `filter[is_shipping]` | `1` / `0` |
| `filter[always_cc]` | `1` / `0` |
| `sort` | `id`, `name`, `created_at` (prefix `-` for descending) |
| `per_page` | Defaults to `config('app.query.default_per_page')` |
| `page` | |

Without `sort`, the default address comes first, then the rest newest first.

There are no includes.

## Request body (store / update)

```jsonc
{
  "name": "Acme Holdings Ltd",
  "contact_person": "Jane Wanjiru",
  "email": "accounts@acme.co.ke",
  "phone": "+254700000000",
  "physical_address": "Westlands Road, P.O. Box 12345-00100, Nairobi",
  "tax_pin": "P051234567X",
  "is_shipping": false,
  "is_default": true,
  "always_cc": false
}
```

| Field | Type | Required | Rules |
| --- | --- | --- | --- |
| `name` | string | Yes on store; optional on update (keeps the name on file) | Max 255 |
| `contact_person` | string/null | No | Max 255 |
| `email` | string/null | When `always_cc` is `true` | Valid email, max 255 |
| `phone` | string/null | No | Max 50 |
| `physical_address` | string/null | No | Max 2000. Free text: street, P.O. Box, town |
| `tax_pin` | string/null | No | Max 50 |
| `is_shipping` | boolean | No | Defaults to `false` |
| `is_default` | boolean | No | See [Default address](#default-address) |
| `always_cc` | boolean | No | Defaults to `false`. Needs an `email` |

- **The party comes from the route.** Never send `billable_type`, `billable_id`, `user_id` or
  `facility_id` in the body.
- **Update keeps fields you leave out.** On update, a field missing from the body keeps its value.
  Send `null` to clear an optional field. A body of only `{"is_default": true}` sets the default.
- **Always-CC email message.** When `always_cc` is `true` without an `email`, the `422` on `email`
  reads "An address copied on every document needs an email."

## Response shape

Index rows and show:

```jsonc
{
  "id": 37,
  "billable_kind": "user", // user | property
  "billable_id": 481,
  "name": "Acme Holdings Ltd",
  "contact_person": "Jane Wanjiru",
  "email": "accounts@acme.co.ke",
  "phone": "+254700000000",
  "physical_address": "Westlands Road, P.O. Box 12345-00100, Nairobi",
  "tax_pin": "P051234567X",
  "is_shipping": false,
  "is_default": true,
  "always_cc": false,
  "is_locked": true,
  "locked_at": "2026-10-22T09:14:03.000000Z", // null until a document uses the address
  "created_at": "2026-10-20T07:41:55.000000Z",
  "updated_at": "2026-10-22T09:14:03.000000Z",
  "permissions": { "view": true, "update": true, "delete": true }
}
```

- `billable_kind` is `property` for a property's address. It is `user` for a tenant, supplier or
  landlord.
- `billable_id` is the `facilities.id` or `users.id` from the route.

Store and update use the `DataResource` envelope. The address returned is the one **now on file**,
which can have a new `id` after an update:

```jsonc
{
  "data": {
    "message": "Billing address created successfully.", // or "Billing address updated successfully."
    "billing_address": { /* …the shape above… */ }
  }
}
```

Destroy:

```jsonc
{ "data": { "message": "Billing address deleted successfully." } }
```

## Default address

A party with addresses always has exactly one default.

- **First address.** A party's first address becomes its default, whatever `is_default` says.
- **Moving the default.** Sending `is_default: true` on store or update makes that address the
  default and unsets the previous one.
- **Unsetting is ignored.** Sending `is_default: false` on the current default does nothing. To move
  the default, make another address the default.
- **Deleting the default.** Deleting the default promotes the party's oldest remaining address. A
  party with no addresses left has no default.

After any of these, refetch the list. More than one row can change.

## Locked addresses

An address is **locked** once a document has been issued with it (`is_locked: true`, `locked_at` set).
The documents that do this are invoices, credit notes, receipts, LPOs, payment vouchers, remittances
and customer invoices.

A locked address is never changed in place. When an update changes any **printed detail** (`name`,
`contact_person`, `email`, `phone`, `physical_address` or `tax_pin`):

1. **A new address is created.** It holds the changes, has a **new `id`** and is not locked. It keeps
   the old address's `is_shipping` and `always_cc` unless the body changes them.
2. **The default moves.** If the old address was the default, the new one becomes the default.
3. **The old address is soft-deleted.** It drops out of the list and its id returns `404`. Documents
   issued with it keep printing what they were issued with.

The update response returns the new address. **Reload the list after every update.** The row you
edited may have been replaced by one with a different id.

An update that changes only flags (`is_shipping`, `always_cc`, `is_default`) is applied in place, even
on a locked address. An address no document has used yet is always edited in place.

Deleting a locked address is a soft delete too. Documents issued with it keep printing it.

Suggested UI copy on the edit form of a locked address: "This address is on issued documents. Saving
creates a new version; documents already issued keep the old one."

## Where addresses are used

### Printed documents

Each document records the address of each of its parties when it is created. It prints:

1. **The stamped address.** This is the address recorded at issue, even if it was later replaced or
   deleted.
2. **The party's current default.** Used when nothing was recorded, such as documents issued before
   billing addresses existed.
3. **The party's own details.** Name, email, phone, address and tax PIN from the user, or for a
   property its name, physical address and the landlord's tax PIN. Used when the party has no address
   at all.

| Document | Parties printed |
| --- | --- |
| Invoice, credit note, receipt | property, tenant, landlord |
| LPO | property, property shipping ("Deliver To"), supplier, landlord |
| Payment voucher | payee |
| Remittance | property, landlord |
| Customer invoice | customer (the landlord) |

**LPO "Deliver To".** This is the property's shipping address, picked in this order:

1. the default, if it is a shipping address
2. otherwise the oldest shipping address
3. otherwise the default

Changing an LPO's supplier records the new supplier's address.

The addresses appear on PDFs through document-template designer blocks. Templates show them only once
the block is added in the designer. Block keys and payload paths are in `docs/document-templates.md`.

**ETR signing is unchanged.** It still uses the user's own `tax_pin`, not the billing address.

### Statements

Tenant and supplier statements show the party's **current** default billing address in their headers.
A statement is produced when it is run, so it has no stamped address.

### Emails

These document emails go **To** the party's default billing address email:

- invoices, credit notes and receipts
- invoice reminders
- LPOs
- payment vouchers and remittances

When the default has no email, they fall back to the user's own `email`.

Every address of the party marked `always_cc` is copied (**CC**). Duplicates and the To address are
removed.

Emails use the addresses on file **when they are sent**, not the ones stamped on the document. A
reminder for an old invoice therefore reaches the current contact.

All other emails (welcome, PIN, approvals and so on), SMS and WhatsApp are unchanged. They use the
user's own email and phone.

## After deploy

```bash
php artisan migrate
php artisan db:seed --class=NewPermissionsSeeder
php artisan db:seed --class=PortalAdminRolesSeeder
php artisan billing-addresses:backfill            # add --company={id} for one company, --dry-run to preview
```

What each step does:

- **`NewPermissionsSeeder`** creates the four `*-billing-address` permissions.
- **`PortalAdminRolesSeeder`** grants them to the Tenant and Vendor admin roles.
- **`billing-addresses:backfill`** creates a default address for every tenant, supplier, landlord and
  property that has none. It copies the party's current details (the same values documents fall back
  to). It is safe to run again: a party that already has an address is skipped. Existing documents are
  not stamped. They print the party's default, which matches what they printed before.
