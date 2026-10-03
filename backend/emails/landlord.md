# Landlord: "Your properties" email

A landlord is told when one of their properties changes, and every such email lists all the
properties managed for them with their current status. The same table appears in the landlord
section of the welcome email. Shared rules (sending after commit, suppression, branding) are in
[../emails.md](../emails.md).

## When it is sent

| Event | Where | Headline |
|---|---|---|
| A property is created | `CreateFacilityAction`, after the property, its roles and its management contract are saved | ":property was added to your portfolio" |
| A property is deactivated | `DeactivateFacilityAction` (`PATCH .../property-management/facilities/{facility}/deactivate`) | ":property was deactivated" |
| A property is reactivated | `ActivateFacilityAction` (`PATCH .../property-management/facilities/{facility}/activate`) | ":property was reactivated" |
| A property is deleted (soft delete) | `DeleteFacilityAction` (`DELETE .../property-management/facilities/{facility}`) | ":property was removed from your portfolio" |

A property has no "terminated" state, and nothing in the app writes a management contract's
status, so these four events are the only ones. Activating a property that is already active,
or deactivating one that is already inactive, sends nothing: the status did not change.

Each action calls `NotifyLandlordOfPropertiesAction::execute($landlord, $company, $property, $change)`,
which sends `LandlordPropertiesNotification` through `OutboundEmail::send()`, so the email
leaves only once the request's transaction commits.

## Who is skipped, and why

- **A `landlord_id` that is not a landlord.** `CreateFacilityAction` defaults `landlord_id` to
  the staff member creating the property when none is given. Only a user in the `landlord` user
  group is emailed.
- **A landlord with no email address**, or no landlord at all.
- **A new landlord's own properties.** `LandlordController::store` creates the landlord and
  their properties together inside `OutboundEmail::suppress()`, so they receive one welcome
  email listing the portfolio rather than one email per property.
- **The EPMAS importer**, which runs inside `OutboundEmail::suppress()` and does not use
  `CreateFacilityAction` anyway.

## The email

`App\Notifications\PropertyManagement\LandlordPropertiesNotification`: queued, delivered in-app
(`database`) and by email; branded with, and filed under, the property's company.

- Title `YOUR PROPERTIES`; subject "Property added: :property" (deactivated, reactivated,
  removed likewise).
- A headline saying what changed, a greeting, then a table of **every** property of the landlord
  in that company, by name: Property, Status (`Active`, `Inactive`, ...). When at least one of
  them has a management contract, a third column shows the contract's effective status (an
  active contract past its end date reads `Expired`); a property without one shows `—`.
- Deleted properties are not listed; the email about a deletion names the property in its
  headline only.
- Button "View properties", linking `FrontendUrl::to('landlord.properties')`
  (`/landlord/facilities` in the landlord portal).
- A closing note with the company's email and phone, when it has them.

The list is read when the notification is rendered, so a queued email shows the portfolio as it
stands then. It is scoped to the notification's company explicitly: a landlord may own
properties managed by more than one company, and one company's email does not list another's.

The in-app copy (`toArray()`) carries `title`, `message` (the headline), `change`
(`added`, `deactivated`, `reactivated`, `deleted`), `facility_id` and the usual
`resource_type`/`resource_id`/`resource_url` (none for a deleted property).

## Welcome section

`App\Support\Mail\Welcome\LandlordWelcomeSection` (registered for the `landlord` portal in
`config('emails.welcome_sections')`) adds, after the shared welcome content, one line and the
same property table, captioned "Your properties", scoped to the welcoming company. A landlord
with no properties gets nothing added.

## Preview

`GET /_mail/landlord-properties` (local only) renders the email for the first landlord who owns
a property, as if that property had just been added. It shows "no sample data" when no
landlord has one.
