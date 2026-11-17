# Property Management API (Frontend)

Base prefix:

`/api/v1/app/{company}/property-management`

## Modules

- Dashboard
  - `docs/frontend/app/property-management/dashboard.md`
- Facilities
  - `docs/frontend/app/property-management/facilities/facilities.md`
  - `docs/frontend/app/property-management/facilities/floor_generation.md`
  - `docs/frontend/app/property-management/facilities/utility-meters.md`
  - `docs/frontend/app/property-management/facilities/meter-reading-extraction.md`
  - `docs/frontend/app/property-management/facilities/space-occupancy.md`
  - `docs/frontend/app/property-management/facilities/budget-settings.md`
  - `docs/frontend/api/v1/app/billing-addresses.md` (`facilities/{facility}/billing-addresses`)
- Tickets
  - `docs/frontend/app/property-management/tickets.md`
- Procurement
  - `docs/frontend/app/property-management/procurement.md`
- FRC sanctions list (quick action)
  - `docs/frontend/app/property-management/frc-sanctions.md`
- Adhoc ETR signing (quick action)
  - `docs/frontend/app/property-management/finance/adhoc-etr-signing.md`
- Finance
  - `docs/frontend/app/property-management/finance.md`
  - `docs/frontend/app/property-management/finance/bill.md`
  - `docs/frontend/app/property-management/finance/expense.md`
  - `docs/frontend/app/property-management/finance/payment-voucher.md`
  - `docs/frontend/app/property-management/finance/remittance.md`
  - `docs/frontend/app/property-management/finance/settlement.md`
  - `docs/frontend/app/property-management/finance/vendor-statement.md`
  - `docs/frontend/app/property-management/finance/bank-statement-reconciliation.md`
  - `docs/frontend/app/property-management/finance/bank-payments.md`
- Reports
  - `docs/frontend/app/property-management/reports/README.md`
  - `docs/frontend/app/property-management/reports/billings-and-collections.md`
- Settings
  - `docs/frontend/app/property-management/settings.md`
- Lease Management
  - `docs/frontend/app/property-management/lease-management/lease-applications.md`
  - `docs/frontend/app/property-management/lease-management/leases.md`
  - `docs/frontend/app/property-management/lease-management/invoices.md`
  - `docs/frontend/app/property-management/lease-management/credit-notes.md`
  - `docs/frontend/app/property-management/lease-management/receipts.md`
  - `docs/frontend/app/property-management/lease-management/receipt-reallocation.md`
  - `docs/frontend/app/property-management/lease-management/lease-deposits.md`
  - `docs/frontend/app/property-management/lease-management/lease-escalations.md`
  - `docs/frontend/app/property-management/lease-management/lease-opening-balances.md`
  - `docs/frontend/app/property-management/lease-management/charges-imports.md`

## Rules that span pages

- **ETR signing is current-month only.** An invoice is signed on the ETR only in the month of its
  approval date (`created_at`, which Backdate moves), read in the company's timezone
  (`companies.timezone`, default `Africa/Nairobi`). Out of the month, `POST .../invoices/{invoice}/sign`
  answers `422` with `code: "etr_period_closed"`, and every invoice carries `etr_signing` for the
  Sign button. To sign a missed invoice, change its date to this month first. Credit notes and
  customer invoices are not affected. See `lease-management/invoices.md`, "Current month only".

Apply shared auth, tenancy, and error handling guidance from:

- `docs/frontend/app/README.md`
