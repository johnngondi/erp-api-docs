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

1. Comma-separated numeric list (no spaces required): treats each value as exact invoice reference.
Input format: `number,number,number`

Example:

```http
GET /lease-management/invoices?filter[search]=1,2,3
```

2. Comma-separated CU invoice codes (letters + digits), exact match.
Input format: `CODE1,CODE2`

Example:

```http
GET /lease-management/invoices?filter[search]=INV100,INV200
```

3. Any single value (text, number, phrase) uses normal text search.
Input format: any string

Examples:

```http
GET /lease-management/invoices?filter[search]=Collins
GET /lease-management/invoices?filter[search]=15000
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

1. Comma-separated numeric list (no spaces required): treats each value as exact credit note reference.
Input format: `number,number,number`

Example:

```http
GET /lease-management/credit-notes?filter[search]=1,2,3
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

3. Any single value (text, number, phrase) uses normal text search.
Input format: any string

Examples:

```http
GET /lease-management/credit-notes?filter[search]=Collins
GET /lease-management/credit-notes?filter[search]=15000
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

1. Comma-separated numeric list (no spaces required): exact match by receipt references.
Input format: `number,number,number`

This applies to:
- receipt IDs
- transaction numbers

Example:

```http
GET /lease-management/receipts?filter[search]=1,2,3
```

2. Comma-separated transaction code list (letters + digits), exact match.
Input format: `CODE1,CODE2`

Example:

```http
GET /lease-management/receipts?filter[search]=TRX100,TRX200
```

3. Any single value (text/number/phrase) uses normal text search.
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
GET /lease-management/receipts?filter[search]=15000
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

1. Comma-separated numeric list (no spaces required): exact match by bid IDs.
Input format: `number,number,number`

Example:

```http
GET /procurement/requests/10/bids?filter[search]=1,2,3
```

2. Any other single value (text/number/phrase) uses normal text search.
Input format: any string

Search covers:
- bid ID
- amount
- total
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

1. Comma-separated numeric list (no spaces required): exact match by LPO IDs.
Input format: `number,number,number`

Example:

```http
GET /procurement/lpos?filter[search]=1,2,3
```

2. Any other single value (text/number/phrase) uses normal text search.
Input format: any string

Search covers:
- LPO ID
- document number
- notes
- amount
- total
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



