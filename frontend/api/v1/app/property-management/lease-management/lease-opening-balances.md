# Lease Opening Balances API

Domain: `Property Management > Lease Management`

Base route:

`/api/v1/app/{company}/property-management/lease-management/leases/{lease}/opening-balances`

## Endpoints

- `GET /leases/{lease}/opening-balances`
- `POST /leases/{lease}/opening-balances`
- `GET /leases/{lease}/opening-balances/{leaseOpeningBalance}`
- `PUT/PATCH /leases/{lease}/opening-balances/{leaseOpeningBalance}`
- `DELETE /leases/{lease}/opening-balances/{leaseOpeningBalance}`

## List Lease Opening Balances

`GET /api/v1/app/{company}/property-management/lease-management/leases/{lease}/opening-balances`

Supported query params:

- Filters:
  - `filter[id]`
  - `filter[lease_component_id]`
  - `filter[opening_balance_at]`
  - `filter[created_at]`
- Sort:
  - `sort=id,lease_component_id,opening_balance_at,created_at,amount,billed_amount`
- Include: not supported
- Select fields: not supported
- Pagination: `per_page`, `page`

Example:

`GET /api/v1/app/12/property-management/lease-management/leases/101/opening-balances?sort=-opening_balance_at`

## Create Lease Opening Balance

`POST /api/v1/app/{company}/property-management/lease-management/leases/{lease}/opening-balances`

Request body:

| Field | Required | Type | Allowed Values / Notes |
|---|---|---|---|
| `amount` | Yes | number | - |
| `opening_balance_at` | Yes | date (`YYYY-MM-DD`) | - |
| `lease_component_id` | No | integer | Must exist in `lease_components.id` |
| `notes` | No | string | Optional |

Example request:

```json
{
  "lease_component_id": 9,
  "amount": 200,
  "opening_balance_at": "2026-01-01",
  "notes": "Carried from previous accounting year"
}
```

Example response:

```json
{
  "message": "Lease opening balance created successfully",
  "opening_balance": {
    "id": 88,
    "amount": 200,
    "billed_amount": 0
  }
}
```

### Creating several opening balances at once

Send an `opening_balances` array and they are raised together: **every component in arrears on one
invoice, every component in credit on one credit note.**

```json
{ "opening_balances": [
  { "lease_component_id": 1,  "amount": 15000, "opening_balance_at": "2026-08-31" },
  { "lease_component_id": 2,  "amount": 4000,  "opening_balance_at": "2026-08-31" },
  { "lease_component_id": 13, "amount": -2500, "opening_balance_at": "2026-08-31" }
]}
```

| Field | Required | Type | Allowed Values / Notes |
|---|---|---|---|
| `opening_balances` | Yes | array | At least one row |
| `opening_balances[].lease_component_id` | Yes | integer | Must have a matching lease item component on this lease |
| `opening_balances[].amount` | Yes | number | **Signed.** Positive is owed by the tenant, negative is owed to them. Zero is skipped |
| `opening_balances[].opening_balance_at` | Yes | date (`YYYY-MM-DD`) | The date the balance is as at |
| `opening_balances[].tax_amount` | No | number | Signed like `amount`; omit for no tax |

In the example above, Rent and Service Charge land on one invoice and Conveyancing Costs on its own
credit note. Each row's response carries either `invoice` or `credit_note`, never both.

The response carries **`opening_balances`** (the whole set) alongside `opening_balance` (the first),
so the single-row caller keeps working unchanged.

Errors are keyed by position: `opening_balances.1.amount`.

One opening balance at a time still works exactly as documented above.

## Update Lease Opening Balance

`PUT/PATCH /api/v1/app/{company}/property-management/lease-management/leases/{lease}/opening-balances/{leaseOpeningBalance}`

Use the same payload shape as create.

## Delete Lease Opening Balance

`DELETE /api/v1/app/{company}/property-management/lease-management/leases/{lease}/opening-balances/{leaseOpeningBalance}`

### Balances raised together cannot be changed one at a time

Update and delete both cancel or reissue the balance's invoice or credit note. When several rows
share one, doing that to a single row would take the other components with it - so it is refused:

```json
{ "message": "This opening balance shares one document with 1 other. Remove or reissue the whole document instead of changing one line of it.",
  "errors": { "opening_balance": ["This opening balance shares one document with 1 other. ..."] } }
```

**422, keyed `opening_balance`.** Show the message and offer to reverse the whole document instead.
A balance that owns its invoice alone updates and deletes exactly as before.

Each row carries a flag so the client can decide whether to offer edit and delete without waiting
for a 422:

| Field | Type | Notes |
|---|---|---|
| `shares_document` | boolean | `true` when others sit on the same invoice or credit note |
| `document_siblings` | integer | How many others, for the message |

**A pair raised together does not always share.** One row in arrears and one in credit split
across an invoice and a credit note, so each ends up alone on its own document and both stay
individually editable - read the flag rather than assuming anything from how they were submitted.


## Frontend Error Handling

Apply shared rules in `docs/frontend/app/README.md`.
