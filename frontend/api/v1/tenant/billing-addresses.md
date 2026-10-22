# Tenant Billing Addresses API

Domain: `Tenant Portal`

Base route:

`/api/v1/tenant/billing-addresses`

The billing addresses of the tenant account the signed-in user acts for. The tenant is **inferred from
the authenticated user**. There is no tenant id in the URL.

These are the same addresses staff see under `/api/v1/app/{company}/users/tenants/{tenant}/billing-addresses`.
The request body, response shape and rules are shared, and are described in the
[app Billing Addresses doc](../app/billing-addresses.md):

- [Request body](../app/billing-addresses.md#request-body-store--update)
- [Response shape](../app/billing-addresses.md#response-shape)
- [Default address](../app/billing-addresses.md#default-address)
- [Locked addresses](../app/billing-addresses.md#locked-addresses)
- [Where addresses are used](../app/billing-addresses.md#where-addresses-are-used)

## Endpoints

- `GET    /billing-addresses` — list
- `POST   /billing-addresses` — create
- `GET    /billing-addresses/{billingAddress}` — show
- `PUT|PATCH /billing-addresses/{billingAddress}` — update
- `DELETE /billing-addresses/{billingAddress}` — soft delete

## List

Paginated.

- `filter[is_default]`, `filter[is_shipping]`, `filter[always_cc]` (`1` / `0`)
- `sort=id,name,created_at` (prefix `-` for descending)
- `per_page`, `page`

Without `sort`, the default address comes first, then the rest newest first.

## Access

- **Ownership:** an address belonging to any other party returns `404`.
- **Permissions:** `view-billing-address`, `create-billing-address`, `update-billing-address` and
  `delete-billing-address` are granted to the Tenant Admin role.
- **Row permissions:** use each row's `permissions` map to show or hide edit and delete.

## What the tenant should know

- **The default address.** It is printed on invoices, credit notes and receipts, and receives invoice,
  credit note, receipt and reminder emails.
- **Always CC.** Mark an address `always_cc` (for example an accounts department) to copy its email on
  every one of those emails. It needs an `email`.
- **Editing a used address.** Once a document has been issued with an address, saving changes to it
  creates a new address with a new `id` and retires the old one. Documents already issued keep the old
  address. Reload the list after every update.
- **SMS and WhatsApp.** These still go to the tenant's own phone number.
