# List filters

Every list's filters in one place, generated from the code. Sent as `filter[<name>]=<value>`.

## Conventions

- **Exact** filters match the whole value. A comma list names several: `filter[facility_id]=6,7`. Every
  id, status and type filter is exact, so `filter[facility_id]=6` no longer returns property 60.
- **Range** filters take two dates, inclusive of the whole end day:
  `filter[start_between]=2026-09-01,2026-09-30`. A range is named after its field with `_at`
  dropped (`start_at` → `start_between`, `created_at` → `created_between`); a field not ending in
  `_at` keeps its name (`transaction_date` → `transaction_date_between`). A backwards, incomplete
  or malformed range is a `422` keyed `filter.<name>`, not an empty list.
- **Single date** filters are the ones lists already had, unchanged. Most match by the date's
  text, so `2026-09-15` is that day and `2026-09` is that month; on the access-management lists
  they take a whole date only. Use the range for anything new.
- **From/to** filters (bills and the statements) take `filter[<field>][from]=…&filter[<field>][to]=…`
  as before, and now have a `_between` filter beside them as well.
- **Text** filters match part of the value (`LIKE %value%`): names, emails, numbers typed into a
  search box. `filter[type]` on uploads is one on purpose: it is a MIME type, so `pdf` finds
  `application/pdf`.
- **Other** filters are computed (`search`, `pending_with_me`, …); see each list's own page.

`tests/Feature/ListFiltersTest.php` reads every list's filters and fails on a partial id, status
or type filter, or a date filter without its range, so these conventions hold for new lists too.

## Staff app (`/api/v1/app/{company}/…`)

| List | Exact | Range | Single date | Text | Other |
|---|---|---|---|---|---|
| `access-management/approval-templates` | `model_type`, `is_active` | `created_between` | `created_at` | `name` | — |
| `access-management/companies` | `status` | `created_between` | `created_at` | — | `search` |
| `access-management/company-departments` | `status` | `created_between` | `created_at` | — | `search`, `company_id` |
| `access-management/company-departments/{companyDepartment}/divisions` | `status` | `created_between` | `created_at` | — | `search`, `company_id` |
| `access-management/company-offices` | `country_id`, `city_id`, `status` | `created_between` | `created_at` | — | `search`, `company_id` |
| `access-management/company-users` | `role`, `status`, `company_office_id`, `company_department_id`, `company_department_division_id` | `created_between` | `created_at` | — | `search`, `company_id` |
| `access-management/messaging-channels` | `channel`, `driver`, `status` | — | — | — | — |
| `access-management/reports/system/audit-trail` | `subject_type`, `event` | — | — | — | `user_id`, `created_between` |
| `access-management/responsibility-transfers` | `status`, `is_permanent`, `from_user_id`, `to_user_id` | `created_between` | `created_at` | — | `direction` |
| `access-management/roles` | `id`, `user_group_id`, `enforce_on_facility` | `created_between` | `created_at` | — | `search`, `company_id` |
| `access-management/user-groups` | `id` | — | — | `title`, `description` | — |
| `facility-management/assets` | `id`, `facility_id`, `asset_type_id`, `status`, `warranty_status` | `created_between` | `created_at` | `name` | — |
| `facility-management/assets/insurance/policies` | `id`, `created_by_id` | `start_date_between`, `end_date_between` | `start_date`, `end_date` | `policy_number` | — |
| `facility-management/assets/insurance/providers` | — | — | — | `name`, `insurance_company_name`, `contact_person` | — |
| `facility-management/fleet/bookings` | `id`, `trip_id`, `user_id`, `pickup_city_id`, `status`, `trip.driver_id` | `pickup_between` | `pickup_at` | `luggage_size`, `has_luggage` | — |
| `facility-management/fleet/drivers` | `id`, `facility_id` | — | — | `user.name`, `user.email`, `user.phone`, `facility.name` | — |
| `facility-management/fleet/trips` | `id`, `vehicle_id`, `driver_id`, `departure_city_id`, `destination_city_id`, `status`, `stops.city_id` | `depart_between`, `arrive_between` | `depart_at`, `arrive_at` | `departure.name`, `destination.name`, `vehicle.license_plate`, `driver.name` | — |
| `facility-management/fleet/trips/{trip}/stops` | `city_id` | — | — | `city.name` | — |
| `facility-management/fleet/vehicles` | `id`, `vehicle_type`, `ownership_type`, `vehicle_make_id`, `vehicle_category_id`, `facility_id` | — | — | `license_plate`, `description`, `facility.name` | — |
| `facility-management/power-management/power-sources` | `power_source_type_id`, `asset_id`, `maintenance_contract_id`, `status`, `activity_status` | `commissioned_between`, `created_between` | `commissioned_at`, `created_at` | `name`, `co2_per_kwh`, `max_generation_capacity`, `has_maintenance_contract` | — |
| `facility-management/power-management/power-sources/{powerSource}/activity-logs` | `id`, `power_source_id` | `start_between`, `stop_between` | `start_at`, `stop_at` | `power_generated` | — |
| `facility-management/sensors` | `asset_id`, `facility_id`, `facility_space_id`, `type`, `current_status` | — | — | — | — |
| `facility-management/sensors/available-devices` | `facility_id` | — | — | — | — |
| `facility-management/sensors/{sensor}/actions` | — | — | — | `action_trigger`, `action_to_take` | — |
| `facility-management/settings/utility-management/utilities` | `id`, `utility_type`, `stock_keeping_unit_id` | `created_between` | `created_at` | `name`, `description`, `is_regulated`, `default_price_per_unit` | — |
| `facility-management/settings/utility-management/utilities/{utility}/prices` | `currency_id`, `facility_id` | `created_between` | `created_at` | `price_per_unit` | — |
| `project-management/projects` | `id`, `status`, `facility_id` | `created_between`, `initiated_between`, `estimated_start_date_between`, `completed_between`, `estimated_end_date_between` | `created_at`, `initiated_at`, `estimated_start_date`, `completed_at`, `estimated_end_date` | `title`, `total_budget` | — |
| `project-management/projects/{project}/milestones` | `status` | — | — | `title` | — |
| `project-management/projects/{project}/milestones/{milestone}/tasks` | `status`, `assigned_to_id` | — | — | `title`, `priority`, `estimated_duration_in_days` | — |
| `project-management/projects/{project}/updates` | — | — | — | `title`, `description` | — |
| `property-management/facilities` | `status`, `landlord_id`, `facility_type_id`, `country_id`, `city_id` | `created_between` | `created_at` | — | `search` |
| `property-management/facilities/blocks/{block}/floors` | `id` | — | — | `name`, `is_parking`, `is_divisible`, `common_space`, `leasable_space`, `landlord_space`, `total_space` | — |
| `property-management/facilities/blocks/{block}/wings` | `id` | `created_between` | `created_at` | `name` | — |
| `property-management/facilities/utility-meters` | `lease_id`, `utility_id`, `facility_id`, `reading_unit_id`, `status` | `created_between` | `created_at` | `meter_number`, `meter_serial_number`, `name`, `is_faulty` | `search` |
| `property-management/facilities/utility-meters/{meter}/readings` | `utility_meter_id` | `created_between` | `created_at` | `current_reading`, `is_faulty` | — |
| `property-management/facilities/wings/{wing}/floors` | `id` | — | — | `name`, `is_parking`, `is_divisible`, `common_space`, `leasable_space`, `landlord_space`, `total_space` | — |
| `property-management/facilities/{facility}/bank-accounts` | `bank_account_id` | — | — | `purpose`, `is_active` | — |
| `property-management/facilities/{facility}/blocks` | `id` | `created_between` | `created_at` | `name` | — |
| `property-management/facilities/{facility}/floors` | `id` | — | — | `name`, `is_parking`, `is_divisible`, `common_space`, `leasable_space`, `landlord_space`, `total_space` | — |
| `property-management/facilities/{facility}/spaces` | `status`, `name`, `space_type`, `size`, `is_parking`, `is_signage`, `parking_type` | — | — | — | — |
| `property-management/facilities/{facility}/utilities` | `utility_id`, `vendor_id` | `created_between` | `created_at` | `utility_bill_distribution_method` | — |
| `property-management/facilities/{facility}/utility-prices` | `utility_id`, `currency_id` | `created_between` | `created_at` | `price_per_unit` | — |
| `property-management/facilities/{facility}/wings` | `id` | `created_between` | `created_at` | `name` | — |
| `property-management/facility-spaces/spaces` | `id`, `space_type`, `size`, `status`, `facility_floor_id` | `created_between` | `created_at` | `name` | — |
| `property-management/facility-types` | `id`, `division_type` | `created_between` | `created_at` | `title`, `has_tax` | `search` |
| `property-management/finance/bills` | `invoice_validation_status`, `cu_validation_status`, `facility_id`, `vendor_id`, `expense_type_id`, `expense_category_id`, `type`, `status` | `invoice_date_between`, `created_between`, `invoice_uploaded_between`, `expense_posted_between` | `invoice_date (from/to)`, `created_at (from/to)`, `invoice_uploaded_at (from/to)`, `expense_posted_at (from/to)` | — | `search`, `contract_id`, `contract_type`, `lpo_number`, `landlord_id`, `payable` |
| `property-management/finance/bills/liabilities` | `vendor_id`, `facility_id`, `expense_type_id`, `expense_category_id`, `withholding_tax_id` | `expense_posted_between` | `expense_posted_at (from/to)` | — | — |
| `property-management/finance/budgets` | `facility_id`, `status` | — | — | — | `search`, `period` |
| `property-management/finance/expenses` | `facility_id`, `vendor_id`, `expense_type_id`, `expense_sub_type_id`, `expense_category_id`, `status` | `created_between`, `transaction_between` | `created_at`, `transaction_at` | — | `search` |
| `property-management/finance/payment-vouchers` | `payable_user_id`, `credit_bank_account_id`, `debit_bank_account_id`, `payment_method_id`, `payable_as`, `status` | `created_between`, `released_between`, `paid_between` | `created_at`, `released_at`, `paid_at` | — | `search` |
| `property-management/finance/remittances` | `facility_id`, `landlord_id`, `status` | `created_between`, `period_from_between`, `period_to_between` | `created_at`, `period_from`, `period_to` | — | `search` |
| `property-management/finance/settlements` | `debit_bank_account_id`, `payment_method_id`, `status` | `created_between` | `created_at` | — | `search` |
| `property-management/finance/vendor-statements` | `vendor_id`, `facility_contract_id`, `facility_id` | `transaction_between` | `transaction_at (from/to)` | — | — |
| `property-management/frc-sanctions` | `id`, `status`, `regime`, `category`, `source` | `listed_between`, `delisted_between` | — | — | `search`, `has_matches` |
| `property-management/frc-sanctions/audits` | `id`, `trigger` | `created_between` | — | — | — |
| `property-management/frc-sanctions/matches` | `id`, `subject_type`, `frc_sanction_id` | `flagged_between` | — | — | `search`, `regime`, `current` (default `true`) |
| `property-management/frc-sanctions/notices` | `id`, `status`, `regime` | `created_between` | — | — | — |
| `property-management/lease-management/charges/utility-billings` | `id`, `facility_id`, `utility_id`, `created_by_id` | `created_between`, `bill_date_between` | `created_at`, `bill_date` | — | — |
| `property-management/lease-management/credit-notes` | `facility_id`, `lease_id`, `status`, `cu_validation_status` | `due_between`, `created_between` | `due_at`, `created_at` | — | `search`, `etr_status`, `reversed` |
| `property-management/lease-management/exit-notices` | `id`, `lease_id`, `initiated_by_id`, `status`, `tenant_id`, `facility_id`, `lease.user_id`, `lease.facility_id` | `exit_date_between`, `created_between` | `exit_date`, `created_at` | — | `search` |
| `property-management/lease-management/invoices` | `facility_id`, `lease_id`, `status`, `cu_validation_status`, `user_id` | `created_between`, `due_between` | `due_at`, `created_at` | — | `search`, `etr_status`, `reversed`, `reissued` |
| `property-management/lease-management/lease-applications` | `applicant_type`, `status`, `user_id`, `facility_type_id`, `city_id`, `applicant_registered_country_id`, `registration_type`, `is_renewal`, `lease_id`, `referral_source` | `application_submitted_between`, `proposed_start_date_between`, `reviewed_between`, `created_between` | `application_submitted_at`, `proposed_start_date`, `reviewed_at`, `created_at` | `parking_required`, `generator_required` | `search` |
| `property-management/lease-management/lease-applications/{application}/escalations` | `lease_component_id` | — | — | `cycle` | — |
| `property-management/lease-management/lease-applications/{application}/spaces` | `facility_space_id` | — | — | — | — |
| `property-management/lease-management/leases` | `facility_id`, `user_id`, `status` | `created_between`, `start_between`, `end_between`, `next_due_between` | `created_at`, `start_at`, `end_at`, `next_due_at` | `billing_cycle` | `search` |
| `property-management/lease-management/leases/{lease}/billings` | `lease_id`, `invoice_id`, `lease_component_id` | `transaction_date_between` | `transaction_date` | `amount`, `tax`, `total` | — |
| `property-management/lease-management/leases/{lease}/collections` | `id`, `lease_id`, `receipt_allocation_id`, `lease_item_component_id`, `invoice_item_id` | `transaction_date_between` | `transaction_date` | `amount`, `tax`, `total` | — |
| `property-management/lease-management/leases/{lease}/deposits` | `id`, `lease_item_component_id` | `created_between` | `created_at` | — | — |
| `property-management/lease-management/leases/{lease}/escalations` | `id`, `lease_id`, `lease_component_id` | `start_between`, `next_due_between`, `created_between` | `start_at`, `next_due`, `created_at` | `cycle`, `period`, `rate` | — |
| `property-management/lease-management/leases/{lease}/items` | `lease_id`, `facility_space_id` | — | — | — | — |
| `property-management/lease-management/leases/{lease}/items/{leaseItem}/components` | `lease_item_id`, `lease_component_id`, `tax_id` | — | — | `cost_per_sqft`, `cost_per_month`, `hs_code` | — |
| `property-management/lease-management/leases/{lease}/opening-balances` | `id`, `lease_component_id` | `opening_balance_between`, `created_between` | `opening_balance_at`, `created_at` | — | — |
| `property-management/lease-management/loo-tags` | `category`, `is_computed` | — | — | `label`, `key` | — |
| `property-management/lease-management/loos` | `status`, `type`, `facility_id`, `loo_template_id`, `loo_preparation_type`, `loo_preparation_id` | — | — | `our_ref` | — |
| `property-management/lease-management/loos/{loo}/comments` | `attempt`, `comment_in` | — | — | — | `resolved`, `unresolved` |
| `property-management/lease-management/loos/{loo}/spaces` | `facility_space_id` | — | — | — | — |
| `property-management/lease-management/receipts` | `receiving_account_id`, `payment_method_id`, `facility_id`, `status` | `transaction_date_between`, `created_between` | `transaction_date`, `created_at` | — | `search` |
| `property-management/lease-management/tenant-statements` | `lease_id`, `tenant_id` | `transaction_between` | `transaction_at (from/to)` | — | — |
| `property-management/procurement/contracts` | `id`, `status`, `facility_id`, `vendor_id`, `expense_type_id`, `expense_category_id`, `expense_sub_type_id`, `asset_id`, `type`, `property_id`, `supplier_id`, `landlord_id` | `start_between`, `end_between`, `created_between` | `start_at`, `end_at`, `created_at` | `title` | `search` |
| `property-management/procurement/inventories` | `facility_id`, `inventory_category_id`, `storage_room_id`, `shelf_id` | — | — | `name`, `current_stock` | — |
| `property-management/procurement/inventories/categories` | `id`, `expense_type_id`, `expense_sub_type_id` | — | — | `name` | — |
| `property-management/procurement/inventories/purchase-items` | `id`, `inventory_category_id`, `currency_id`, `tax_id` | — | — | `name`, `base_price` | `search`, `vendor_id`, `facility_id` |
| `property-management/procurement/lpos` | `facility_id`, `is_direct`, `type`, `vendor_id`, `expense_category_id`, `expense_type_id`, `expense_sub_type_id`, `status` | `delivered_between`, `created_between` | `delivered_at`, `created_at` | `rating`, `amount`, `total` | `search` |
| `property-management/procurement/requests` | `ticket_id`, `facility_id`, `asset_id`, `expense_type_id`, `expense_category_id`, `expense_sub_type_id`, `type`, `status` | `deadline_between`, `created_between` | `deadline`, `created_at` | `priority`, `is_emergency`, `require_site_visit` | `search` |
| `property-management/procurement/requests/{procurementRequest}/bids` | `vendor_id`, `currency_id`, `status` | `delivery_between`, `created_between` | `delivery_at`, `created_at` | `amount`, `total` | `search` |
| `property-management/procurement/requests/{procurementRequest}/items` | `id`, `inventory_id`, `selected_vendor_id` | — | — | `quantity` | — |
| `property-management/settings/documents/templates` | `document_type`, `is_active`, `is_default` | — | — | `name` | `facility_id` |
| `property-management/settings/finance/banks/accounts` | `type`, `status`, `user_group_id`, `user_id`, `bank_branch_id` | `created_between` | `created_at` | — | `search` |
| `property-management/settings/finance/banks/accounts/{account}/reco-reports` | `report_type`, `difference_status`, `statement_token` | `created_between` | `created_at` | `period_start`, `period_end` | — |
| `property-management/settings/finance/banks/accounts/{account}/transactions` | — | `transaction_between` | `transaction_at (from/to)` | — | — |
| `property-management/settings/finance/expense-categories` | `id` | `created_between` | `created_at` | `name`, `description` | `search` |
| `property-management/settings/finance/expense-types` | `id`, `expense_category_id` | `created_between` | `created_at` | `name`, `description`, `is_procurable`, `can_have_default_vendor`, `can_have_preferred_vendor` | `search` |
| `property-management/settings/finance/expense-types/{type}/sub-types` | `id`, `default_vendor_id` | `created_between` | `created_at` | `name`, `description`, `is_procurable`, `can_have_default_vendor`, `can_have_preferred_vendor` | `search` |
| `property-management/settings/general` | — | — | — | `key`, `name`, `description`, `value` | `group` |
| `property-management/settings/lease-management/payment-priorities` | `lease_component_id`, `facility_id` | — | — | — | — |
| `property-management/settings/loo-templates` | `is_active`, `is_default` | — | — | `name` | `facility_id`, `facility_type_id` |
| `property-management/settings/procurement/request-steps-template-groups` | — | `created_between` | `created_at` | `description`, `title`, `is_default`, `facilities` | — |
| `property-management/settings/tickets/categories` | `id`, `status`, `priority`, `expense_type_id`, `expense_sub_type_id`, `is_procurable` | `created_between` | `created_at` | `name`, `description` | — |
| `property-management/tickets` | `ticket_type`, `ticket_category_id`, `facility_id`, `facility_space_id`, `asset_id`, `status` | `created_between`, `closed_between` | `created_at`, `closed_at` | `priority` | `search` |
| `property-management/tickets/categories` | `id`, `priority`, `is_procurable` | — | — | `name` | — |
| `users/landlords` | `id` | `created_between` | `created_at` | `name`, `email`, `phone` | `search` |
| `users/landlords/{landlord}/management-contracts` | `id`, `facility_id`, `management_fee_type`, `letting_fee_type`, `re_letting_fee_type` | `created_between` | `created_at` | — | — |
| `users/tenants` | `id` | `created_between` | `created_at` | `name`, `email`, `phone` | `search`, `facility_id` |
| `users/vendors` | `id`, `country_id`, `city_id`, `status`, `business_registration_number`, `is_tax_merchant` | `created_between` | `created_at` | `name`, `email`, `phone` | `search` |

## Tenant portal

| List | Exact | Range | Single date | Text | Other |
|---|---|---|---|---|---|
| `tenant/billing/credit-notes` | `facility_id`, `lease_id`, `status` | `due_between`, `created_between` | `due_at`, `created_at` | — | `search` |
| `tenant/billing/invoices` | `facility_id`, `lease_id`, `status` | `due_between`, `created_between` | `due_at`, `created_at` | — | `search` |
| `tenant/billing/receipts` | `receiving_account_id`, `payment_method_id`, `facility_id`, `status` | `transaction_date_between`, `created_between` | `transaction_date`, `created_at` | — | `search` |
| `tenant/facilities` | `status`, `landlord_id`, `facility_type_id`, `country_id`, `city_id` | `created_between` | `created_at` | — | `search` |
| `tenant/facilities/{facility}/spaces` | `status`, `space_type`, `size` | — | — | `name` | — |
| `tenant/lease-applications` | `applicant_type`, `status`, `facility_type_id`, `city_id`, `applicant_registered_country_id`, `registration_type`, `is_renewal`, `lease_id`, `referral_source` | `application_submitted_between`, `proposed_start_date_between`, `reviewed_between`, `created_between` | `application_submitted_at`, `proposed_start_date`, `reviewed_at`, `created_at` | `parking_required`, `generator_required` | `search` |
| `tenant/leases` | `facility_id`, `user_id`, `status` | `created_between`, `start_between`, `end_between`, `next_due_between` | `created_at`, `start_at`, `end_at`, `next_due_at` | `billing_cycle` | `search` |
| `tenant/leases/{lease}/deposits` | `id`, `lease_item_component_id` | `created_between` | `created_at` | — | — |
| `tenant/leases/{lease}/escalations` | `id`, `lease_component_id` | `start_between`, `next_due_between`, `created_between` | `start_at`, `next_due`, `created_at` | `cycle`, `period`, `rate` | — |
| `tenant/leases/{lease}/exit-notices` | `id`, `lease_id`, `status`, `initiated_by_id`, `lease.user_id`, `lease.facility_id` | `exit_date_between`, `created_between` | `exit_date`, `created_at` | — | `search` |
| `tenant/leases/{lease}/opening-balances` | `id`, `lease_component_id` | `opening_balance_between`, `created_between` | `opening_balance_at`, `created_at` | — | — |
| `tenant/tenant-statements` | `lease_id`, `tenant_id` | `transaction_between` | `transaction_at (from/to)` | — | — |
| `tenant/tickets` | `ticket_type`, `ticket_category_id`, `facility_id`, `facility_space_id`, `asset_id`, `status` | `created_between`, `closed_between` | `created_at`, `closed_at` | `priority` | `search` |

## Landlord portal

| List | Exact | Range | Single date | Text | Other |
|---|---|---|---|---|---|
| `landlord/facilities` | `status`, `facility_type_id`, `country_id`, `city_id`, `reporting_currency_id` | `created_between`, `construction_date_between` | `created_at`, `construction_date` | `utility_bill_distribution_method`, `lease_component_allocation_priority_method` | `search` |
| `landlord/facilities/{facility}/collections` | `id`, `lease_id`, `receipt_allocation_id`, `lease_item_component_id`, `invoice_item_id`, `status` | `created_between`, `transaction_date_between` | `created_at`, `transaction_date` | — | `search` |
| `landlord/facilities/{facility}/expenses` | `expense_type_id`, `expense_category_id`, `vendor_id`, `status`, `currency_id`, `tax_id`, `bill_id` | `created_between`, `transaction_between` | `created_at`, `transaction_at` | — | `search` |
| `landlord/facilities/{facility}/leases` | `user_id`, `status` | `created_between`, `start_between`, `end_between`, `next_due_between` | `created_at`, `start_at`, `end_at`, `next_due_at` | `billing_cycle` | `search` |
| `landlord/finance/bank-accounts` | `status`, `bank_branch_id` | `created_between` | `created_at` | — | `search` |
| `landlord/finance/payment-vouchers` | `credit_bank_account_id`, `status` | `created_between`, `released_between`, `paid_between` | `created_at`, `released_at`, `paid_at` | — | `search` |
| `landlord/finance/remittances` | `facility_id`, `status` | `created_between`, `period_from_between`, `period_to_between` | `created_at`, `period_from`, `period_to` | `is_advance` | `search` |

## Supplier portal

| List | Exact | Range | Single date | Text | Other |
|---|---|---|---|---|---|
| `vendor/finance/bills` | `facility_id`, `vendor_id`, `expense_category_id`, `type`, `status` | `invoice_date_between`, `created_between`, `invoice_uploaded_between`, `expense_posted_between` | `invoice_date (from/to)`, `created_at (from/to)`, `invoice_uploaded_at (from/to)`, `expense_posted_at (from/to)` | — | `search`, `contract_id` |
| `vendor/finance/payment-vouchers` | `credit_bank_account_id`, `status` | `created_between`, `released_between`, `paid_between` | `created_at`, `released_at`, `paid_at` | — | `search` |
| `vendor/finance/vendor-statements` | `vendor_id`, `facility_contract_id`, `facility_id` | `transaction_between` | `transaction_at (from/to)` | — | — |
| `vendor/procurement/contracts` | `id`, `status`, `facility_id`, `expense_type_id`, `expense_category_id`, `expense_sub_type_id`, `asset_id` | `start_between`, `end_between`, `created_between` | `start_at`, `end_at`, `created_at` | `title` | — |
| `vendor/procurement/lpos` | `facility_id`, `is_direct`, `type`, `expense_category_id`, `status` | `delivered_between`, `created_between` | `delivered_at`, `created_at` | `rating`, `amount`, `total` | `search` |
| `vendor/procurement/purchase-items/{openJob}` | `type`, `id` | — | — | `name` | — |
| `vendor/procurement/rfq/open-jobs` | `facility_id`, `expense_type_id`, `expense_sub_type_id`, `type`, `status` | `deadline_between`, `created_between` | `deadline`, `created_at` | `priority`, `is_emergency`, `require_site_visit` | `search`, `not_quoted` |
| `vendor/procurement/rfq/quotes` | `procurement_request_id`, `currency_id`, `status` | `delivery_between`, `created_between` | `delivery_at`, `created_at` | `amount`, `total` | `search` |
| `vendor/procurement/rfq/site-visit-reports` | `procurement_request_id`, `technician_id` | `delivery_between`, `created_between` | `delivery_at`, `created_at` | — | `search` |
| `vendor/users` | `id` | `created_between` | `created_at` | `user.name`, `user.email`, `is_technician` | — |

## Shared (`/api/v1/settings/…`, listings, search)

| List | Exact | Range | Single date | Text | Other |
|---|---|---|---|---|---|
| `auth/portals` | `id` | — | — | `title`, `description` | — |
| `listings` | `id`, `facility_type_id`, `status`, `country_id`, `city_id` | `construction_date_between` | `construction_date` | `name`, `description` | `search` |
| `search/purchase-items/{expenseSubType}` | `type` | — | — | — | — |
| `settings/accounting/banks` | `status` | — | — | `name` | — |
| `settings/accounting/currencies` | `id` | — | — | `name`, `code` | — |
| `settings/accounting/payment-methods` | — | — | — | `name`, `code`, `description` | — |
| `settings/accounting/taxes` | `id` | — | — | `name`, `value`, `hs_code`, `is_discount_deductible`, `is_default` | — |
| `settings/accounting/withholdings` | `id` | — | — | `name`, `value`, `minimum_amount`, `cap_amount` | — |
| `settings/companies` | `id`, `status` | `created_between` | `created_at` | — | `search` |
| `settings/companies/{company}/users` | `id`, `role`, `status` | `created_between` | `created_at` | — | `search` |
| `settings/file-management/uploads` | `user_id` | `created_between` | `created_at` | `title`, `file_name`, `type`, `extension`, `size` | — |
| `settings/fleet/vehicle-categories` | `id` | `created_between` | `created_at` | `name` | — |
| `settings/fleet/vehicle-makes` | `id`, `vehicle_model_id` | `created_between` | `created_at` | `name`, `description` | — |
| `settings/fleet/vehicle-models` | `id` | `created_between` | `created_at` | `name`, `description` | — |
| `settings/lease-management/lease-components` | `tax_id`, `status` | — | — | `hs_code`, `name`, `is_autobilled`, `is_a_charge`, `is_land_fee_charge`, `is_utility_charge`, `is_legal_fees_deposit`, `is_deposit` | `search` |
| `settings/power-management/fuel-types` | `id`, `status` | `created_between` | `created_at` | `name`, `description`, `co2_per_kwh` | — |
| `settings/power-management/power-source-types` | `id`, `power_source_fuel_type` | `created_between` | `created_at` | `name`, `description` | — |
| `settings/procurement/inventory-categories` | `id`, `expense_type_id`, `expense_sub_type_id` | — | — | `name` | — |
| `settings/procurement/skus` | `id` | — | — | `name`, `symbol` | — |
| `settings/skus` | `id` | — | — | `name`, `symbol` | — |
| `settings/ticket-categories` | `id`, `priority`, `is_procurable` | — | — | `name` | — |
