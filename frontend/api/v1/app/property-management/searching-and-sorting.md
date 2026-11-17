# Property Management API: Searching and Sorting

This document is for frontend developers integrating list endpoints.

## Conventions

- Query style uses Spatie Query Builder conventions.
- Filters are sent as `filter[field]=value`.
- Sorting is sent as `sort=field` (ascending) or `sort=-field` (descending).
- Pagination uses `per_page`.
- Multiple filters can be combined in one request.

Example shape:

```http
GET /.../resource?filter[search]=...&filter[status]=...&sort=-created_at&per_page=20
```

## Searching by ID

`filter[search]` is answered by the search engine **and** by a direct ID match, not one or the
other. A numeric term matches any record whose ID it is, on top of whatever the engine returns.

This applies to leases, procurement requests and remittances. Bills, invoices, credit notes,
expenses, LPOs, procurement bids, receipts and payment vouchers answer a number search exactly
instead: see [Searching by number](#searching-by-number-bills-invoices-credit-notes-receipts-and-vouchers).

Two consequences worth knowing:

**Pasting an ID always finds the record.** Even when the search index is missing, stale or the
engine is unreachable. That is the case people reach for an ID in, so it is the case that most
needs to work.

**A numeric term is still a text search as well.** Searching `1004` finds the record whose ID is
1004 *and* the one whose invoice number is `1004`. The ID match is added, never substituted, so
nothing that used to be found stops being found.

> This is a floor for IDs, not a cover for a broken index. A text search against an unreachable
> engine still correctly returns nothing rather than silently falling back to a slower, different
> search — a broken index should look broken.

## Searching by number: bills, invoices, credit notes, receipts and vouchers

On these listings, a `filter[search]` made **only of digits** is answered exactly, by the
database. The search engine is not asked. Each number returns every record that matches it in any
of these ways:

| Listing | A number matches |
|---|---|
| Bills (`/finance/bills`) | the id · `total` · `invoice_number` · `tax_invoice_number` (CU number) · bills raised from the LPO with that id or document number |
| Expenses (`/finance/expenses`) | the id · `total` · `invoice_number` · `tax_invoice_number` |
| Invoices (`/lease-management/invoices`) | the id · `total` · `cu_invoice_number` · `cu_reference_number` |
| Credit notes (`/lease-management/credit-notes`) | the id · `total` · `cu_invoice_number` · `cu_reference_number` |
| LPOs (`/procurement/lpos`) | the id · `total` · `document_number` |
| Procurement bids (`/procurement/requests/{procurementRequest}/bids`) | the id · `total` |
| Receipts (`/lease-management/receipts`) | the id · `amount` · `transaction_number` |
| Payment vouchers (`/finance/payment-vouchers`) | the id · `amount` · `payment_advice_number` · `transaction_number` |

- **Exact, not "contains".** `50000` finds a bill whose total is 50,000.00 and the bill numbered
  `50000`, never one worth 500,001 or numbered `INV-50000`. Document numbers are compared as typed,
  so a leading zero counts.
- **Everything that matches, together.** The record with that id comes back *with* the records
  worth that amount, not instead of them.
- **A comma still separates values.** `50000,75000` matches each number the same way.
- The listing's other filters still apply, so `filter[facility_id]` narrows a number search as usual.
- The same rule applies on the tenant, vendor and landlord portals' versions of these listings.

```http
GET /finance/bills?filter[search]=50000
GET /lease-management/receipts?filter[search]=1205
GET /lease-management/invoices?filter[search]=50000,75000
```

## Searching by money: the final total only

Bills, expenses, invoices, credit notes, LPOs and procurement bids carry an amount before VAT, the
VAT, and the final total. A money search matches **the final total only**. The amount before VAT,
the VAT (`tax`), the amount paid and the balance are never matched. Receipts and payment vouchers
carry no VAT: their `amount` is their total.

Take a bill of KES 10,000.00 before VAT, KES 1,600.00 VAT, KES 11,600.00 total:

| `filter[search]` | Finds the bill? | Why |
|---|---|---|
| `11600` | Yes | the total |
| `KES 11,600.00` | Yes | currency and decimals are ignored |
| `10000` | No | the amount before VAT is not searched |
| `1600` | No | the VAT is not searched |
| `11,600` | No (by amount) | a comma separates values: this is the list 11 and 600 |

**Type amounts without commas.** A comma in the search box separates values, because that is how
a list of ids is searched: `11,600` is the list 11 and 600, each matched as a plain number (the
record with that id, that document number, or that total). It is never eleven thousand six hundred.

**Formats read as money.** A plain number (`11600`) matches the record with that id or document
number, or the total. Written with a currency code or symbol (`KES`, `KSh`, `Ksh.`, `Shs`, `USD`,
`$`, `€`, `£` and the other codes in use), decimals (up to five places), a minus sign or a trailing
`/=`, the term can only be an amount: there, spaces and thousands commas are ignored. These all
search for 11,600.00:

```
11600   11600.00   11,600.00   KES 11600   KES 11,600   KSh11600   Ksh. 11,600/=   11600 KES   11 600
```

- **Compared as shown.** The total is compared rounded to two decimal places, and without its
  sign: a credit note stored as -11,600.00 is found by `11600` (and by `-11,600`).
- **Written as money, it is only an amount.** `KES 11,600` or `11600.00` matches the total alone,
  never an id or a document number.
- **A list is several plain numbers.** `12,15,19` matches each number the way `12` alone would:
  ids, document numbers or totals.
- **Anything else is a text search**, unchanged: `INV-11600`, `acme`, or a number mixed with
  words (`acme 11600`). Names, references, suppliers, tenants and properties are searched as
  before.
- These listings have no export, so there is no export to keep in step.

Affected endpoints (staff app, under `/api/v1/app/{company}/property-management`, plus the portals):

| Listing | Endpoint |
|---|---|
| Bills | `GET /finance/bills` · vendor `GET /api/v1/vendor/finance/bills` |
| Expenses | `GET /finance/expenses` · landlord `GET /api/v1/landlord/facilities/{facility}/expenses` |
| Invoices | `GET /lease-management/invoices` · tenant `GET /api/v1/tenant/billing/invoices` |
| Credit notes | `GET /lease-management/credit-notes` · tenant `GET /api/v1/tenant/billing/credit-notes` |
| LPOs | `GET /procurement/lpos` · vendor `GET /api/v1/vendor/procurement/lpos` |
| Procurement bids | `GET /procurement/requests/{procurementRequest}/bids` · vendor `GET /api/v1/vendor/procurement/rfq/quotes` |

```http
GET /finance/bills?filter[search]=11600
GET /finance/expenses?filter[search]=KES%2011%2C600.00
GET /lease-management/credit-notes?filter[search]=11600
```

## Invoice (`/lease-management/invoices`)

### Supported filters

- `filter[search]` - string
- `filter[facility_id]` - integer (ID from list of facilities)
- `filter[lease_id]` - integer (lease ID)
- `filter[due_at]` - date string (recommended format: `YYYY-MM-DD`)
- `filter[created_at]` - date or datetime string (`YYYY-MM-DD` or ISO datetime)
- `filter[status]` - string enum (see allowed values below)

### `filter[status]` available options

- `pending`
- `unpaid`
- `partially paid`
- `paid`
- `cancelled`

Examples:

```http
GET /lease-management/invoices?filter[status]=unpaid
GET /lease-management/invoices?filter[status]=partially%20paid
```

### `filter[search]` behavior

`search` has special behavior:

1. Numbers only, one or comma-separated: an exact match on the id, the final total, the CU invoice
number or the CU reference number (never the amount before VAT or the VAT). See
[Searching by number](#searching-by-number-bills-invoices-credit-notes-receipts-and-vouchers).
An amount written as money (`KES 11,600`, `11600.00`) matches the total alone: see
[Searching by money](#searching-by-money-the-final-total-only).
Input format: `number`, `number,number,number` or an amount

Example:

```http
GET /lease-management/invoices?filter[search]=1,2,3
GET /lease-management/invoices?filter[search]=15000
```

2. Comma-separated CU invoice codes (letters + digits), exact match.
Input format: `CODE1,CODE2`

Example:

```http
GET /lease-management/invoices?filter[search]=INV100,INV200
```

3. Any other single value (text, phrase, text with numbers) uses normal text search.
Input format: any string

Examples:

```http
GET /lease-management/invoices?filter[search]=Collins
```

### Supported sorts

- `sort=id`
- `sort=due_at`
- `sort=created_at`
- `sort=amount`
- `sort=tax`
- `sort=total`
- `sort=paid`
- `sort=balance`

Sort format:

- Ascending: `sort=<field>`
- Descending: `sort=-<field>`

Examples:

```http
GET /lease-management/invoices?sort=due_at
GET /lease-management/invoices?sort=-created_at
GET /lease-management/invoices?sort=-total
```

### Combined examples

```http
GET /lease-management/invoices?filter[facility_id]=12&filter[status]=unpaid&sort=-due_at&per_page=25
GET /lease-management/invoices?filter[search]=INV100,INV200&sort=-id&per_page=50
GET /lease-management/invoices?filter[search]=Collins&filter[facility_id]=12&sort=-created_at
```

## Credit Note (`/lease-management/credit-notes`)

### Supported filters

- `filter[search]` - string
- `filter[facility_id]` - integer (ID from list of facilities)
- `filter[lease_id]` - integer (lease ID)
- `filter[due_at]` - date string (recommended format: `YYYY-MM-DD`)
- `filter[created_at]` - date or datetime string (`YYYY-MM-DD` or ISO datetime)
- `filter[status]` - string enum (see allowed values below)

### `filter[status]` available options

- `pending`
- `applied`
- `cancelled`

Examples:

```http
GET /lease-management/credit-notes?filter[status]=applied
GET /lease-management/credit-notes?filter[status]=cancelled
```

`search` has special behavior:

1. Numbers only, one or comma-separated: an exact match on the id, the final total, the CU invoice
number or the CU reference number (never the amount before VAT or the VAT). See
[Searching by number](#searching-by-number-bills-invoices-credit-notes-receipts-and-vouchers).
An amount written as money (`KES 11,600`, `11600.00`) matches the total alone: see
[Searching by money](#searching-by-money-the-final-total-only).
Input format: `number`, `number,number,number` or an amount

Example:

```http
GET /lease-management/credit-notes?filter[search]=1,2,3
GET /lease-management/credit-notes?filter[search]=15000
```

2. Comma-separated code list (letters + digits), exact match.
Input format: `CODE1,CODE2`

This applies to:
- CU invoice numbers
- CU reference numbers

Example:

```http
GET /lease-management/credit-notes?filter[search]=INV100,REF200
```

3. Any other single value (text, phrase, text with numbers) uses normal text search.
Input format: any string

Examples:

```http
GET /lease-management/credit-notes?filter[search]=Collins
```

### Supported sorts

- `sort=id`
- `sort=due_at`
- `sort=created_at`
- `sort=amount`
- `sort=tax`
- `sort=total`
- `sort=paid`
- `sort=balance`

Sort format:

- Ascending: `sort=<field>`
- Descending: `sort=-<field>`

Examples:

```http
GET /lease-management/credit-notes?sort=due_at
GET /lease-management/credit-notes?sort=-created_at
GET /lease-management/credit-notes?sort=-total
```

### Combined examples

```http
GET /lease-management/credit-notes?filter[facility_id]=12&filter[status]=applied&sort=-due_at&per_page=25
GET /lease-management/credit-notes?filter[search]=INV100,REF200&sort=-id&per_page=50
GET /lease-management/credit-notes?filter[search]=Collins&filter[facility_id]=12&sort=-created_at
```

## Receipt (`/lease-management/receipts`)

### Supported filters

- `filter[search]` - string
- `filter[receiving_account_id]` - integer (receiving bank account ID)
- `filter[transaction_date]` - date or datetime string (`YYYY-MM-DD` or ISO datetime)
- `filter[payment_method_id]` - integer (payment method ID)
- `filter[status]` - string enum (see allowed values below)
- `filter[created_at]` - date or datetime string (`YYYY-MM-DD` or ISO datetime)

### `filter[status]` available options

- `pending`
- `confirmed`
- `cancelled`

Examples:

```http
GET /lease-management/receipts?filter[status]=confirmed
GET /lease-management/receipts?filter[status]=cancelled
```

### `filter[search]` behavior

`search` has special behavior:

1. Numbers only, one or comma-separated: an exact match on the id, amount or transaction number.
See [Searching by number](#searching-by-number-bills-invoices-credit-notes-receipts-and-vouchers).
Input format: `number` or `number,number,number`

Example:

```http
GET /lease-management/receipts?filter[search]=1,2,3
GET /lease-management/receipts?filter[search]=15000
```

2. Comma-separated transaction code list (letters + digits), exact match.
Input format: `CODE1,CODE2`

Example:

```http
GET /lease-management/receipts?filter[search]=TRX100,TRX200
```

3. Any other single value (text, phrase, text with numbers) uses normal text search.
Input format: any string

Search covers:
- receipt ID
- transaction number
- paying user name
- amount
- receiving user name

Examples:

```http
GET /lease-management/receipts?filter[search]=Collins
```

### Supported sorts

- `sort=id`
- `sort=transaction_date`
- `sort=amount`
- `sort=created_at`

Sort format:

- Ascending: `sort=<field>`
- Descending: `sort=-<field>`

Examples:

```http
GET /lease-management/receipts?sort=transaction_date
GET /lease-management/receipts?sort=-created_at
GET /lease-management/receipts?sort=-amount
```

### Combined examples

```http
GET /lease-management/receipts?filter[status]=confirmed&filter[payment_method_id]=1&sort=-transaction_date&per_page=25
GET /lease-management/receipts?filter[search]=TRX100,TRX200&sort=-id&per_page=50
GET /lease-management/receipts?filter[search]=Collins&filter[receiving_account_id]=3&sort=-created_at
```

## Lease (`/lease-management/leases`)

### Supported filters

- `filter[search]` - string
- `filter[facility_id]` - integer (facility ID)
- `filter[user_id]` - integer (tenant user ID)
- `filter[status]` - string enum (see allowed values below)
- `filter[created_at]` - date or datetime string (`YYYY-MM-DD` or ISO datetime)
- `filter[start_at]` - date or datetime string (`YYYY-MM-DD` or ISO datetime)
- `filter[end_at]` - date or datetime string (`YYYY-MM-DD` or ISO datetime)
- `filter[next_due_at]` - date or datetime string (`YYYY-MM-DD` or ISO datetime)
- `filter[billing_cycle]` - string enum (cycle value)

### `filter[status]` available options

- `active`
- `suspended`
- `terminated`

Examples:

```http
GET /lease-management/leases?filter[status]=active
GET /lease-management/leases?filter[status]=suspended
```

### `filter[search]` behavior

`search` has special behavior:

1. Comma-separated numeric list (no spaces required): exact match by lease IDs.
Input format: `number,number,number`

Example:

```http
GET /lease-management/leases?filter[search]=1,2,3
```

2. Any other single value (text/number/phrase) uses normal text search.
Input format: any string

Search covers:
- lease ID
- tenant name
- facility name
- status
- billing cycle
- `start_at`
- `end_at`
- `next_due_at`

Examples:

```http
GET /lease-management/leases?filter[search]=Collins
GET /lease-management/leases?filter[search]=monthly
```

### Supported sorts

- `sort=id`
- `sort=created_at`
- `sort=start_at`
- `sort=end_at`

Sort format:

- Ascending: `sort=<field>`
- Descending: `sort=-<field>`

Examples:

```http
GET /lease-management/leases?sort=end_at
GET /lease-management/leases?sort=-created_at
GET /lease-management/leases?sort=-start_at
```

### Combined examples

```http
GET /lease-management/leases?filter[facility_id]=2&filter[status]=active&sort=-end_at&per_page=25
GET /lease-management/leases?filter[search]=1,2,3&sort=-id&per_page=50
GET /lease-management/leases?filter[search]=Collins&filter[billing_cycle]=monthly&sort=-created_at
```

## Lease Application (`/lease-management/lease-applications`)

### Supported filters

- `filter[search]` - string
- `filter[applicant_type]` - string enum (`personal`, `business`)
- `filter[status]` - string enum (see allowed values below)
- `filter[user_id]` - integer (applicant user ID)
- `filter[facility_type_id]` - integer (facility type ID)
- `filter[city_id]` - integer (city ID)
- `filter[applicant_registered_country_id]` - integer (country ID)
- `filter[application_submitted_at]` - date or datetime string (`YYYY-MM-DD` or ISO datetime)
- `filter[reviewed_at]` - date or datetime string (`YYYY-MM-DD` or ISO datetime)
- `filter[parking_required]` - boolean (`true` or `false`)
- `filter[generator_required]` - boolean (`true` or `false`)
- `filter[registration_type]` - string enum (`national_id`, `business_license`, `passport`)
- `filter[created_at]` - date or datetime string (`YYYY-MM-DD` or ISO datetime)

### `filter[status]` available options

- `pending`
- `approved`
- `review`
- `rejected`

Examples:

```http
GET /lease-management/lease-applications?filter[status]=pending
GET /lease-management/lease-applications?filter[status]=approved
```

### `filter[search]` behavior

`search` has special behavior:

1. Comma-separated numeric list (no spaces required): exact match by lease application IDs.
Input format: `number,number,number`

Example:

```http
GET /lease-management/lease-applications?filter[search]=1,2,3
```

2. Any other single value (text/number/phrase) uses normal text search.
Input format: any string

Search covers:
- application ID
- applicant name
- applicant contact email
- applicant contact phone
- applicant type
- status
- application submitted date
- applicant user name
- facility type name
- city name

Examples:

```http
GET /lease-management/lease-applications?filter[search]=Acme
GET /lease-management/lease-applications?filter[search]=Nairobi
```

### Supported sorts

- `sort=id`
- `sort=created_at`
- `sort=application_submitted_at`
- `sort=reviewed_at`

Sort format:

- Ascending: `sort=<field>`
- Descending: `sort=-<field>`

Examples:

```http
GET /lease-management/lease-applications?sort=application_submitted_at
GET /lease-management/lease-applications?sort=-created_at
GET /lease-management/lease-applications?sort=-reviewed_at
```

### Combined examples

```http
GET /lease-management/lease-applications?filter[status]=review&filter[facility_type_id]=2&sort=-application_submitted_at&per_page=25
GET /lease-management/lease-applications?filter[search]=1,2,3&sort=-id&per_page=50
GET /lease-management/lease-applications?filter[search]=Acme&filter[city_id]=1&sort=-created_at
```

## Facility (`/facilities`)

### Supported filters

- `filter[search]` - string
- `filter[status]` - string enum (see allowed values below)
- `filter[landlord_id]` - integer (landlord user ID)
- `filter[facility_type_id]` - integer (facility type ID)
- `filter[country_id]` - integer (country ID)
- `filter[city_id]` - integer (city ID)
- `filter[created_at]` - date or datetime string (`YYYY-MM-DD` or ISO datetime)

### `filter[status]` available options

- `pending`
- `active`
- `inactive`

Examples:

```http
GET /facilities?filter[status]=active
GET /facilities?filter[status]=inactive
```

### `filter[search]` behavior

`search` has special behavior:

1. Comma-separated numeric list (no spaces required): exact match by facility IDs.
Input format: `number,number,number`

Example:

```http
GET /facilities?filter[search]=1,2,3
```

2. Any other single value (text/number/phrase) uses normal text search.
Input format: any string

Search covers:
- facility ID
- facility name
- facility description
- physical address
- status
- landlord name
- facility type title
- country name
- city name

Examples:

```http
GET /facilities?filter[search]=Riverside
GET /facilities?filter[search]=Nairobi
```

### Supported sorts

- `sort=id`
- `sort=name`
- `sort=status`
- `sort=created_at`
- `sort=construction_date`

Sort format:

- Ascending: `sort=<field>`
- Descending: `sort=-<field>`

Examples:

```http
GET /facilities?sort=name
GET /facilities?sort=-created_at
GET /facilities?sort=-construction_date
```

### Combined examples

```http
GET /facilities?filter[status]=active&filter[city_id]=1&sort=-created_at&per_page=25
GET /facilities?filter[search]=1,2,3&sort=-id&per_page=50
GET /facilities?filter[search]=Riverside&filter[facility_type_id]=2&sort=name
```

## Ticket (`/tickets`)

### Supported filters

- `filter[search]` - string
- `filter[ticket_type]` - string enum (`asset`, `facility`, `other`)
- `filter[ticket_category_id]` - integer (ticket category ID)
- `filter[facility_id]` - integer (facility ID)
- `filter[facility_space_id]` - integer (facility space ID)
- `filter[asset_id]` - integer (asset ID)
- `filter[status]` - string enum (see allowed values below)
- `filter[priority]` - string enum (`high`, `normal`, `low`)
- `filter[created_at]` - date or datetime string (`YYYY-MM-DD` or ISO datetime)
- `filter[closed_at]` - date or datetime string (`YYYY-MM-DD` or ISO datetime)

### `filter[status]` available options

- `open`
- `closed`

Examples:

```http
GET /tickets?filter[status]=open
GET /tickets?filter[status]=closed
```

### `filter[search]` behavior

`search` has special behavior:

1. Comma-separated numeric list (no spaces required): exact match by ticket IDs.
Input format: `number,number,number`

Example:

```http
GET /tickets?filter[search]=1,2,3
```

2. Any other single value (text/number/phrase) uses normal text search.
Input format: any string

Search covers:
- ticket ID
- title
- description
- ticket type
- priority
- status
- facility name
- asset name
- ticket category name
- owner name

Examples:

```http
GET /tickets?filter[search]=Broken AC
GET /tickets?filter[search]=Riverside
```

### Supported sorts

- `sort=id`
- `sort=created_at`
- `sort=closed_at`
- `sort=priority`
- `sort=status`

Sort format:

- Ascending: `sort=<field>`
- Descending: `sort=-<field>`

Examples:

```http
GET /tickets?sort=priority
GET /tickets?sort=-created_at
GET /tickets?sort=-closed_at
```

### Combined examples

```http
GET /tickets?filter[status]=open&filter[facility_id]=2&sort=-created_at&per_page=25
GET /tickets?filter[search]=1,2,3&sort=-id&per_page=50
GET /tickets?filter[search]=Broken AC&filter[priority]=high&sort=priority
```

## Procurement Request (`/procurement/requests`)

### Supported filters

- `filter[search]` - string
- `filter[ticket_id]` - integer (ticket ID)
- `filter[facility_id]` - integer (facility ID)
- `filter[asset_id]` - integer (asset ID)
- `filter[expense_type_id]` - integer (expense type ID)
- `filter[expense_sub_type_id]` - integer (expense sub-type ID)
- `filter[priority]` - string enum (`high`, `normal`, `low`)
- `filter[type]` - string enum (`work`, `purchase`)
- `filter[status]` - string enum (see allowed values below)
- `filter[is_emergency]` - boolean (`true` or `false`)
- `filter[require_site_visit]` - boolean (`true` or `false`)
- `filter[deadline]` - date or datetime string (`YYYY-MM-DD` or ISO datetime)
- `filter[created_at]` - date or datetime string (`YYYY-MM-DD` or ISO datetime)

### `filter[status]` available options

- `new`
- `request`
- `order`
- `quotes-review`
- `lpo`
- `rejected`

Examples:

```http
GET /procurement/requests?filter[status]=new
GET /procurement/requests?filter[status]=quotes-review
```

### `filter[search]` behavior

`search` has special behavior:

1. Comma-separated numeric list (no spaces required): exact match by procurement request IDs.
Input format: `number,number,number`

Example:

```http
GET /procurement/requests?filter[search]=1,2,3
```

2. Any other single value (text/number/phrase) uses normal text search.
Input format: any string

Search covers:
- request ID
- title
- description
- type
- priority
- status
- deadline
- facility name
- asset name
- expense type name
- expense sub-type name
- owner name

Examples:

```http
GET /procurement/requests?filter[search]=Generator
GET /procurement/requests?filter[search]=Riverside
```

### Supported sorts

- `sort=id`
- `sort=created_at`
- `sort=deadline`
- `sort=priority`
- `sort=status`

Sort format:

- Ascending: `sort=<field>`
- Descending: `sort=-<field>`

Examples:

```http
GET /procurement/requests?sort=deadline
GET /procurement/requests?sort=-created_at
GET /procurement/requests?sort=-priority
```

### Combined examples

```http
GET /procurement/requests?filter[status]=request&filter[type]=work&sort=-deadline&per_page=25
GET /procurement/requests?filter[search]=1,2,3&sort=-id&per_page=50
GET /procurement/requests?filter[search]=Generator&filter[is_emergency]=true&sort=-created_at
```

## Procurement Request Bid (`/procurement/requests/{procurementRequest}/bids`)

### Supported filters

- `filter[search]` - string
- `filter[vendor_id]` - integer (vendor user ID)
- `filter[currency_id]` - integer (currency ID)
- `filter[status]` - string enum (see allowed values below)
- `filter[delivery_at]` - date or datetime string (`YYYY-MM-DD` or ISO datetime)
- `filter[amount]` - number
- `filter[total]` - number
- `filter[created_at]` - date or datetime string (`YYYY-MM-DD` or ISO datetime)

### `filter[status]` available options

- `bid`
- `recommended`
- `selected`
- `rejected`

Examples:

```http
GET /procurement/requests/10/bids?filter[status]=bid
GET /procurement/requests/10/bids?filter[status]=selected
```

### `filter[search]` behavior

`search` has special behavior:

1. Numbers only, one or comma-separated: an exact match on the bid id or the final total (never
the amount before discount or VAT). An amount written as money (`KES 11,600`, `11600.00`) matches
the total alone. See [Searching by money](#searching-by-money-the-final-total-only).
Input format: `number`, `number,number,number` or an amount

Example:

```http
GET /procurement/requests/10/bids?filter[search]=1,2,3
GET /procurement/requests/10/bids?filter[search]=11600
```

2. Any other single value (text/number/phrase) uses normal text search.
Input format: any string

Search covers:
- bid ID
- notes
- status
- delivery date
- vendor name
- procurement request title
- currency code

Examples:

```http
GET /procurement/requests/10/bids?filter[search]=Acme
GET /procurement/requests/10/bids?filter[search]=KES
```

### Supported sorts

- `sort=id`
- `sort=created_at`
- `sort=delivery_at`
- `sort=amount`
- `sort=total`
- `sort=status`

Sort format:

- Ascending: `sort=<field>`
- Descending: `sort=-<field>`

Examples:

```http
GET /procurement/requests/10/bids?sort=delivery_at
GET /procurement/requests/10/bids?sort=-created_at
GET /procurement/requests/10/bids?sort=-amount
```

### Combined examples

```http
GET /procurement/requests/10/bids?filter[status]=recommended&filter[currency_id]=1&sort=-delivery_at&per_page=25
GET /procurement/requests/10/bids?filter[search]=1,2,3&sort=-id&per_page=50
GET /procurement/requests/10/bids?filter[search]=Acme&filter[status]=recommended&sort=-created_at
```

## Procurement LPO (`/procurement/lpos`)

### Supported filters

- `filter[search]` - string
- `filter[vendor_id]` - integer (vendor user ID)
- `filter[status]` - string enum (see allowed values below)
- `filter[delivered_at]` - date or datetime string (`YYYY-MM-DD` or ISO datetime)
- `filter[rating]` - number
- `filter[amount]` - number
- `filter[total]` - number
- `filter[created_at]` - date or datetime string (`YYYY-MM-DD` or ISO datetime)

### `filter[status]` available options

- `lpo`
- `delivered`
- `cancelled`

Examples:

```http
GET /procurement/lpos?filter[status]=lpo
GET /procurement/lpos?filter[status]=delivered
```

### `filter[search]` behavior

`search` has special behavior:

1. Numbers only, one or comma-separated: an exact match on the LPO id, the document number or the
final total (never the amount before discount or VAT). An amount written as money
(`KES 11,600`, `11600.00`) matches the total alone. See
[Searching by money](#searching-by-money-the-final-total-only).
Input format: `number`, `number,number,number` or an amount

Example:

```http
GET /procurement/lpos?filter[search]=1,2,3
GET /procurement/lpos?filter[search]=KES%2011%2C600
```

2. Any other single value (text/number/phrase) uses normal text search.
Input format: any string

Search covers:
- LPO ID
- document number
- notes
- status
- delivery date
- delivered date
- vendor name
- procurement request title
- currency code

Examples:

```http
GET /procurement/lpos?filter[search]=LPO-001
GET /procurement/lpos?filter[search]=Acme
```

### Supported sorts

- `sort=id`
- `sort=created_at`
- `sort=delivered_at`
- `sort=amount`
- `sort=total`
- `sort=rating`
- `sort=status`

Sort format:

- Ascending: `sort=<field>`
- Descending: `sort=-<field>`

Examples:

```http
GET /procurement/lpos?sort=delivered_at
GET /procurement/lpos?sort=-created_at
GET /procurement/lpos?sort=-amount
```

### Combined examples

```http
GET /procurement/lpos?filter[status]=lpo&filter[delivered_at]=2026-03-01&sort=-delivered_at&per_page=25
GET /procurement/lpos?filter[search]=1,2,3&sort=-id&per_page=50
GET /procurement/lpos?filter[search]=Acme&filter[vendor_id]=12&sort=-created_at
```

## Utility Meter (`/facilities/utility-meters`)

### Supported filters

- `filter[search]` - string
- `filter[meter_number]` - string
- `filter[meter_serial_number]` - string
- `filter[name]` - string
- `filter[utility_id]` - integer (lease component ID)
- `filter[facility_id]` - integer (facility ID)
- `filter[reading_unit_id]` - integer (SKU ID)
- `filter[status]` - string enum (see allowed values below)
- `filter[created_at]` - date or datetime string (`YYYY-MM-DD` or ISO datetime)

### `filter[status]` available options

- `active`
- `inactive`

Examples:

```http
GET /facilities/utility-meters?filter[status]=active
GET /facilities/utility-meters?filter[status]=inactive
```

### `filter[search]` behavior

`search` has special behavior:

1. Comma-separated numeric list (no spaces required): exact match by utility meter IDs.
Input format: `number,number,number`

Example:

```http
GET /facilities/utility-meters?filter[search]=1,2,3
```

2. Any other single value (text/number/phrase) uses normal text search.
Input format: any string

Search covers:
- meter ID
- meter number
- meter serial number
- name
- status
- facility name
- utility (lease component) name

Examples:

```http
GET /facilities/utility-meters?filter[search]=SN-987654322
GET /facilities/utility-meters?filter[search]=Electricity
```

### Supported sorts

- `sort=meter_number`
- `sort=meter_serial_number`
- `sort=name`
- `sort=utility_id`
- `sort=facility_id`
- `sort=reading_unit_id`
- `sort=status`
- `sort=created_at`

Sort format:

- Ascending: `sort=<field>`
- Descending: `sort=-<field>`

Examples:

```http
GET /facilities/utility-meters?sort=name
GET /facilities/utility-meters?sort=-created_at
```

### Combined examples

```http
GET /facilities/utility-meters?filter[facility_id]=2&filter[status]=active&sort=name&per_page=25
GET /facilities/utility-meters?filter[search]=1,2,3&sort=-created_at&per_page=50
GET /facilities/utility-meters?filter[search]=Electricity&filter[utility_id]=6&sort=name
```

## Facility Budget (`/finance/budgets`)

### Supported filters

- `filter[search]` - string
- `filter[facility_id]` - integer (facility ID)
- `filter[status]` - string enum (see allowed values below)
- `filter[period]` - date string (`YYYY-MM-DD`); matches budgets whose period contains the date

### `filter[status]` available options

- `ok`
- `at-risk`

Examples:

```http
GET /finance/budgets?filter[status]=ok
GET /finance/budgets?filter[status]=at-risk
```

### `filter[search]` behavior

`search` has special behavior:

1. Comma-separated numeric list (no spaces required): exact match by budget IDs.
Input format: `number,number,number`

Example:

```http
GET /finance/budgets?filter[search]=1,2,3
```

2. Any other single value (text/number/phrase) uses normal text search.
Input format: any string

Search covers:
- budget ID
- facility name
- currency code
- status
- period start / period end dates

Examples:

```http
GET /finance/budgets?filter[search]=Riverside
GET /finance/budgets?filter[search]=KES
```

### Supported sorts

- `sort=id`
- `sort=period_start`
- `sort=period_end`
- `sort=income_budget`
- `sort=expense_budget`
- `sort=created_at`

Sort format:

- Ascending: `sort=<field>`
- Descending: `sort=-<field>`

Examples:

```http
GET /finance/budgets?sort=period_start
GET /finance/budgets?sort=-created_at
```

### Combined examples

```http
GET /finance/budgets?filter[facility_id]=2&filter[status]=at-risk&sort=-period_start&per_page=25
GET /finance/budgets?filter[search]=1,2,3&sort=-period_start&per_page=50
GET /finance/budgets?filter[search]=Riverside&filter[status]=ok&sort=-created_at
```

## Add New Models To This Doc

When adding search/sort support for another model, append a new section using this template:

```md
## <Model Name> (`<endpoint>`)

### Supported filters
- `filter[...]`

### `filter[search]` behavior
- Exact-match rules (if any)
- Full-text behavior

### Supported sorts
- `sort=...`

### Combined examples
- Real request examples
```



