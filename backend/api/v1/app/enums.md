# App API Enum Values and Colors

These values come from enum classes used across App API resources/responses. Use these for badges/chips in frontend.

## AssetStatus

Class: `App\Enums\AssetStatus`

| Value | Color |
|---|---|
| `in use` | `success` |
| `decommissioned` | `danger` |
| `suspended` | `warning` |

## AssetValueChangeType

Class: `App\Enums\AssetValueChangeType`

| Value | Color |
|---|---|
| `appreciate` | `success` |
| `depreciate` | `warning` |

## BidStatus

Class: `App\Enums\BidStatus`

| Value | Color |
|---|---|
| `bid` | `secondary` |
| `selected` | `success` |
| `recommended` | `primary` |
| `rejected` | `danger` |

## BillStatus

Class: `App\Enums\BillStatus`

| Value | Color |
|---|---|
| `pending` | `warning` |
| `unpaid` | `danger` |
| `partially-paid` | `info` |
| `paid` | `success` |
| `cancelled` | `secondary` |

## BookingStatus

Class: `App\Enums\BookingStatus`

| Value | Color |
|---|---|
| `new` | `primary` |
| `confirmed` | `success` |
| `boarded` | `secondary` |

## ContractStatus

Class: `App\Enums\ContractStatus`

| Value | Color |
|---|---|
| `pending` | `warning` |
| `active` | `success` |
| `suspended` | `secondary` |
| `expired` | `danger` |
| `inactive` | `muted` |

## CreditNoteStatus

Class: `App\Enums\CreditNoteStatus`

| Value | Color |
|---|---|
| `pending` | `secondary` |
| `applied` | `success` |
| `cancelled` | `danger` |

## ExpenseStatus

Class: `App\Enums\ExpenseStatus`

| Value | Color |
|---|---|
| `pending` | `info` |
| `unpaid` | `warning` |
| `partially-paid` | `primary` |
| `paid` | `success` |
| `cancelled` | `danger` |

## InvoiceStatus

Class: `App\Enums\InvoiceStatus`

| Value | Color |
|---|---|
| `pending` | `secondary` |
| `unpaid` | `warning` |
| `partially paid` | `primary` |
| `paid` | `success` |
| `cancelled` | `danger` |

## LeaseApplicationStatus

Class: `App\Enums\LeaseApplicationStatus`

| Value | Color |
|---|---|
| `pending` | `secondary` |
| `approved` | `success` |
| `review` | `info` |
| `rejected` | `danger` |

## LeaseStatus

Class: `App\Enums\LeaseStatus`

| Value | Color |
|---|---|
| `active` | `success` |
| `suspended` | `secondary` |
| `terminated` | `danger` |

## LpoStatus

Class: `App\Enums\LpoStatus`

| Value | Color |
|---|---|
| `lpo` | `primary` |
| `delivered` | `success` |
| `cancelled` | `danger` |

## PaymentVoucherPayableAs

Class: `App\Enums\PaymentVoucherPayableAs`

| Value | Color |
|---|---|
| `vendor` | `primary` |
| `landlord` | `info` |
| `tenant` | `success` |
| `other` | `secondary` |

## PaymentVoucherStatus

Class: `App\Enums\PaymentVoucherStatus`

| Value | Color |
|---|---|
| `pending` | `secondary` |
| `released` | `info` |
| `paid` | `success` |
| `cancelled` | `danger` |

## PayoutStatus

Class: `App\Enums\PayoutStatus`

| Value | Color |
|---|---|
| `pending` | `secondary` |
| `approved` | `primary` |
| `settled` | `success` |
| `cancelled` | `danger` |

## Priority

Class: `App\Enums\Priority`

| Value | Color |
|---|---|
| `high` | `danger` |
| `normal` | `primary` |
| `low` | `success` |

## ProcurementRequestStatus

Class: `App\Enums\ProcurementRequestStatus`

| Value | Color |
|---|---|
| `new` | `info` |
| `request` | `secondary` |
| `order` | `primary` |
| `quotes-review` | `primary` |
| `lpo` | `success` |
| `rejected` | `danger` |

## ProcurementRequestStepStatus

Class: `App\Enums\ProcurementRequestStepStatus`

| Value | Color |
|---|---|
| `pending` | `-` |
| `approved` | `-` |
| `review` | `-` |
| `rejected` | `-` |
| `waiting-quotes` | `-` |
| `quotes-review` | `-` |

## ProjectMilestoneStatus

Class: `App\Enums\ProjectMilestoneStatus`

| Value | Color |
|---|---|
| `pending` | `secondary` |
| `in-progress` | `primary` |
| `late` | `warning` |
| `stuck` | `danger` |
| `completed` | `success` |

## ProjectStatus

Class: `App\Enums\ProjectStatus`

| Value | Color |
|---|---|
| `pending` | `warning` |
| `in-progress` | `primary` |
| `completed` | `success` |
| `cancelled` | `secondary` |
| `stalled` | `danger` |

## ProjectTaskPriority

Class: `App\Enums\ProjectTaskPriority`

| Value | Color |
|---|---|
| `low` | `success` |
| `normal` | `primary` |
| `high` | `danger` |

## ProjectTaskStatus

Class: `App\Enums\ProjectTaskStatus`

| Value | Color |
|---|---|
| `pending` | `secondary` |
| `in-progress` | `primary` |
| `late` | `warning` |
| `stuck` | `danger` |
| `done` | `success` |

## ReceiptAllocationStatus

Class: `App\Enums\ReceiptAllocationStatus`

| Value | Color |
|---|---|
| `pending` | `secondary` |
| `confirmed` | `success` |
| `cancelled` | `cancelled` |

## ReceiptStatus

Class: `App\Enums\ReceiptStatus`

| Value | Color |
|---|---|
| `pending` | `secondary` |
| `confirmed` | `success` |
| `cancelled` | `danger` |

## RemittanceStatus

Class: `App\Enums\RemittanceStatus`

| Value | Color |
|---|---|
| `pending` | `secondary` |
| `unpaid` | `warning` |
| `paid` | `success` |
| `cancelled` | `danger` |

## SensorCurrentStatus

Class: `App\Enums\SensorCurrentStatus`

| Value | Color |
|---|---|
| `ok` | `success` |
| `warning` | `warning` |
| `critical` | `danger` |

## SettlementStatus

Class: `App\Enums\SettlementStatus`

| Value | Color |
|---|---|
| `pending` | `info` |
| `settled` | `success` |
| `cancelled` | `danger` |

## SpaceStatus

Class: `App\Enums\SpaceStatus`

| Value | Color |
|---|---|
| `available` | `success` |
| `occupied` | `secondary` |

## Status

Class: `App\Enums\Status`

| Value | Color |
|---|---|
| `pending` | `secondary` |
| `active` | `success` |
| `suspended` | `warning` |
| `expired` | `danger` |
| `inactive` | `danger` |
| `in-use` | `success` |
| `decommissioned` | `danger` |
| `running` | `success` |
| `off` | `danger` |
| `terminated` | `danger` |
| `cancelled` | `danger` |
| `available` | `success` |
| `occupied` | `danger` |
| `confirmed` | `success` |
| `unpaid` | `danger` |
| `partially paid` | `danger` |
| `applied` | `success` |
| `paid` | `success` |

## SystemService

Class: `App\Enums\SystemService`

| Value | Color |
|---|---|
| `sensor` | `primary` |
| `gps` | `primary` |
| `leasing` | `primary` |

## SystemType

Class: `App\Enums\SystemType`

| Value | Color |
|---|---|
| `first-party` | `success` |
| `third-party` | `warning` |

## TenantStatementStatus

Class: `App\Enums\TenantStatementStatus`

| Value | Color |
|---|---|
| `confirmed` | `success` |
| `cancelled` | `danger` |

## TicketStatus

Class: `App\Enums\TicketStatus`

| Value | Color |
|---|---|
| `closed` | `success` |
| `open` | `primary` |

## TripStatus

Class: `App\Enums\TripStatus`

| Value | Color |
|---|---|
| `open` | `success` |
| `full` | `warning` |
| `in-progress` | `primary` |
| `completed` | `secondary` |

## UtilityType

Class: `App\Enums\UtilityType`

| Value | Color |
|---|---|
| `electricity` | `success` |
| `gas` | `default` |
| `water` | `primary` |
| `other` | `warning` |

## VehicleStatus

Class: `App\Enums\VehicleStatus`

| Value | Color |
|---|---|
| `in use` | `success` |
| `decommissioned` | `danger` |
| `suspended` | `warning` |

