# App API Request Contracts (DTO Reference)

This section lists request payload fields inferred from Data DTO classes used by App API controllers.

Legend:
- `Required`: must be sent in request
- `Optional`: may be omitted
- `Rules`: validation attributes and conditional requirements (`RequiredIf`, `RequiredUnless`, etc.)

## AssetData

Class: `App\Data\AssetData`

| Field | Required | Type | Allowed Values (Enum) | Rules |
|---|---|---|---|---|
| `name` | Yes | `string` | - | `Required` |
| `facility_id` | Yes | `int` | - | `Exists` |
| `facility_space_id` | Yes | `int` | - | `Exists` |
| `asset_type_id` | Yes | `int` | - | `Exists` |
| `purchased_at` | Yes | `string` | - | `Date` |
| `currency` | Yes | `string` | - | `Exists` |
| `purchase_price` | Yes | `int` | - | `Numeric` |
| `value_change_type` | Yes | `string` | `appreciate`, `depreciate` | `In` |
| `value_change_rate` | Yes | `float` | - | `Numeric` |
| `value_change_duration` | Yes | `string` | - | - |
| `last_maintenance_at` | Yes | `string` | - | `Date` |
| `current_value` | No | `Spatie\LaravelData\Optional|float` | - | `Numeric` |
| `last_value_change_at` | No | `Spatie\LaravelData\Optional|string|null` | - | `Date`, `RequiredUnless` |
| `warranty_status` | No | `Spatie\LaravelData\Optional|string|null` | `active`, `inactive` | `In` |
| `warranty_period_in_days` | No | `Spatie\LaravelData\Optional|float|null` | - | `Numeric` |
| `has_inventory` | No | `Spatie\LaravelData\Optional|bool` | - | - |
| `description` | No | `Spatie\LaravelData\Optional|string|null` | - | - |
| `photo` | No | `Spatie\LaravelData\Optional|string|null` | - | - |

## BankAccountData

Class: `App\Data\BankAccountData`

| Field | Required | Type | Allowed Values (Enum) | Rules |
|---|---|---|---|---|
| `bank_branch_id` | Yes | `int` | - | `Exists` |
| `account_name` | Yes | `string` | - | - |
| `account_number` | Yes | `string` | - | - |
| `type` | No | `?string` | `system`, `user` | `In` |
| `user_group_id` | No | `?int` | - | `Exists` |
| `user_id` | No | `?int` | - | `Exists` |
| `alias` | No | `Spatie\LaravelData\Optional|string|null` | - | - |
| `status` | No | `string` | `active`, `inactive` | `In` |

## BillData

Class: `App\Data\BillData`

| Field | Required | Type | Allowed Values (Enum) | Rules |
|---|---|---|---|---|
| `vendor_id` | Yes | `int` | - | `Exists` |
| `facility_id` | Yes | `int` | - | `Exists` |
| `expense_type_id` | Yes | `int` | - | `Exists` |
| `currency_id` | No | `?int` | - | `Exists` |
| `type` | No | `string` | `lpo`, `contract`, `liability`, `other` | `In` |
| `billable_type` | No | `?string` | - | - |
| `billable_id` | No | `?int` | - | - |
| `notes` | No | `?string` | - | - |
| `invoice_number` | No | `?string` | - | - |
| `invoice_date` | No | `?string` | - | `Date` |
| `invoice_upload_id` | No | `?int` | - | `Exists` |
| `tax_invoice_number` | No | `?string` | - | - |
| `invoice_uploaded_at` | No | `?string` | - | `Date` |
| `expense_posted_at` | No | `?string` | - | `Date` |
| `tax_regime` | No | `?string` | - | - |
| `items` | No | `array` | - | - |

## BookingData

Class: `App\Data\BookingData`

| Field | Required | Type | Allowed Values (Enum) | Rules |
|---|---|---|---|---|
| `trip_id` | Yes | `int` | - | `Exists` |
| `user_id` | Yes | `int` | - | `Exists` |
| `exact_pickup_location` | Yes | `string` | - | `Max` |
| `pickup_at` | Yes | `string` | - | `Date`, `AfterOrEqual` |
| `seats` | Yes | `int` | - | - |
| `has_luggage` | Yes | `bool` | - | - |
| `pickup_city_id` | No | `?int` | - | `Exists` |
| `luggage_size` | No | `?string` | `small`, `medium`, `large`, `huge` | `In` |
| `description` | No | `?string` | - | - |

## ContractData

Class: `App\Data\ContractData`

| Field | Required | Type | Allowed Values (Enum) | Rules |
|---|---|---|---|---|
| `title` | Yes | `?string` | - | `Nullable` |
| `notes` | Yes | `?string` | - | `Nullable` |
| `vendor_id` | Yes | `int` | - | `Exists` |
| `facility_id` | Yes | `int` | - | `Exists` |
| `start_at` | Yes | `string` | - | `Date` |
| `period_in_days` | Yes | `int` | - | - |
| `type` | Yes | `string` | `fixed`, `on-demand` | `In` |
| `amount` | Yes | `int` | - | - |
| `end_at` | No | `?string` | - | `Computed` |
| `currency_id` | No | `Spatie\LaravelData\Optional|int|null` | - | `Exists` |
| `expense_type_id` | No | `Spatie\LaravelData\Optional|int|null` | - | `Exists` |
| `expense_sub_type_id` | No | `Spatie\LaravelData\Optional|int|null` | - | `Exists` |
| `asset_id` | No | `Spatie\LaravelData\Optional|int|null` | - | `Exists` |
| `agreement_upload_id` | No | `Spatie\LaravelData\Optional|int|null` | - | `Exists` |
| `has_tax` | No | `bool` | - | - |
| `tax_id` | No | `Spatie\LaravelData\Optional|int|null` | - | `Exists` |
| `creator_id` | No | `Spatie\LaravelData\Optional|int|null` | - | `Exists` |
| `approver_id` | No | `Spatie\LaravelData\Optional|int|null` | - | `Exists` |
| `status` | No | `Spatie\LaravelData\Optional|string|null` | `pending`, `active`, `suspended`, `expired`, `inactive` | `In` |

## CreditNoteData

Class: `App\Data\CreditNoteData`

| Field | Required | Type | Allowed Values (Enum) | Rules |
|---|---|---|---|---|
| `lease_id` | Yes | `int` | - | `Required` |
| `due_at` | Yes | `string` | - | `Required`, `Date` |
| `notes` | Yes | `string` | - | `Required` |
| `items` | Yes | `array` | - | - |
| `invoice_id` | No | `?int` | - | - |
| `cu_reference_number` | No | `?string` | - | - |
| `is_credit` | No | `Spatie\LaravelData\Optional|bool|null` | - | - |
| `paid` | No | `Spatie\LaravelData\Optional|float|null` | - | - |
| `cu_invoice_number` | No | `?string` | - | - |
| `cu_serial_number` | No | `?string` | - | - |
| `cu_credit_note_verify_url` | No | `?string` | - | - |

## DriverData

Class: `App\Data\DriverData`

| Field | Required | Type | Allowed Values (Enum) | Rules |
|---|---|---|---|---|
| `name` | Yes | `string` | - | - |
| `phone` | Yes | `string` | - | - |
| `email` | Yes | `string` | - | - |
| `facility_id` | Yes | `int` | - | - |
| `experience` | Yes | `int` | - | - |
| `license_upload_id` | No | `?int` | - | - |
| `can_drive_manual` | No | `?bool` | - | - |
| `qualified_vehicle_categories` | No | `?array` | - | - |

## ExpenseData

Class: `App\Data\ExpenseData`

| Field | Required | Type | Allowed Values (Enum) | Rules |
|---|---|---|---|---|
| `vendor_id` | Yes | `int` | - | `Exists` |
| `facility_id` | Yes | `int` | - | `Exists` |
| `creating_user_id` | Yes | `?int` | - | `Exists` |
| `notes` | Yes | `?string` | - | - |
| `bill_id` | Yes | `?int` | - | `Exists` |
| `transaction_at` | Yes | `string` | - | `Date` |
| `expense_type_id` | Yes | `int` | - | `Exists` |
| `invoice_number` | Yes | `string` | - | - |
| `invoice_upload_id` | Yes | `?int` | - | `Exists` |
| `tax_invoice_number` | Yes | `?string` | - | - |
| `currency_id` | Yes | `?int` | - | `Exists` |
| `amount` | Yes | `float` | - | - |
| `tax_id` | Yes | `?int` | - | `Exists` |
| `paid` | No | `?float` | - | - |
| `status` | No | `string` | `pending`, `unpaid`, `partially-paid`, `paid`, `cancelled` | `In` |

## FacilityBlockData

Class: `App\Data\FacilityBlockData`

| Field | Required | Type | Allowed Values (Enum) | Rules |
|---|---|---|---|---|
| `name` | Yes | `string` | - | - |
| `has_wings` | No | `Spatie\LaravelData\Optional|bool` | - | - |
| `leasable_space` | No | `Spatie\LaravelData\Optional|int` | - | - |
| `common_space` | No | `Spatie\LaravelData\Optional|int` | - | - |
| `landlord_space` | No | `Spatie\LaravelData\Optional|int` | - | - |

## FacilitySpaceData

Class: `App\Data\FacilitySpaceData`

| Field | Required | Type | Allowed Values (Enum) | Rules |
|---|---|---|---|---|
| `name` | Yes | `string` | - | - |
| `facility_floor_id` | Yes | `int` | - | `Exists` |
| `space_type` | Yes | `string` | - | - |
| `size` | Yes | `int` | - | `Min` |
| `is_parking` | No | `?bool` | - | - |
| `is_signage` | No | `?bool` | - | - |

## InsurancePolicyData

Class: `App\Data\InsurancePolicyData`

| Field | Required | Type | Allowed Values (Enum) | Rules |
|---|---|---|---|---|
| `policy_number` | Yes | `string` | - | - |
| `provider_id` | Yes | `int` | - | `Exists` |
| `owner_id` | Yes | `int` | - | `Exists` |
| `negotiated_by` | Yes | `string` | - | - |
| `currency` | Yes | `string` | - | - |
| `premium_pa` | Yes | `float` | - | - |
| `coverage_amount` | Yes | `float` | - | - |
| `start_date` | Yes | `string` | - | `Date` |
| `duration_in_days` | Yes | `int` | - | - |
| `signed_by` | No | `Spatie\LaravelData\Optional|int|null` | - | `Exists` |
| `upload_id` | No | `Spatie\LaravelData\Optional|int|null` | - | `Exists` |
| `covered_assets` | No | `Spatie\LaravelData\Optional|array|null` | - | - |

## InsuranceProviderData

Class: `App\Data\InsuranceProviderData`

| Field | Required | Type | Allowed Values (Enum) | Rules |
|---|---|---|---|---|
| `name` | Yes | `string` | - | - |
| `contact_person` | Yes | `string` | - | - |
| `contact_phone` | Yes | `string` | - | - |
| `insurance_company_name` | Yes | `string` | - | - |
| `claim_email` | Yes | `string` | - | - |
| `claim_phone` | Yes | `string` | - | - |
| `email` | No | `?string` | - | - |
| `phone` | No | `?string` | - | - |
| `contact_email` | No | `?string` | - | - |

## InvoiceData

Class: `App\Data\InvoiceData`

| Field | Required | Type | Allowed Values (Enum) | Rules |
|---|---|---|---|---|
| `lease_id` | Yes | `int` | - | `Required` |
| `due_at` | Yes | `string` | - | `Required`, `Date` |
| `cu_reference_number` | Yes | `?string` | - | - |
| `is_credit` | No | `Spatie\LaravelData\Optional|bool|null` | - | - |
| `paid` | No | `Spatie\LaravelData\Optional|float|null` | - | - |
| `notes` | Yes | `string` | - | `Required` |
| `cu_invoice_number` | Yes | `?string` | - | - |
| `cu_serial_number` | Yes | `?string` | - | - |
| `cu_invoice_verify_url` | Yes | `?string` | - | - |
| `items` | Yes | `array` | - | - |

## LeaseApplicationReviewData

Class: `App\Data\LeaseApplicationReviewData`

| Field | Required | Type | Allowed Values (Enum) | Rules |
|---|---|---|---|---|
| `status` | No | `?string` | `pending`, `approved`, `review`, `rejected` | `In` |
| `comments` | No | `?string` | - | `RequiredIf` |
| `signed_agreement_upload_id` | No | `Spatie\LaravelData\Optional|int|null` | - | `Exists`, `RequiredIf` |

## LeaseComponentsPaymentPriorityData

Class: `App\Data\LeaseComponentsPaymentPriorityData`

| Field | Required | Type | Allowed Values (Enum) | Rules |
|---|---|---|---|---|
| `facility_id` | Yes | `int` | - | `Required`, `Numeric`, `Exists` |
| `lease_component_id` | Yes | `int` | - | `Required`, `Numeric`, `Exists` |
| `priority` | Yes | `int` | - | `Required`, `Numeric` |

## LeaseData

Class: `App\Data\LeaseData`

| Field | Required | Type | Allowed Values (Enum) | Rules |
|---|---|---|---|---|
| `user_id` | Yes | `int` | - | `Required`, `Exists` |
| `start_at` | Yes | `string` | - | `Required`, `Date` |
| `period_in_years` | Yes | `int` | - | `Required` |
| `facility_id` | Yes | `int` | - | `Required`, `Exists` |
| `period_in_months` | Yes | `int` | - | `Required` |
| `billing_cycle` | Yes | `string` | `biennial`, `annually`, `biannual`, `quarterly`, `monthly`, `weekly`, `daily`, `hourly` | `Required`, `In` |
| `next_due_at` | Yes | `string` | - | `Required`, `Date` |
| `status` | No | `Spatie\LaravelData\Optional|string|null` | `active`, `suspended`, `terminated` | `In` |
| `end_at` | No | `string` | - | - |
| `currency_id` | No | `?int` | - | `Exists` |

## LeaseDepositData

Class: `App\Data\LeaseDepositData`

| Field | Required | Type | Allowed Values (Enum) | Rules |
|---|---|---|---|---|
| `lease_item_component_id` | Yes | `int` | - | `Required`, `Exists` |
| `amount` | Yes | `float` | - | `Required` |
| `billed` | Yes | `?bool` | - | `Required` |

## LeaseEscalationData

Class: `App\Data\LeaseEscalationData`

| Field | Required | Type | Allowed Values (Enum) | Rules |
|---|---|---|---|---|
| `lease_component_id` | Yes | `int` | - | `Required`, `Exists` |
| `start_at` | Yes | `string` | - | `Required`, `Date` |
| `cycle` | Yes | `string` | `days`, `weeks`, `months`, `years` | `Required`, `In` |
| `period` | Yes | `int` | - | `Required` |
| `rate` | Yes | `int` | - | `Required`, `Numeric` |
| `next_due` | Yes | `string` | - | `Required`, `Date` |

## LeaseItemComponentData

Class: `App\Data\LeaseItemComponentData`

| Field | Required | Type | Allowed Values (Enum) | Rules |
|---|---|---|---|---|
| `lease_component_id` | Yes | `int` | - | `Exists` |
| `tax_id` | Yes | `int` | - | `Exists` |
| `cost_per_sqft` | Yes | `int` | - | `Required` |
| `cost_per_month` | No | `Spatie\LaravelData\Optional|int` | - | - |
| `hs_code` | No | `Spatie\LaravelData\Optional|string|null` | - | - |

## LeaseItemData

Class: `App\Data\LeaseItemData`

| Field | Required | Type | Allowed Values (Enum) | Rules |
|---|---|---|---|---|
| `facility_space_id` | Yes | `int` | - | `Required`, `Exists` |
| `components` | Yes | `array` | - | `Required` |

## LeaseOpeningBalanceData

Class: `App\Data\LeaseOpeningBalanceData`

| Field | Required | Type | Allowed Values (Enum) | Rules |
|---|---|---|---|---|
| `lease_component_id` | No | `Spatie\LaravelData\Optional|int` | - | `Exists` |
| `amount` | Yes | `float` | - | `Required`, `Numeric` |
| `opening_balance_at` | Yes | `string` | - | `Required`, `Date` |
| `notes` | No | `Spatie\LaravelData\Optional|string|null` | - | - |

## PaymentAdviceData

Class: `App\Data\PaymentAdviceData`

| Field | Required | Type | Allowed Values (Enum) | Rules |
|---|---|---|---|---|
| `payment_advice_number` | Yes | `string` | - | `Required` |
| `payment_advice_upload_id` | Yes | `int` | - | `Required`, `Exists` |

## PaymentVoucherData

Class: `App\Data\PaymentVoucherData`

| Field | Required | Type | Allowed Values (Enum) | Rules |
|---|---|---|---|---|
| `payable_user_id` | Yes | `int` | - | `Exists` |
| `payable_as` | Yes | `string` | `vendor`, `landlord`, `tenant`, `other` | `In` |
| `credit_bank_account_id` | Yes | `int` | - | `Exists` |
| `currency_id` | No | `?int` | - | `Exists` |
| `items` | No | `array` | - | - |

## PowerSourceActivityLogData

Class: `App\Data\PowerSourceActivityLogData`

| Field | Required | Type | Allowed Values (Enum) | Rules |
|---|---|---|---|---|
| `power_source_id` | No | `Spatie\LaravelData\Optional|int` | - | `Exists` |
| `start_at` | Yes | `string` | - | `Date` |
| `stop_at` | Yes | `string` | - | `Date` |
| `power_generated` | Yes | `float` | - | `Numeric` |

## PowerSourceData

Class: `App\Data\PowerSourceData`

| Field | Required | Type | Allowed Values (Enum) | Rules |
|---|---|---|---|---|
| `name` | Yes | `Spatie\LaravelData\Optional|string` | - | `Required` |
| `power_source_type_id` | No | `Spatie\LaravelData\Optional|int` | - | `Exists` |
| `facility_id` | No | `Spatie\LaravelData\Optional|int` | - | `Exists` |
| `co2_per_kwh` | No | `Spatie\LaravelData\Optional|float` | - | `Numeric` |
| `max_generation_capacity` | No | `Spatie\LaravelData\Optional|float` | - | - |
| `commissioned_at` | No | `Spatie\LaravelData\Optional|string` | - | `Date` |
| `description` | No | `Spatie\LaravelData\Optional|string|null` | - | - |
| `is_asset` | No | `Spatie\LaravelData\Optional|bool|null` | - | - |
| `asset_id` | No | `Spatie\LaravelData\Optional|int|null` | - | `Exists` |
| `has_maintenance_contract` | No | `Spatie\LaravelData\Optional|bool|null` | - | - |
| `maintenance_contract_id` | No | `Spatie\LaravelData\Optional|int|null` | - | `Exists` |
| `status` | No | `Spatie\LaravelData\Optional|string|null` | - | - |
| `activity_status` | No | `Spatie\LaravelData\Optional|string|null` | - | - |
| `suspended_by` | No | `Spatie\LaravelData\Optional|int|null` | - | `Exists` |
| `suspended_at` | No | `Spatie\LaravelData\Optional|string|null` | - | `Date` |
| `suspension_reason` | No | `Spatie\LaravelData\Optional|string|null` | - | - |
| `decommissioned_by` | No | `Spatie\LaravelData\Optional|int|null` | - | `Exists` |
| `decommissioned_at` | No | `Spatie\LaravelData\Optional|string|null` | - | `Date` |
| `decommission_reason` | No | `Spatie\LaravelData\Optional|string|null` | - | - |

## ProcurementRequestData

Class: `App\Data\ProcurementRequestData`

| Field | Required | Type | Allowed Values (Enum) | Rules |
|---|---|---|---|---|
| `title` | Yes | `string` | - | `Required` |
| `description` | Yes | `string` | - | `Required` |
| `type` | Yes | `string` | `purchase`, `work` | `In` |
| `expense_type_id` | Yes | `int` | - | `Exists` |
| `expense_sub_type_id` | Yes | `int` | - | `Exists` |
| `facility_id` | Yes | `int` | - | `Exists` |
| `ticket_id` | No | `Spatie\LaravelData\Optional|int|null` | - | `Exists` |
| `asset_id` | No | `Spatie\LaravelData\Optional|int|null` | - | `Exists` |
| `owner_id` | No | `Spatie\LaravelData\Optional|int|null` | - | `Exists` |
| `items` | No | `Spatie\LaravelData\Optional|array|null` | - | `DataCollectionOf`, `RequiredIf` |

## ProcurementRequestItemData

Class: `App\Data\ProcurementRequestItemData`

| Field | Required | Type | Allowed Values (Enum) | Rules |
|---|---|---|---|---|
| `purchase_item_id` | Yes | `int` | - | `Exists` |
| `quantity` | Yes | `int` | - | `Min` |

## ProcurementRequestStepData

Class: `App\Data\ProcurementRequestStepData`

| Field | Required | Type | Allowed Values (Enum) | Rules |
|---|---|---|---|---|
| `status` | Yes | `string` | - | - |
| `preferred_vendor_id` | Yes | `?int` | - | - |
| `selected_option` | Yes | `?int` | - | - |
| `deadline` | Yes | `?string` | - | - |
| `vendors` | Yes | `?array` | - | - |
| `comment` | Yes | `?string` | - | - |
| `has_preferred_vendor` | No | `bool` | - | - |
| `override` | No | `bool` | - | - |

## ProcurementRequestStepTemplateData

Class: `App\Data\ProcurementRequestStepTemplateData`

| Field | Required | Type | Allowed Values (Enum) | Rules |
|---|---|---|---|---|
| `title` | Yes | `string` | - | - |
| `role_id` | Yes | `int` | - | - |
| `description` | No | `Spatie\LaravelData\Optional|string|null` | - | - |
| `select_preferred_vendor` | No | `Spatie\LaravelData\Optional|bool|null` | - | - |
| `handle_procurement` | No | `Spatie\LaravelData\Optional|bool|null` | - | - |

## ProcurementRequestStepTemplateGroupData

Class: `App\Data\ProcurementRequestStepTemplateGroupData`

| Field | Required | Type | Allowed Values (Enum) | Rules |
|---|---|---|---|---|
| `title` | Yes | `string` | - | - |
| `description` | No | `Spatie\LaravelData\Optional|string|null` | - | - |
| `is_default` | No | `Spatie\LaravelData\Optional|bool|null` | - | - |
| `facilities` | No | `Spatie\LaravelData\Optional|array|null` | - | - |

## ProjectData

Class: `App\Data\ProjectData`

| Field | Required | Type | Allowed Values (Enum) | Rules |
|---|---|---|---|---|
| `facility_id` | Yes | `int` | - | `Exists` |
| `title` | Yes | `string` | - | - |
| `description` | Yes | `string` | - | - |
| `estimated_start_date` | Yes | `string` | - | `Date` |
| `estimated_duration_in_days` | Yes | `float` | - | `Numeric` |
| `currency_id` | No | `?int` | - | `Exists` |
| `total_budget` | No | `float` | - | `Numeric` |
| `project_manager_id` | No | `?int` | - | `Exists` |
| `initiated_at` | No | `?string` | - | `Date` |
| `completed_at` | No | `?string` | - | `Date` |
| `status` | No | `string` | `pending`, `in-progress`, `completed`, `stalled`, `cancelled` | `In` |

## ProjectMilestoneData

Class: `App\Data\ProjectMilestoneData`

| Field | Required | Type | Allowed Values (Enum) | Rules |
|---|---|---|---|---|
| `title` | Yes | `string` | - | - |
| `description` | No | `Spatie\LaravelData\Optional|string` | - | - |
| `percentage_of_budget` | Yes | `float` | - | `Numeric` |
| `estimated_start_date` | Yes | `string` | - | `Date` |
| `estimated_duration_in_days` | Yes | `float` | - | `Numeric` |
| `responsible_person_id` | No | `?int` | - | `Exists` |
| `started_at` | No | `?string` | - | `Date` |
| `completed_at` | No | `?string` | - | `Date` |

## ProjectTaskData

Class: `App\Data\ProjectTaskData`

| Field | Required | Type | Allowed Values (Enum) | Rules |
|---|---|---|---|---|
| `title` | Yes | `string` | - | - |
| `estimated_start_date` | Yes | `string` | - | `Date` |
| `estimated_duration_in_days` | Yes | `float` | - | `Numeric` |
| `description` | No | `Spatie\LaravelData\Optional|string|null` | - | - |
| `priority` | No | `Spatie\LaravelData\Optional|string|null` | `low`, `normal`, `high` | `In` |
| `assigned_to_id` | No | `?int` | - | `Exists` |

## ProjectUpdateData

Class: `App\Data\ProjectUpdateData`

| Field | Required | Type | Allowed Values (Enum) | Rules |
|---|---|---|---|---|
| `description` | Yes | `string` | - | - |
| `title` | No | `Spatie\LaravelData\Optional|string|null` | - | - |
| `attachment_upload_id` | No | `?int` | - | `Exists` |
| `added_by` | No | `string` | `user`, `system` | `In` |
| `added_by_id` | No | `?int` | - | `Exists` |
| `project_item_id` | No | `?int` | - | - |
| `project_item_type` | No | `?string` | - | - |

## PurchaseItemData

Class: `App\Data\PurchaseItemData`

| Field | Required | Type | Allowed Values (Enum) | Rules |
|---|---|---|---|---|
| `type` | Yes | `string` | `service`, `product` | `In` |
| `name` | Yes | `string` | - | `Unique` |
| `inventory_category_id` | Yes | `int` | - | `Exists` |
| `stock_keeping_unit_id` | No | `?int` | - | `Exists` |
| `tax_id` | No | `?int` | - | `Exists` |
| `currency_id` | No | `?int` | - | `Exists` |
| `is_regulated` | No | `?bool` | - | - |
| `base_price` | No | `?float` | - | - |
| `vendors` | No | `Spatie\LaravelData\Optional|array` | - | - |

## PurchaseItemPriceData

Class: `App\Data\PurchaseItemPriceData`

| Field | Required | Type | Allowed Values (Enum) | Rules |
|---|---|---|---|---|
| `price` | Yes | `int` | - | - |

## ReceiptData

Class: `App\Data\ReceiptData`

| Field | Required | Type | Allowed Values (Enum) | Rules |
|---|---|---|---|---|
| `transaction_number` | Yes | `string` | - | `Required` |
| `receiving_account_id` | Yes | `int` | - | `Required`, `Exists` |
| `transaction_date` | Yes | `string` | - | `Required`, `Date` |
| `payment_method_id` | Yes | `int` | - | `Required`, `Exists` |
| `paying_user_id` | Yes | `int` | - | `Required`, `Exists` |
| `amount` | Yes | `int` | - | `Required` |
| `currency_id` | Yes | `?int` | - | `Exists` |
| `allocations` | Yes | `array` | - | - |

## ReceiptUpdateData

Class: `App\Data\ReceiptUpdateData`

| Field | Required | Type | Allowed Values (Enum) | Rules |
|---|---|---|---|---|
| `transaction_number` | Yes | `string` | - | `Required` |
| `receiving_account_id` | Yes | `int` | - | `Required`, `Exists` |
| `transaction_date` | Yes | `string` | - | `Required`, `Date` |
| `payment_method_id` | Yes | `int` | - | `Required`, `Exists` |
| `paying_user_id` | Yes | `int` | - | `Required`, `Exists` |
| `receiving_user_id` | Yes | `int` | - | `Required`, `Exists` |

## RoleData

Class: `App\Data\RoleData`

| Field | Required | Type | Allowed Values (Enum) | Rules |
|---|---|---|---|---|
| `name` | Yes | `string` | - | - |
| `user_group_id` | Yes | `int` | - | `Exists` |
| `description` | No | `Illuminate\Support\Optional|string|null` | - | - |
| `guard_name` | No | `Illuminate\Support\Optional|string` | - | - |
| `enforce_on_facility` | No | `Illuminate\Support\Optional|bool` | - | - |
| `permissions` | No | `Illuminate\Support\Optional|array` | - | - |

## SensorActionData

Class: `App\Data\SensorActionData`

| Field | Required | Type | Allowed Values (Enum) | Rules |
|---|---|---|---|---|
| `action_trigger` | Yes | `string` | `warning`, `critical` | `In` |
| `action_to_take` | Yes | `string` | `notify`, `ticket`, `request` | `In` |
| `notifiable` | No | `?array` | - | `RequiredIf` |
| `priority` | No | `Spatie\LaravelData\Optional|string|null` | `high`, `normal`, `low` | `RequiredUnless`, `In` |
| `issue_category_id` | No | `Spatie\LaravelData\Optional|int|null` | - | `RequiredIf`, `Exists` |
| `request_type` | No | `Spatie\LaravelData\Optional|string|null` | `work`, `purchase` | `RequiredIf`, `In` |
| `expense_type_id` | No | `Spatie\LaravelData\Optional|int|null` | - | `RequiredIf`, `Exists` |
| `expense_sub_type_id` | No | `Spatie\LaravelData\Optional|int|null` | - | `RequiredIf`, `Exists` |
| `purchase_item_id` | No | `Spatie\LaravelData\Optional|int|null` | - | `RequiredIf`, `Exists` |
| `purchase_item_quantity` | No | `Spatie\LaravelData\Optional|int|null` | - | `RequiredIf` |

## SensorData

Class: `App\Data\SensorData`

| Field | Required | Type | Allowed Values (Enum) | Rules |
|---|---|---|---|---|
| `facility_id` | Yes | `int` | - | `Exists` |
| `sensor_unique_identifier` | Yes | `string` | - | `Exists`, `Unique` |
| `type` | Yes | `string` | `asset`, `space` | `In` |
| `facility_space_id` | No | `Illuminate\Support\Optional|int|null` | - | `Exists`, `RequiredIf` |
| `asset_id` | No | `Illuminate\Support\Optional|int|null` | - | `RequiredIf` |

## SettlementData

Class: `App\Data\SettlementData`

| Field | Required | Type | Allowed Values (Enum) | Rules |
|---|---|---|---|---|
| `debit_bank_account_id` | Yes | `int` | - | `Exists` |
| `payment_advice_number` | No | `?string` | - | - |
| `payment_advice_upload_id` | No | `?int` | - | `Exists` |
| `payment_vouchers` | No | `?array` | - | - |

## TicketData

Class: `App\Data\TicketData`

| Field | Required | Type | Allowed Values (Enum) | Rules |
|---|---|---|---|---|
| `title` | Yes | `string` | - | `Required` |
| `description` | Yes | `string` | - | `Required` |
| `ticket_type` | Yes | `string` | `asset`, `facility`, `other` | `In` |
| `ticket_category_id` | Yes | `int` | - | `Exists` |
| `facility_id` | Yes | `int` | - | `Exists` |
| `facility_space_id` | No | `Spatie\LaravelData\Optional|int|null` | - | `RequiredIf`, `Exists` |
| `asset_id` | No | `Spatie\LaravelData\Optional|int|null` | - | `RequiredIf`, `Exists` |
| `owner_id` | No | `Spatie\LaravelData\Optional|int|null` | - | `Exists` |
| `user_group_id` | No | `Spatie\LaravelData\Optional|int|null` | - | `Exists` |

## TripData

Class: `App\Data\TripData`

| Field | Required | Type | Allowed Values (Enum) | Rules |
|---|---|---|---|---|
| `vehicle_id` | Yes | `int` | - | `Exists` |
| `driver_id` | Yes | `int` | - | `Exists` |
| `departure_city_id` | Yes | `int` | - | `Exists` |
| `destination_city_id` | Yes | `int` | - | `Exists` |
| `depart_at` | Yes | `string` | - | `Date`, `AfterOrEqual` |

## TripStopData

Class: `App\Data\TripStopData`

| Field | Required | Type | Allowed Values (Enum) | Rules |
|---|---|---|---|---|
| `city_id` | Yes | `int` | - | `Exists` |
| `stop_duration` | Yes | `int` | - | - |
| `order` | No | `Spatie\LaravelData\Optional|int|null` | - | - |

## UpdateBillData

Class: `App\Data\UpdateBillData`

| Field | Required | Type | Allowed Values (Enum) | Rules |
|---|---|---|---|---|
| `invoice_number` | No | `?string` | - | - |
| `invoice_date` | No | `?string` | - | `Date` |
| `invoice_upload_id` | No | `?int` | - | `Exists` |
| `tax_invoice_number` | No | `?string` | - | - |
| `notes` | No | `?string` | - | - |

## UploadInvoiceBillData

Class: `App\Data\UploadInvoiceBillData`

| Field | Required | Type | Allowed Values (Enum) | Rules |
|---|---|---|---|---|
| `invoice_number` | Yes | `string` | - | - |
| `invoice_date` | Yes | `string` | - | `Date` |
| `tax_invoice_number` | Yes | `string` | - | - |
| `invoice_upload_id` | No | `?int` | - | `Exists` |
| `notes` | No | `?string` | - | - |

## UtilityData

Class: `App\Data\UtilityData`

| Field | Required | Type | Allowed Values (Enum) | Rules |
|---|---|---|---|---|
| `name` | Yes | `string` | - | `Required` |
| `description` | No | `Spatie\LaravelData\Optional|string|null` | - | - |
| `utility_type` | Yes | `string` | `electricity`, `water`, `gas`, `other` | `In` |
| `stock_keeping_unit_id` | Yes | `int` | - | `Exists` |
| `is_regulated` | No | `Spatie\LaravelData\Optional|bool|null` | - | - |
| `default_price_per_unit` | Yes | `Spatie\LaravelData\Optional|int` | - | `Required` |

## UtilityMeterData

Class: `App\Data\UtilityMeterData`

| Field | Required | Type | Allowed Values (Enum) | Rules |
|---|---|---|---|---|
| `meter_number` | Yes | `string` | - | `Required` |
| `meter_serial_number` | Yes | `string` | - | `Required`, `Unique` |
| `name` | Yes | `string` | - | `Required` |
| `description` | No | `Spatie\LaravelData\Optional|string|null` | - | - |
| `utility_id` | Yes | `int` | - | `Exists` |
| `facility_id` | Yes | `int` | - | `Exists` |
| `reading_unit` | Yes | `int` | - | `Exists` |
| `initial_reading` | Yes | `float` | - | `Required` |
| `status` | No | `Spatie\LaravelData\Optional|string|null` | `active`, `inactive` | `In` |

## UtilityMeterReadingData

Class: `App\Data\UtilityMeterReadingData`

| Field | Required | Type | Allowed Values (Enum) | Rules |
|---|---|---|---|---|
| `current_reading` | Yes | `float` | - | - |
| `reading_image_id` | No | `Spatie\LaravelData\Optional|int|null` | - | `Exists` |

## UtilityPriceData

Class: `App\Data\UtilityPriceData`

| Field | Required | Type | Allowed Values (Enum) | Rules |
|---|---|---|---|---|
| `facility_id` | No | `Spatie\LaravelData\Optional|int` | - | `Exists` |
| `currency_id` | No | `Spatie\LaravelData\Optional|int` | - | `Exists` |
| `price_per_unit` | No | `Spatie\LaravelData\Optional|int` | - | - |
| `utility_id` | No | `Spatie\LaravelData\Optional|int` | - | `Exists` |

## VehicleData

Class: `App\Data\VehicleData`

| Field | Required | Type | Allowed Values (Enum) | Rules |
|---|---|---|---|---|
| `facility_id` | Yes | `int` | - | `Exists` |
| `license_plate` | Yes | `string` | - | `Unique` |
| `color` | Yes | `string` | - | - |
| `vehicle_type` | Yes | `string` | `personal`, `passengers`, `cargo` | `In` |
| `ownership_type` | Yes | `string` | `own`, `lease` | `In` |
| `transmission_type` | No | `?string` | `auto`, `manual` | `In` |
| `passengers` | No | `?int` | - | - |
| `description` | No | `Spatie\LaravelData\Optional|string|null` | - | - |
| `vehicle_photo_upload_id` | No | `?int` | - | `Exists` |
| `document_upload_id` | No | `?int` | - | `Exists` |
| `vehicle_category_id` | No | `?int` | - | `Exists` |
| `vehicle_make_id` | No | `?int` | - | `Exists` |
| `manage_as_asset` | No | `?bool` | - | - |
| `allow_scheduling` | No | `?bool` | - | - |
| `asset_id` | No | `?int` | - | `Exists` |
| `creator_id` | No | `?int` | - | `Exists` |
| `purchased_at` | No | `?string` | - | `Date` |
| `currency_id` | No | `?string` | - | - |
| `gps_tracker_id` | No | `?int` | - | `Exists` |
| `purchase_price` | No | `?int` | - | - |
| `current_value` | No | `?int` | - | - |
| `status` | No | `?string` | `in use`, `decommissioned`, `suspended` | `In` |
| `drivers` | No | `?array` | - | - |

