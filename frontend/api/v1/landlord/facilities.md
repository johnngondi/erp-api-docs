# Landlord Facilities API

Base route:

`/api/v1/landlord`

## Facilities

- `GET /facilities`
- `GET /facilities/{facility}`

### List Filters

- `filter[search]`
- `filter[created_at]`
- `filter[construction_date]`
- `filter[status]`
- `filter[utility_bill_distribution_method]`
- `filter[lease_component_allocation_priority_method]`
- `filter[facility_type_id]`
- `filter[country_id]`
- `filter[city_id]`
- `filter[reporting_currency_id]`

### Enum Filter Options

- `filter[status]`: `pending`, `active`, `inactive`
- `filter[utility_bill_distribution_method]`: `utility provider rate`, `distribute to tenants`, `defined`
- `filter[lease_component_allocation_priority_method]`: `order`, `percentage`, `proportional`

### List Sorts

- `sort=id,name,leasable_space,common_space,landlord_space,created_at`

### Default Scope

- Only facilities owned by authenticated landlord are returned.

## Facility Expenses (Tab)

- `GET /facilities/{facility}/expenses`
- `GET /facilities/{facility}/expenses/{expense}`

### Filters

- `filter[search]`
- `filter[expense_type_id]`
- `filter[vendor_id]`
- `filter[status]`
- `filter[created_at]`
- `filter[transaction_at]`
- `filter[currency_id]`
- `filter[tax_id]`
- `filter[bill_id]`

### Enum Filter Options

- `filter[status]`: `pending`, `unpaid`, `partially-paid`, `paid`, `cancelled`

### Sorts

- `sort=id,invoice_number,amount,tax,total,paid,balance,transaction_at,created_at,updated_at`

### Eager Loaded

- `vendor`
- `expenseType`
- `currency`
- `taxType`
- `invoiceUpload`
- `bill`

## Facility Collections (Tab)

- `GET /facilities/{facility}/collections`
- `GET /facilities/{facility}/collections/{collection}`

### Filters

- `filter[search]`
- `filter[id]`
- `filter[lease_id]`
- `filter[status]`
- `filter[created_at]`
- `filter[transaction_date]`

### Enum Filter Options

- `filter[status]`: `confirmed`, `cancelled`

### Sorts

- `sort=id,amount,tax,total,transaction_date,created_at`

### Eager Loaded

- `lease`
- `lease.user`
- `lease.currency`
- `receiptAllocation.receipt`
- `invoiceItem.invoice`
- `leaseItemComponent`

## Facility Leases (Tab)

- `GET /facilities/{facility}/leases`
- `GET /facilities/{facility}/leases/{lease}`

### Filters

- `filter[search]`
- `filter[user_id]`
- `filter[status]`
- `filter[created_at]`
- `filter[start_at]`
- `filter[end_at]`
- `filter[next_due_at]`
- `filter[billing_cycle]`

### Enum Filter Options

- `filter[status]`: `active`, `suspended`, `terminated`
- `filter[billing_cycle]`: `biennial`, `annually`, `biannual`, `quarterly`, `monthly`, `weekly`, `daily`, `hourly`

### Sorts

- `sort=id,created_at,start_at,end_at,next_due_at`

### Eager Loaded

- `user`
- `currency`
- `leaseItems.facilitySpace`
- `leaseItems.leaseItemComponents.leaseComponent`

## Enum Sources

- Facilities status/options: facilities migrations (`create_facilities_table`, and related facility settings migrations).
- Expenses status options: `App\Enums\ExpenseStatus` and expenses migration.
- Collections status options: `create_lease_collections_table` migration.
- Lease status/options: `App\Enums\LeaseStatus`, `App\Data\LeaseData` validation, and leases migration.
