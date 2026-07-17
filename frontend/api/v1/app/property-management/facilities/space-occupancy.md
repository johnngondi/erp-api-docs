# Facility Space Occupancy

Domain: `Property Management > Facilities`

A facility space is let as **two independent slots** — its rent and its service
charge — which may be held by two different leases, one lease, or neither. Three
columns on `facility_spaces` capture this:

| Field | Type | Notes |
|---|---|---|
| `status` | `SpaceStatus` | Serialized as `{ value, label, color }`. See the table below. |
| `rent_occupied_by_lease_id` | integer, nullable | The lease holding the rent, if any. |
| `service_charge_occupied_by_lease_id` | integer, nullable | The lease holding the service charge, if any. |

`FacilitySpaceResource` also exposes a convenience boolean `is_available_for_application`.

## How occupancy is derived

These three fields are **never set directly** — they are recomputed from the
underlying leases by `SyncFacilitySpaceStatusAction`, which is the single writer.
A lease **holds** a slot on a space when it has a lease item on that space whose
components include a rent (resp. service-charge) component, and the lease is
`active` or `suspended` (`terminated` releases the slot; suspended does not).

The status is then the first row below that matches:

| Condition | `status` | Visible when applying |
|---|---|---|
| The rent-holding lease has a pending exit notice | `on notice` | Yes |
| Rent held **and** service charge held | `unavailable` | No |
| Rent held, service charge free | `only service charge available` | No |
| Rent free, service charge held | `only rent available` | Yes |
| Neither held, but an open lease application targets the space | `under consideration` | No |
| Neither held | `available` | Yes |

`on notice` deliberately outranks `unavailable`: an exit notice on the rent lease
means the rent is about to free up, which is what the application picker cares
about.

## Recompute triggers

The recompute runs automatically whenever the truth can change:

- A lease item is created or deleted (`CreateLeaseItemAction`, `DeleteLeaseItemAction`).
- A lease is terminated, activated or suspended (`TerminateLeaseAction`,
  `ActivateLeaseAction`, `SuspendLeaseAction`); processing an approved exit notice
  and deleting a lease funnel through termination.
- A tenant exit notice is created, has its status changed (e.g. rejected → back off
  notice), or is deleted — handled by `TenantExitNoticeObserver`.
- A lease application's targeted spaces are attached, changed, or the application is
  reviewed (approved / rejected releases `under consideration`).

## Application visibility

When a tenant browses spaces to apply for, only spaces whose status is in the
**visible** set above (`available`, `only rent available`, `on notice`) should be
offered. Filter with the `visibleForApplication` scope, or by
`is_available_for_application` on the resource:

```
GET .../facilities/{facility}/spaces?filter[status]=available
```

## Enforcement

When adding lease items, a rent component cannot be attached to a space whose
`rent_occupied_by_lease_id` is already set (and likewise for the service charge);
the request is rejected with a `facility_space_id` validation error. The two slots
are enforced independently, so one lease may take the rent while another takes the
service charge.
