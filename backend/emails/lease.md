# Lease email

The tenant is emailed once, the first time their lease becomes active. The shell, blocks and
sending rules are in [../emails.md](../emails.md).

- Notification: `App\Notifications\PropertyManagement\LeaseIssuedNotification`. It is queued, goes to
  the lease's tenant (`$lease->user`), and writes the in-app record and sends the email. It sends
  no SMS.
- Sent by: `App\Actions\PropertyManagement\LeaseManagement\Lease\NotifyTenantOfLeaseAction`, never
  directly.
- Preview: `GET /_mail/lease` (local only), rendered from the first active lease that has units, or else
  from the first active lease.

## When it is sent

A lease can become active in three ways, and each one calls `NotifyTenantOfLeaseAction`:

| Route to active | Where | Notes |
|---|---|---|
| Final approval step | `SendLeaseIssuedEmailListener` on `LeaseApprovedEvent` | `ApprovalService` writes the status with `saveQuietly()`, so a model observer would never see it. The event is dispatched inside the approval transaction. |
| Manual activation | `ActivateLeaseAction` (`PATCH .../leases/{lease}/activate`) | Only when the lease was `pending`. Re-activating a suspended or terminated lease sends nothing. |
| Active from creation | `LeaseController::store`, `PromoteLooToLeaseAction::execute` | Applies when the creator's role bypasses lease approval, when the approval chain has no applicable step, or when the company has no lease approval template. |

The action sends only when all of these hold:

1. Emails are not suppressed (`OutboundEmail::suppress()`, which wraps the EPMAS importer).
2. The lease, read again from the database, is `active`. The caller's copy can be stale: a
   lease created with no status takes `active` from the column default, and approval writes
   the status quietly on its own instance.
3. `leases.tenant_notified_at` is empty.
4. The tenant has an email address.

It then stamps `tenant_notified_at` with a conditional update (`WHERE tenant_notified_at IS NULL`,
with no model events), and sends through `OutboundEmail::send()`. The send waits for the
surrounding transaction to commit. If the transaction rolls back, the stamp and the email are
both dropped.

### Once only

- Calling the action again sends nothing, because the stamp is set. Two calls that race each
  other send one email, because only one conditional update matches.
- A lease that is suspended or terminated and then re-activated is not emailed again.
- A lease that existed before this email has no stamp. Re-activating it by hand still sends
  nothing, because `ActivateLeaseAction` notifies only when the lease leaves `pending`.
- A tenant with no address is not stamped. If they gain an address later, the next activation
  of a pending lease emails them. A lease that is already active stays unsent.

## What it contains

Subject `Your lease for {property}`, header title `YOUR LEASE`, and these blocks:

1. **Text**: `Dear {tenant name},` and a line saying that the lease for the property is now active.
2. **Key values**: Property, Lease ID (the lease id, as the portals show it: "Lease ID 181"),
   Start date, End date (`d M, Y`), Term (`5 years, 6 months`), Billing cycle (`Quarterly`),
   Monthly rent, Monthly service charge, Deposit, and Currency (`Kenya Shilling (KES)`).
   - Money is the currency code followed by `number_format(…, 2)`, for example `KES 125,000.50`.
     It is stored in currency units, not cents.
   - The monthly figures are the lease's `total_rent_per_month` and
     `total_service_charge_per_month`. The deposit is `total_deposit_amount`, which counts
     unrefunded deposits.
   - A money row whose amount is zero is left out. A lease that is active from creation has no
     units yet, and `KES 0.00` would read as a term of the lease.
3. **Table** "Leased units": one row per lease item, with the space name, rent per month and
   service charge per month. Components are matched to rent and service charge by the same
   expense-category rule as the lease totals, so the rows add up to them. The table is left out
   entirely when the lease has no items.
4. **Button** "View lease", linking `FrontendUrl::to('tenant.lease', ['id' => $lease->id])`, which
   is `/tenant/lease-management/leases/{id}` on the frontend.
5. **Note**: "Questions about your lease? Write to {company email}". When the company has no
   email, it says to contact the property manager instead.

The in-app record carries `title` "Lease active", `message`, `lease_id`, `facility_name` and the
resolved `resource_url` of the lease. Its subject is the lease and its company is the lease's
company.

## Deployment

The `tenant_notified_at` migration does not backfill. Leases that are already active have no
stamp, but no route above emails them, because only a lease leaving `pending`, a new lease, or
an approval reaches the action.
