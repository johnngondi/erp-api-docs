# Vendor API: Searching and Sorting

This document is for frontend developers integrating vendor-facing list endpoints.

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

## Open Job (`/procurement/rfq/open-jobs`)

### Supported filters

- `filter[search]` - string
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
GET /procurement/rfq/open-jobs?filter[status]=order
GET /procurement/rfq/open-jobs?filter[status]=quotes-review
```

### `filter[search]` behavior

`search` has special behavior:

1. Comma-separated numeric list (no spaces required): exact match by procurement request IDs.
Input format: `number,number,number`

Example:

```http
GET /procurement/rfq/open-jobs?filter[search]=1,2,3
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
GET /procurement/rfq/open-jobs?filter[search]=Generator
GET /procurement/rfq/open-jobs?filter[search]=Riverside
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
GET /procurement/rfq/open-jobs?sort=deadline
GET /procurement/rfq/open-jobs?sort=-created_at
GET /procurement/rfq/open-jobs?sort=-priority
```

### Combined examples

```http
GET /procurement/rfq/open-jobs?filter[status]=order&filter[type]=work&sort=-deadline&per_page=25
GET /procurement/rfq/open-jobs?filter[search]=1,2,3&sort=-id&per_page=50
GET /procurement/rfq/open-jobs?filter[search]=Generator&filter[is_emergency]=true&sort=-created_at
```

## Quote (`/procurement/rfq/quotes`)

### Supported filters

- `filter[search]` - string
- `filter[procurement_request_id]` - integer (open job ID)
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
GET /procurement/rfq/quotes?filter[status]=bid
GET /procurement/rfq/quotes?filter[status]=selected
```

### `filter[search]` behavior

`search` has special behavior:

1. Comma-separated numeric list (no spaces required): exact match by quote IDs.
Input format: `number,number,number`

Example:

```http
GET /procurement/rfq/quotes?filter[search]=1,2,3
```

2. Any other single value (text/number/phrase) uses normal text search.
Input format: any string

Search covers:
- quote ID
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
GET /procurement/rfq/quotes?filter[search]=Acme
GET /procurement/rfq/quotes?filter[search]=KES
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
GET /procurement/rfq/quotes?sort=delivery_at
GET /procurement/rfq/quotes?sort=-created_at
GET /procurement/rfq/quotes?sort=-amount
```

### Combined examples

```http
GET /procurement/rfq/quotes?filter[status]=bid&filter[currency_id]=1&sort=-delivery_at&per_page=25
GET /procurement/rfq/quotes?filter[search]=1,2,3&sort=-id&per_page=50
GET /procurement/rfq/quotes?filter[search]=Acme&filter[status]=bid&sort=-created_at
```

## LPO (`/procurement/lpos`)

### Supported filters

- `filter[search]` - string
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
GET /procurement/lpos?filter[search]=Acme&filter[rating]=5&sort=-created_at
```

## Site Visit Report (`/procurement/rfq/site-visit-reports`)

### Supported filters

- `filter[search]` - string
- `filter[procurement_request_id]` - integer (open job ID)
- `filter[technician_id]` - integer (technician user ID)
- `filter[delivery_at]` - date or datetime string (`YYYY-MM-DD` or ISO datetime)
- `filter[created_at]` - date or datetime string (`YYYY-MM-DD` or ISO datetime)

### `filter[search]` behavior

`search` has special behavior:

1. Comma-separated numeric list (no spaces required): exact match by site visit report IDs.
Input format: `number,number,number`

Example:

```http
GET /procurement/rfq/site-visit-reports?filter[search]=1,2,3
```

2. Any other single value (text/number/phrase) uses normal text search.
Input format: any string

Search covers:
- report ID
- notes
- delivery date
- created date
- open job title
- technician name

Examples:

```http
GET /procurement/rfq/site-visit-reports?filter[search]=Leak check
GET /procurement/rfq/site-visit-reports?filter[search]=Collins
```

### Supported sorts

- `sort=id`
- `sort=created_at`
- `sort=delivery_at`

Sort format:

- Ascending: `sort=<field>`
- Descending: `sort=-<field>`

Examples:

```http
GET /procurement/rfq/site-visit-reports?sort=delivery_at
GET /procurement/rfq/site-visit-reports?sort=-created_at
GET /procurement/rfq/site-visit-reports?sort=-id
```

### Combined examples

```http
GET /procurement/rfq/site-visit-reports?filter[procurement_request_id]=10&filter[delivery_at]=2026-03-01&sort=-delivery_at&per_page=25
GET /procurement/rfq/site-visit-reports?filter[search]=1,2,3&sort=-id&per_page=50
GET /procurement/rfq/site-visit-reports?filter[search]=Collins&filter[technician_id]=9&sort=-created_at
```

## Add New Vendor Models To This Doc

When adding search/sort support for another vendor-facing model, append a new section using this template:

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

