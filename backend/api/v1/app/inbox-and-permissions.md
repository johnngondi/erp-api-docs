# Inbox summary and `me/permissions`

Two light read endpoints that exist so the frontend can stop polling heavy ones.

- **Inbox summary** replaces the four endpoints every open tab polled every 10 s
  (unread notifications, pending tasks, and the two `responsibility-transfers/mine/*` reads) with one
  counts-only call.
- **`me/permissions`** returns the caller's App permission names across every module (the union of
  what each module dashboard returns under `permissions`), flat and grouped by module, without
  computing any dashboard.

Frontend spec: `docs/frontend/api/v1/app/inbox-and-permissions.md`.

| Method | URI | Name | Controller |
|---|---|---|---|
| `GET` | `/api/v1/app/{company}/inbox/summary` | `app.inbox.summary` | `App\Http\Controllers\Api\V1\App\Inbox\InboxSummaryController` |
| `GET` | `/api/v1/tenant/inbox/summary` | `tenant.inbox.summary` | `App\Http\Controllers\Api\V1\Tenant\Inbox\InboxSummaryController` |
| `GET` | `/api/v1/vendor/inbox/summary` | `vendor.inbox.summary` | `App\Http\Controllers\Api\V1\Vendor\Inbox\InboxSummaryController` |
| `GET` | `/api/v1/landlord/inbox/summary` | `landlord.inbox.summary` | `App\Http\Controllers\Api\V1\Landlord\Inbox\InboxSummaryController` |
| `GET` | `/api/v1/app/{company}/me/permissions` | `app.me.permissions` | `App\Http\Controllers\Api\V1\App\Me\MyPermissionsController` |

Both are caller-scoped and permission-free, exactly like the alerts / pending-tasks / notifications
routes they sit next to: every query is constrained to the authenticated user's own rows, and portal
segregation is the route file's `group:*` middleware. The App routes also carry `company.access`, so
a user who is not a member of `{company}` gets the same `403` the existing inbox routes give, and an
unauthenticated call gets `401`.

## Inbox summary

`GET {portal}/inbox/summary`

```json
{
  "data": {
    "unread_notifications": 3,
    "pending_tasks": 2,
    "active_alerts": 1,
    "open_tickets": 4,
    "outgoing_transfer": {
      "id": 12,
      "status": "active",
      "timestamp_to": "2026-09-27T10:00:00+03:00",
      "to_user": { "id": 5, "name": "Jane" }
    },
    "incoming_transfers": 1
  }
}
```

On the tenant, vendor and landlord portals the three App-only keys are always present with their
empty value: `"open_tickets": null`, `"outgoing_transfer": null`, `"incoming_transfers": 0`.

| Key | Portals | Same scoping as |
|---|---|---|
| `unread_notifications` | all | `GET {portal}/notifications?filter[unread]=true` |
| `pending_tasks` | all | `GET {portal}/pending-tasks` |
| `active_alerts` | all | `GET {portal}/alerts` |
| `open_tickets` | app (null elsewhere) | `open_tickets.total` of `GET /app/{company}/property-management` with no filters |
| `outgoing_transfer` | app (null elsewhere) | `GET /app/{company}/access-management/responsibility-transfers/mine/outgoing` |
| `incoming_transfers` | app (0 elsewhere) | row count of `.../responsibility-transfers/mine/incoming` |

`outgoing_transfer` is either null or the four fields shown. `status` is the raw
`ResponsibilityTransferStatus` value (`pending`, `accepted` or `active`, the open set);
`timestamp_to` is ISO-8601 with offset, or null for a permanent transfer.

Two scoping rules, shared with the listings:

- **Portal.** Alerts, pending tasks and notifications are all narrowed to the answering portal:
  `user_group_id = <portal group> OR user_group_id IS NULL`. A user in both the App and vendor
  groups sees each portal's own rows. Null is legacy ("unscoped, show everywhere").
- **Company.** The transfer reads are scoped to the route `{company}`: a transfer sent or received in
  another company shows when that company is in the URL. The one-open-outgoing rule is a separate,
  cross-company check in `InitiateResponsibilityTransferAction`, which does not use these builders.

### Where the scoping lives

The count and the list must never disagree, so neither re-implements a filter:

- `App\Services\PortalInboxService` exposes the base builders the listing endpoints use:
  `alertsQuery()`, `pendingTasksQuery()`, `notificationsQuery()` and `applyUnread()` (the body of the
  `filter[unread]` callback). The portal rule lives once in its private `scopeToPortal()`, and the
  portal's group id is resolved once per service instance.
- `App\Services\AccessManagement\MyResponsibilityTransfers` holds the `mine/outgoing` and
  `mine/incoming` builders (route company, open statuses, caller as sender / recipient). The company
  is constrained explicitly as well as by the CompanyScoped global scope, which is inert outside an
  HTTP request with `{company}`. Both
  `MyOutgoing…` / `MyIncoming…` controllers read from it.
- `PropertyManagementDashboardService::openTicketsQuery()` is the dashboard's open-tickets predicate,
  and `openTicketsBadgeQuery()` applies it to the caller's unfiltered, all-time property universe
  (`ReportFacilities::scoped()`), which is what the dashboard badge shows.
- `App\Services\Inbox\InboxSummaryService` only composes these.

### Query budget

Counts only: no resources, no eager loads beyond the outgoing transfer's `toUser:id,name`, no
permission maps. Every count is folded into **one** `SELECT (SELECT COUNT(*) …) AS …, …` statement,
so the App portal costs, beyond the per-request middleware baseline:

1. `user_groups` id for the portal (resolved once, used by all three portal-scoped counts);
2. `facility_user` for the caller's allocated properties (the open-tickets universe);
3. the combined counts statement;
4. the outgoing transfer;
5. its `to_user`.

The tenant, vendor and landlord portals run 1 and 3 only.

## `me/permissions`

`GET /api/v1/app/{company}/me/permissions`

```json
{
  "data": {
    "permissions": ["view-facility", "create-facility", "view-lease", "view-role", "generate-report"],
    "modules": {
      "Property Management": ["view-facility", "create-facility", "view-lease"],
      "Access Management": ["view-role", "generate-report"]
    }
  }
}
```

Module-agnostic. `modules` is keyed by permission `tag`. Each entry is exactly what that module's
dashboard returns under `data.permissions`: the caller's `app` roles in `{company}`, their `app`-group
permissions with that tag, narrowed to `view-*`, `create-*`, `generate-*` and `export-*` names.
`permissions` is the flat, de-duplicated union of every module. It is a superset of the Property
Management dashboard's array. `modules` is always a JSON object, `{}` when the caller has nothing.

Both come from `ResolveDashboardPermissionsAction`. The dashboards call `execute($user, 'app', $tag,
$companyId)`, whose signature and output are unchanged. `me/permissions` calls the sibling
`executeByTag($user, 'app', $companyId)`, which runs the same query without the tag filter and
groups the names by tag. Each runs two queries: the user-group id, then one permissions read whose
role and role-permission lookups are subqueries. The action previously ran four.

No server-side caching yet: the cache driver is `file`. Caching arrives with Redis.
