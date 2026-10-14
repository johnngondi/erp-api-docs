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

## Deposits are taken per component (row 143)

A deposit is held against a **component** - Rent, Service Charge, Parking - not against one unit.
Pick from the components the lease has on **at least one** of its units.

All the deposits taken together are billed as **one invoice line** under the **Deposit**
component, narrated with what it covers:

| Taken | Invoice line |
|---|---|
| Rent 120,000 · Service Charge 30,000 · Parking 5,000 | `Deposit: Rent, Service Charge, Parking` - 155,000 |

So the billing ledger books the money as deposit, not as rent. Each deposit row still keeps its own
component (`lease_component`), which is what the deposits tab, the Tenant Deposits report and the
refund read.

Every deposit points at the lease's single **Deposit line** (`lease_item_component`): a Deposit
component added to the lease's first unit at a cost of 0 the first time a deposit is taken, so it
is never billed by the monthly cycle. **Show `lease_component` as the deposit's type**, not
`lease_item_component` - for anything taken since this change that is the Deposit line.

Deposits taken before this change keep their own invoices, with a line per deposit.

## List Lease Deposits

`GET /api/v1/app/{company}/property-management/lease-management/leases/{lease}/deposits`

Supported query params:

- Filters:
  - `filter[id]`
  - `filter[lease_component_id]` (exact)
  - `filter[lease_item_component_id]`
  - `filter[created_at]`
- Sort:
  - `sort=id,lease_component_id,lease_item_component_id,created_at`
- Include: not supported
- Select fields: not supported
- Pagination: `per_page`, `page`

Example:

`GET /api/v1/app/12/property-management/lease-management/leases/101/deposits?filter[lease_component_id]=1&sort=-created_at`

Each row:

| Field | Type | Notes |
|---|---|---|
| `lease_component_id` | integer | What the deposit is held against |
| `lease_component` | object | `{ id, name, ... }` - **show this as the deposit's type** |
| `lease_item_component` | object | The lease line it was billed on: the Deposit line for anything taken since row 143 |
| `amount`, `billed`, `invoice`, `created_at` | - | As before |
| `shares_document`, `document_siblings` | - | See [Deposits raised together cannot be changed one at a time](#deposits-raised-together-cannot-be-changed-one-at-a-time) |

## Create Lease Deposit

`POST /api/v1/app/{company}/property-management/lease-management/leases/{lease}/deposits`

Send a `deposits` array; they are raised together on **one invoice, one line**:

```json
{ "deposits": [
  { "lease_component_id": 1, "amount": 120000 },
  { "lease_component_id": 2, "amount": 30000 },
  { "lease_component_id": 7, "amount": 5000 }
]}
```

| Field | Required | Type | Allowed Values / Notes |
|---|---|---|---|
| `deposits` | Yes | array | At least one deposit |
| `deposits[].lease_component_id` | Yes | integer | A component the lease has on at least one unit. Not the Deposit or legal fee component itself. Once per request |
| `deposits[].amount` | Yes | number | A zero amount is skipped |

A single deposit can still be sent as the body itself (`{ "lease_component_id": 1, "amount": 500 }`).

The response carries **`lease_deposits`** (the whole set) alongside `lease_deposit` (the first).

**Refusals** (422), keyed by position so the row can be marked:

| Error | When |
|---|---|
| `deposits.1.lease_component_id: ["This lease has no unit with Parking."]` | No unit on the lease carries it |
| `deposits.2.lease_component_id: ["Choose what the deposit is held against, such as Rent, rather than Deposit."]` | The Deposit or legal fee component was picked |
| `deposits.3.lease_component_id: ["Rent is listed twice. Take one deposit per component."]` | The same component twice in one request |
| `lease_component_id` / `deposits: ["No deposit component is set up. ..."]` | No lease component is flagged as a deposit in settings |

**The older payload still works.** `lease_item_component_id` (one unit's line) in place of
`lease_component_id` is read as the component that line carries; it must be a line on this lease
(`"The selected component is not on this lease."` otherwise). Move to `lease_component_id` - the
older field will be removed.

### Deposits on a pending lease are billed when the lease is approved

While the lease is `pending`, a deposit is saved **without an invoice**: the row comes back with
`billed: false` and `invoice: null`, and nothing is sent to the tenant. When the lease becomes
active, by approval or through the activate endpoint, every unbilled deposit on it is raised on
**one invoice line** under Deposit, the rows switch to `billed: true` with their `invoice`, and the
invoice is emailed to the tenant once it is processed. A rejected lease never bills its deposits.

On a pending lease, update and delete change the row alone; there is no invoice to reissue or
cancel. On any other lease, deposits bill at once. The `billed` request field is ignored: the
server decides.

## Update Lease Deposit

`PUT/PATCH /api/v1/app/{company}/property-management/lease-management/leases/{lease}/deposits/{leaseDeposit}`

`{ "lease_component_id": 1, "amount": 125000 }` - the same rules as create. A deposit that has its
own invoice is re-raised on a new one (one Deposit line) and the old invoice is cancelled. Sending
back the deposit's own `lease_item_component_id` (the older payload) keeps its component.

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
