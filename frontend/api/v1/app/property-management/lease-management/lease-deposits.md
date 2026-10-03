# Lease Deposits API

Domain: `Property Management > Lease Management`

Base route:

`/api/v1/app/{company}/property-management/lease-management/leases/{lease}/deposits`

## Endpoints

- `GET /leases/{lease}/deposits`
- `POST /leases/{lease}/deposits`
- `GET /leases/{lease}/deposits/{leaseDeposit}`
- `PUT/PATCH /leases/{lease}/deposits/{leaseDeposit}`
- `DELETE /leases/{lease}/deposits/{leaseDeposit}`

## List Lease Deposits

`GET /api/v1/app/{company}/property-management/lease-management/leases/{lease}/deposits`

Supported query params:

- Filters:
  - `filter[id]`
  - `filter[lease_item_component_id]`
  - `filter[created_at]`
- Sort:
  - `sort=id,lease_item_component_id,created_at`
- Include: not supported
- Select fields: not supported
- Pagination: `per_page`, `page`

Example:

`GET /api/v1/app/12/property-management/lease-management/leases/101/deposits?filter[lease_item_component_id]=455&sort=-created_at`

## Create Lease Deposit

`POST /api/v1/app/{company}/property-management/lease-management/leases/{lease}/deposits`

Request body:

| Field | Required | Type | Allowed Values / Notes |
|---|---|---|---|
| `lease_item_component_id` | Yes | integer | Must exist in `lease_item_components.id` |
| `amount` | Yes | number | - |
| `billed` | Yes | boolean | Usually `false` on create unless intentionally billed |

Example request:

```json
{
  "lease_item_component_id": 455,
  "amount": 500,
  "billed": false
}
```

Example response:

```json
{
  "message": "Lease deposit created successfully",
  "lease_deposit": {
    "id": 45,
    "amount": 500,
    "billed": false
  }
}
```

### Creating several deposits at once

Send a `deposits` array and they are raised together on **one invoice**, a line per deposit:

```json
{ "deposits": [
  { "lease_item_component_id": 455, "amount": 50000 },
  { "lease_item_component_id": 456, "amount": 20000 },
  { "lease_item_component_id": 457, "amount": 10000 }
]}
```

| Field | Required | Type | Allowed Values / Notes |
|---|---|---|---|
| `deposits` | Yes | array | At least one deposit |
| `deposits[].lease_item_component_id` | Yes | integer | Must exist in `lease_item_components.id` |
| `deposits[].amount` | Yes | number | A zero amount is skipped |

The response carries **`lease_deposits`** (the whole set) alongside `lease_deposit` (the first),
so the single-row caller keeps working unchanged.

Errors are keyed by position, so the offending row can be marked: `deposits.1.amount`.

One deposit at a time still works exactly as documented above, and still raises its own invoice.

### Deposits on a pending lease are billed when the lease is approved

While the lease is `pending`, a deposit is saved **without an invoice**: the row comes back with
`billed: false` and `invoice: null`, and nothing is sent to the tenant. When the lease becomes
active, by approval or through the activate endpoint, every unbilled deposit on it is raised on
**one invoice**, the rows switch to `billed: true` with their `invoice`, and the invoice is
emailed to the tenant once it is processed. A rejected lease never bills its deposits.

On a pending lease, update and delete change the row alone; there is no invoice to reissue or
cancel. On any other lease, deposits bill at once, as before. The `billed` request field is
ignored: the server decides.

## Update Lease Deposit

`PUT/PATCH /api/v1/app/{company}/property-management/lease-management/leases/{lease}/deposits/{leaseDeposit}`

Use the same payload shape as create.

## Delete Lease Deposit

`DELETE /api/v1/app/{company}/property-management/lease-management/leases/{lease}/deposits/{leaseDeposit}`

### Deposits raised together cannot be changed one at a time

Update and delete both cancel or reissue the deposit's invoice. When several deposits share one
invoice, doing that to a single deposit would take the others with it - so it is refused:

```json
{ "message": "This deposit shares one document with 2 others. Remove or reissue the whole document instead of changing one line of it.",
  "errors": { "deposit": ["This deposit shares one document with 2 others. ..."] } }
```

**422, keyed `deposit`.** Show the message and offer to reverse the whole invoice instead. A
deposit that owns its invoice alone updates and deletes exactly as before.

Each row carries `shares_document` (boolean) and `document_siblings` (integer), so the client can
disable edit and delete up front rather than waiting for the 422.

## Frontend Error Handling

Apply shared rules in `docs/frontend/app/README.md`.
