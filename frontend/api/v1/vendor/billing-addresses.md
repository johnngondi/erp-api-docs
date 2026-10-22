# Vendor Billing Addresses API

Domain: `Supplier Portal`

Base route:

`/api/v1/vendor/billing-addresses`

The billing addresses of the supplier account the signed-in user acts for. Supplier sub-users manage
the addresses of the supplier they belong to. There is no supplier id in the URL.

These are the same addresses staff see under `/api/v1/app/{company}/users/vendors/{vendor}/billing-addresses`.
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
  `delete-billing-address` are granted to the Vendor Admin role. A sub-user needs a role that carries
  them.
- **Row permissions:** use each row's `permissions` map to show or hide edit and delete.

## What the supplier should know

- **The default address.** It is printed on LPOs and payment vouchers, and receives LPO, payment
  voucher and remittance emails.
- **Always CC.** Mark an address `always_cc` to copy its email on every one of those emails. It needs
  an `email`.
- **Editing a used address.** Once a document has been issued with an address, saving changes to it
  creates a new address with a new `id` and retires the old one. Reload the list after every update.
- **ETR tax PIN.** ETR signing still uses the supplier's own tax PIN, not the billing address.
