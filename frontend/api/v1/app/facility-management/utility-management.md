# Utility Management API

> **Moved.** Utility meters and their readings are now a standalone resource
> under **Property Management**:
>
> `/api/v1/app/{company}/property-management/facilities/utility-meters`
>
> See [Utility Meters](../property-management/facilities/utility-meters.md) for
> the current endpoints, payloads and permissions.

The previous `facility-management/utility-management/meters` endpoints have been
removed. Update any clients to the new `property-management/facilities/utility-meters`
path. Key differences:

- Meters are no longer nested under a specific facility; pass `facility_id` in the payload.
- `utility_id` now references `lease_components` (utility charges), not `utilities`.
- Meters accept an optional `space_ids` array (many-to-many with facility spaces).
- Readings live at `.../facilities/utility-meters/{meter}/readings`.
