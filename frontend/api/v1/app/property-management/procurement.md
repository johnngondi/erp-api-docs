# Procurement API

Domain: `Property Management`

Base prefix:

`/api/v1/app/{company}/property-management/procurement`

## Dates

Quote, LPO and site visit `delivery_at`, and a purchase item's `next_update_due`, come back with `raw` as a **plain date**, `"2026-10-01"` (row 142). They used to be UTC
timestamps - `"2026-09-30T21:00:00.000000Z"` for 1 Oct in Nairobi - and a form that sent `raw` back
saved the day before. The server now reads any date it is sent as the day it means in
Nairobi, but send `YYYY-MM-DD`.

## Procurement Requests

Endpoints:

- `GET /requests`
- `POST /requests`
- `GET /requests/{procurementRequest}`
- `PUT/PATCH /requests/{procurementRequest}`
- `DELETE /requests/{procurementRequest}`
- `GET /requests/{procurementRequest}/items`
- `POST /requests/{procurementRequest}/items`
- `GET /requests/{procurementRequest}/items/{item}`
- `PUT/PATCH /requests/{procurementRequest}/items/{item}`
- `DELETE /requests/{procurementRequest}/items/{item}`
- `GET /requests/{procurementRequest}/steps/{step}`
- `PUT/PATCH /requests/{procurementRequest}/steps/{step}`
- `POST /requests/{procurementRequest}/rebase`
- `POST /requests/{procurementRequest}/request-more-quotes`
- `POST /requests/{procurementRequest}/evaluate-anyway`

### List query support

`GET /api/v1/app/{company}/property-management/procurement/requests`

- Filters:
  - `filter[id]`, `filter[title]`, `filter[ticket_id]`, `filter[facility_id]`, `filter[asset_id]`
  - `filter[expense_type_id]`, `filter[expense_category_id]`, `filter[expense_sub_type_id]`, `filter[created_at]`, `filter[priority]`, `filter[type]`
- Sort:
  - `sort=id,title,type,created_at,priority`
- Include: not supported
- Fields: not supported
- Pagination: `per_page`, `page`

Example:

`GET /api/v1/app/12/property-management/procurement/requests?filter[type]=work&sort=-created_at`

### Create/Update payload

A request can be updated or deleted only while its status is `new` or `request` (`permissions.update`
/ `permissions.delete` answer false afterwards, for the requester and permission holders alike). Once
it has gone out for quotes the way to change what was asked is a rebase (below).

| Field | Required | Type | Allowed Values / Notes |
|---|---|---|---|
| `title` | Yes | string | - |
| `description` | Yes | string | - |
| `type` | Yes | string | `purchase`, `work` |
| `expense_type_id` | Yes | integer | Must exist in `expense_types.id` |
| `expense_category_id` | No | integer | Must exist in `expense_categories.id`. The request's own category - any category may be chosen, whatever the expense type's default. When omitted, the expense type's default category is used |
| `expense_sub_type_id` | Yes | integer | Must belong to selected expense type |
| `facility_id` | Yes | integer | Must exist in `facilities.id` |
| `ticket_id` | No | integer | Must exist in `tickets.id` |
| `asset_id` | No | integer | Must exist in `assets.id` |
| `owner_id` | No | integer | Must exist in `users.id`, defaults to current user |
| `items` | Conditional | array | Required when `type=purchase` |

`items[]` object (for `type=purchase`):

| Field | Required | Type | Allowed Values / Notes |
|---|---|---|---|
| `purchase_item_id` | Yes | integer | Must exist in `purchase_items.id` |
| `quantity` | Yes | integer | Minimum `1` |

### Suppliers asked to quote

`GET /requests/{procurementRequest}` returns `suppliers`: every supplier asked to quote on the request, in any round, cancelled invitations included. Those who have quoted come first, in the order they quoted, then the rest by name. The list and create/update responses leave it out. A supplier's live quote is shown over one a rebase shelved.

| Field | Type | Notes |
|---|---|---|
| `id` | integer | The invitation |
| `vendor` | object | `{ id, name, email, phone, profile_photo_url }` |
| `is_default`, `is_preferred` | boolean | - |
| `has_quoted` | boolean | Whether the supplier has sent a quote: a quote exists or `quoted_at` is set |
| `quote` | object or null | `{ id, status: { value, color }, submitted: { raw, formatted, diff } }`. No amounts: those stay on `quotes` and the bids list. `status.value` may be `shelved` |
| `invited` | object | `{ raw, formatted, diff }`: when the supplier was (last) asked (`invited_at`) |
| `round` | integer | The quote round the supplier was (last) asked in |
| `is_carried_over` | boolean | A bidder of the round a rebase cancelled, asked again |
| `cancelled` | boolean | The invitation was cancelled by a rebase |
| `cancelled_at` | object or null | `{ raw, formatted }` |
| `declined` | boolean | The supplier declined the invitation from the supplier portal |
| `declined_at` | object or null | `{ raw, formatted }` |
| `decline_reason` | string or null | The reason the supplier gave, if any |
| `quoted_at` | object or null | `{ raw, formatted }`: when the supplier's quote came in. Kept when the quote is shelved |
| `reminder_count` | integer | Daily quote reminders sent to the supplier for this request |
| `last_reminded_at` | object or null | `{ raw, formatted }` |

### Review step payload

`PUT/PATCH /requests/{procurementRequest}/steps/{step}`

| Field | Required | Type | Allowed Values / Notes |
|---|---|---|---|
| `status` | Yes | string | `approved`, `review`, `rejected` |
| `comment` | Conditional | string | Required in many non-approve flows |
| `preferred_vendor_id` | Conditional | integer | Required if `has_preferred_vendor=true` |
| `selected_option` | Conditional | integer | The quote (work) or price option (purchase) the procurement step picks. Optional on the first pass, where leaving it out keeps the system's recommendation, and required when a later step sent the request back. It must belong to this request. See [Overriding the recommendation](#overriding-the-recommendation) |
| `expense_category_id` | Conditional | integer | Required if current step has `select_expense_category=true` and `status=approved` |
| `vendors` | Conditional | array | Required if `override=true` |
| `has_preferred_vendor` | No | boolean | Default `false` |
| `override` | No | boolean | Default `false` |

`comment` is required when the status is `review` or `rejected`, when `has_preferred_vendor` or
`override` is set, and when `selected_option` is not the system's recommendation.

The quote deadline is never part of this payload: the system sets it (see
[Quote deadline](#quote-deadline)). A `deadline` key sent by an older client is ignored.

### Overriding the recommendation

The step with `handle_procurement=true` may replace the system's pick with another one when it
approves:

| Request type | When the step can choose | What the system recommends |
|---|---|---|
| `work` | Step status `quotes-review` (quotes are in), or `pending` after a later step sent it back | The lowest quote, with ties going to the supplier with the fewest awards |
| `purchase` | Step status `pending` | The cheapest price option, with ties going to the supplier with the fewest purchase LPOs |

`GET /requests/{procurementRequest}/steps/{step}` tells the form what it may do:

- `meta.can_change_option` is `true` when the caller can pick, so show the choice.
- Each quote in `meta.quotes` carries `is_computer_recommended` and `is_selected`.
  `is_computer_recommended` marks the system's pick. The system sets it once, and nothing a
  person does changes it, not even an override. `is_selected` marks the current pick, which is
  the system's until someone overrides it. Preselect `is_selected`.
- Each option in `meta.options` carries `is_computer_recommended` and `is_selected`, which is set
  once the step approves. Before then, preselect `is_computer_recommended`. Unlike a quote's flag,
  an option's system pick is worked out again while nothing is selected and the prices move (see
  [Price options follow the catalogue](#price-options-follow-the-catalogue)); no person changes it.

Send the chosen id as `selected_option` with `status=approved`. If you leave it out on the first
pass, the system's pick stands. If you pick anything other than the system's pick, you must send a
`comment` giving the reason, or the request returns **422** keyed on `comment`. An id from another
request returns **422** keyed on `selected_option`.

### Price options follow the catalogue

The price options of a `purchase` request are not a snapshot. While the request's
`handle_procurement` step is current and `pending`, they are brought back in line with the purchase
item prices:

- on **every** `GET /requests/{procurementRequest}/steps/{step}` by an actor of that step, including
  after a later step sent the request back;
- when a price changes (`PUT .../purchase-items/{item}/prices/{price}`);
- when the item's `vendors`, `currency_id` or `base_price` changes (`PUT .../purchase-items/{item}`;
  a new base price is first written to every supplier's price row);
- when an operator runs `procurement:recompute-purchase-options` (see
  [Maintenance commands](#maintenance-commands)).

There is one option per (purchase item, supplier) priced at the request's property, or at any
property when the item has no price there. A reconcile does three things:

| Case | What happens |
|---|---|
| The pair still has a price | The option keeps its `id`. Quantity (from the request item), unit price, amount, currency (the item's), title (the supplier's name) and `total` are updated if they moved |
| A pair has no option yet (a supplier was added) | A new option is created |
| An option's pair has no price any more (the supplier was removed, or the item left the request) | The option and its item are deleted |

Option ids are stable for pairs that did not change, so a form holding an id can keep it. Refetch
the step before approving if the catalogue may have moved.

**The system's pick.** While no option has `is_selected=true` (the step has not approved yet), the
system's pick is worked out again whenever the options change: `is_computer_recommended` moves to the
cheapest option as it now stands (ties to the supplier with the fewest purchase LPOs). Once the step
has approved and a later step sent the request back, the selected option and the system's pick both
stay where they are; only totals move. "Permanent" means no person changes the system's pick, not
that it is never worked out again.

**A removed supplier.** If the removed supplier's option was the selected one, the request has no
selection any more and the system picks again among the remaining options. The step must choose
again when it approves; a `selected_option` other than the new system pick still needs a `comment`.

### Pending tasks on a request

Each step, and each return to the creator, puts a [pending task](../pending-tasks.md) in front of
whoever has to act next. The `taskable` is the request (`App\Models\FacilityProcurementRequest`), so
the task links to the request page.

| Request state | Who holds a task | Title |
|---|---|---|
| Current step `pending` | Each of the step's `actors` | `Review procurement request: {title}` |
| Current step `pending`, sent back by a later step | Each of the step's `actors` | `Procurement request returned for review: {title}`. The description quotes the comment from the step that sent it back |
| Current step `quotes-review` | Each of the step's `actors` | `Select a quote for {title}` |
| Current step `waiting-quotes` | Nobody, because the request is waiting on suppliers | - |
| Current step `waiting-quotes`, deadline passed with too few quotes | Each of the step's `actors` | `Only {received} of {required} quotes received for {title}`. Cleared by request more quotes, evaluate anyway or a rebase |
| First step sent back to the creator | The request's creator | `Make changes to procurement request: {title}` |
| Rejected, LPO raised, or request deleted | Nobody | - |

The tasks are cleared and raised again every time the request moves, so each person holds at most
one task per request.

Open steps that were dispatched before these tasks existed have no task. Run
`php artisan procurement:sync-step-tasks` once (add `--company={id}` to limit it to one company) to
raise them. Running it again changes nothing.

### Who is invited (round-robin)

A work request's suppliers are picked in rotation, so the same few are not asked every time. The
pool is the **active** suppliers prequalified for the request's category (`expense_sub_type_id`) on
its property (or on every property). They are ordered by:

1. **The supplier asked the fewest times goes first**: the number of invitations
   (`procurement_request_vendors` rows) the supplier holds on the company's requests in the same
   category, every round, cancelled invitations included.
2. Then the **least recently asked** (`MAX(invited_at)`), never asked first.
3. Then by supplier id.

Each round asks `rfq_vendor_batch_size` suppliers (default 10; see [Settings](settings.md)). The
first round of a request leaves the request's own invitations out of the history, so approving the
procurement step a second time asks the same suppliers. A later round never asks a supplier already
asked on the request. A preferred or default supplier set on the request is still invited on the
first round as before.

When the request's sub-category has a default supplier and the company prioritizes it, the first
round starts with that supplier alone, and the rotation is asked only when that stage ends; see
[Default supplier first](#default-supplier-first).

An invitation is **open** while it is neither cancelled (`cancelled_at`, by a rebase) nor declined
(`declined_at`, by the supplier). Only open invitations can quote, send a site visit report or
assign a technician, get daily quote reminders and are told of a new deadline. A supplier whose
invitation was declined can still view the request in the supplier portal.

`procurement_request_vendors` is the record of who was asked, who quoted and who was reminded:
`round`, `is_carried_over`, `invited_at`, `cancelled_at`, `quoted_at` (stamped when the supplier's
quote on the request is created; a shelved quote keeps it), `reminder_count` and `last_reminded_at`
(stamped by each daily quote reminder). Invitations per supplier, `quoted_at` and `reminder_count`
are the basis for a later rule deactivating suppliers who keep not responding (not built yet).

### Default supplier first

A **non-emergency** work request whose sub-category allows a default supplier
(`can_have_default_vendor`) and has one goes to that supplier **alone** when the procurement step is
approved, as long as the company setting `rfq_prioritize_default_vendor` is `yes` (see
[Settings](settings.md)). This is the request's preliminary stage.

The default supplier is, in order:

1. the property's default: the supplier whose prequalification for the sub-category names the
   request's property (`facility_expense_sub_type_vendors.is_default_in_facility_id`); they must be
   prequalified and active;
2. else the sub-category's company-wide `default_vendor_id`, when that supplier is active and the
   property is not in the sub-category's `default_vendor_excluded_facility_ids` (see
   [Settings](settings.md));
3. else nobody, and the request goes straight to the rotation.

While preliminary, the request carries `is_preliminary: true`, `default_vendor_id`, and
`preliminary_until` = now + `rfq_default_vendor_exclusive_window_hours` (default 24). The same value
is its `deadline`. A preferred supplier chosen by the step is asked alongside the default one.

The stage ends in one of four ways:

| What happens | Result |
|---|---|
| The default supplier quotes with an `amount` (before discount and tax) at or under `rfq_default_vendor_auto_accept_max_amount` (default 50000) | The request and the procurement step move to `quotes-review` at once, with that quote recommended. Nobody else is asked |
| The default supplier quotes above that amount | Broadcast (`reason: over-threshold`) |
| The default supplier declines from the supplier portal | Broadcast at once (`reason: declined`) |
| The window passes without a quote | Broadcast by `procurement:release-preliminary-rfqs`, hourly at ten past the hour (`reason: window-expired`) |

A broadcast:

- clears `is_preliminary` and `preliminary_until` and sets the `deadline` to
  `rfq_default_deadline_days` working days from now;
- adds the next `rfq_vendor_batch_size` suppliers in rotation to the same round
  (`RfqInvitationNotification`). The default supplier stays invited and may still quote, unless they
  declined;
- tells every supplier already asked whose invitation is open and who has not quoted the new
  deadline (`RfqDeadlineExtendedNotification`);
- gives each actor of the procurement step an in-app `PreliminaryRfqBroadcastNotification`
  ("RFQ sent to more suppliers"). Its data carries `procurement_request_id`, `reason`
  (`over-threshold`, `declined` or `window-expired`), `invited` (suppliers newly asked) and
  `deadline` (ISO 8601), with a `message` naming the supplier and, for `over-threshold`, their
  amount and the auto-accept amount.

From then on the request follows the normal flow (deadline, shortfall, request more quotes,
evaluate anyway, rebase). Between `preliminary_until` and the next hourly run (at most an hour)
quoting is closed, because the deadline has passed.

Only the first round has a preliminary stage. A round after a rebase, "request more quotes" and an
`override` follow the normal round-robin flow. Emergency requests are unchanged: they still go to
the default supplier only, with no broadcast.

**Step form.** `GET /requests/{procurementRequest}/steps/{step}` returns in `meta`:

- `default_vendor`: `{ id, name }` of the supplier the request will go to first. Set only on the
  procurement step of a first-round, non-emergency work request while the step is `pending` and
  the company prioritizes default suppliers; otherwise `null`;
- `defaultVendorContext`: a sentence to show above the supplier list. With `default_vendor` set it
  names the supplier, the auto-accept amount, the window in hours, the batch size and the
  working-day deadline that follows a broadcast. For an emergency request it says the RFQ goes to
  the default supplier only and not to the others automatically. `null` when there is no default
  supplier;
- `vendors`: with `default_vendor` set, the batch that would be asked after the default supplier
  (the next `rfq_vendor_batch_size` in rotation, the default supplier left out).

### Quote deadline

The quote deadline is set by the system; no step sets or edits it. It is counted in **working
days** (Monday to Friday; public holidays are not excluded) and falls at **17:00** on the day
reached. `rfq_default_deadline_days` (default 3) is used:

- when the procurement step opens the first round, unless the request goes to its default supplier
  first;
- for every round after a rebase;
- for each "request more quotes" extension;
- when a request in its default-supplier stage is sent to the other suppliers (broadcast).

While the request is with its default supplier alone, the deadline is the end of that supplier's
window, `preliminary_until` (`rfq_default_vendor_exclusive_window_hours` from the step approval, in
clock hours, not working days).

### Rebase

`POST /requests/{procurementRequest}/rebase`

Puts a work request back for quotes so the procurement officer can change it and send it out again.

| Field | Required | Type | Notes |
|---|---|---|---|
| `reason` | Yes | string | Max 2000 characters. Sent to every supplier asked as the cancellation reason |

Allowed (`permissions.rebase`) for an actor of the current procurement step (`handle_procurement`)
who holds `update-facility-procurement-request-step`, while that step is `waiting-quotes` or
`quotes-review` (request `order` or `quotes-review`): the chain has not moved past procurement.
Otherwise **403**. `can_rebase` on the request says whether the state allows it.

What happens, in one transaction:

- the round's live quotes (`bid`, `recommended`) become `shelved`, with `shelved.at` and
  `shelved.reason`; `is_computer_recommended` stays on them as history and `round` is unchanged;
- every open invitation is cancelled (`cancelled_at`), and each supplier asked gets
  `RfqCancelledNotification` (in-app and email) with the reason, after commit;
- the request goes back to `request`, `deadline` and the shortfall are cleared, `quote_round` goes
  up by one, and `rebased` records who, when and why;
- the procurement step is `pending` and current again (its review is cleared) and is re-dispatched
  to everyone eligible on the property, who get the "Review procurement request" task. A later step
  that had sent the request back is reset to `pending`.

**Resubmission** is the officer approving the pending procurement step again
(`PUT /requests/{id}/steps/{step}` with `status=approved`). On a round after a rebase:

- the next `rfq_vendor_batch_size` suppliers in rotation are asked (never anyone asked on this
  request before), and get `RfqInvitationNotification`;
- the suppliers whose quotes the rebase shelved are asked again, carried over
  (`is_carried_over=true`), and get `RfqReopenedNotification`: the job is open again, the scope may
  have changed, and they may send a revised quote. Suppliers asked before who did not quote are not
  asked again;
- the deadline is `rfq_default_deadline_days` working days out; the form's `deadline`, preferred
  supplier and `override` are ignored.

A carried-over supplier may quote again: the one-quote-per-supplier rule ignores shelved quotes.

Response:

```json
{
  "data": {
    "message": "Procurement request rebased. The invited suppliers have been told it was cancelled.",
    "request": { "...": "the request as GET /requests/{id} returns it" }
  }
}
```

### Quote shortfall: request more quotes / evaluate anyway

At the deadline (`procurement:quotes-review`, daily at 08:00) a work request waiting for quotes with
fewer live quotes (`bid` or `recommended`) than the company's `min_quotes_for_evaluation` (default 3)
is **not opened**. Instead:

- `quotes_shortfall` is set on the request (`awaiting: true`, `received`, `required`, `since`);
- each actor of the procurement step gets the pending task "Only {received} of {required} quotes
  received for {title}" and an in-app `QuotesShortfallNotification`;
- quoting stays closed (the deadline has passed) until the officer decides, and later runs leave the
  request alone.

With enough quotes the request opens as before (system recommendation, `quotes-review`, "Select a
quote" task).

`POST /requests/{procurementRequest}/request-more-quotes` (no body)

- Extends the deadline to `rfq_default_deadline_days` working days from now and clears the shortfall
  and its task.
- Asks the next `rfq_vendor_batch_size` suppliers in rotation, in the same round
  (`RfqInvitationNotification`).
- Tells every supplier already asked whose invitation is open, quoted or not, the new deadline
  (`RfqDeadlineExtendedNotification`).
- When nobody new is left to ask, the deadline still moves and the message says so.

```json
{
  "data": {
    "message": "Quote deadline extended to 21 Oct 2026, 17:00 and 2 more suppliers asked to quote.",
    "invited": 2,
    "request": { "...": "the request as GET /requests/{id} returns it" }
  }
}
```

With nobody new: `"Quote deadline extended to 21 Oct 2026, 17:00. No other prequalified supplier is left to ask."`, `invited: 0`.

`POST /requests/{procurementRequest}/evaluate-anyway` (no body)

Clears the shortfall and opens the quotes received exactly as the deadline would have: the system
recommends one, the request and step move to `quotes-review`, and the officer gets the "Select a
quote" task. With no quote at all it still opens, with nothing recommended.

```json
{
  "data": {
    "message": "The quotes received are open for evaluation.",
    "request": { "...": "the request as GET /requests/{id} returns it" }
  }
}
```

Both are allowed (`permissions.requestMoreQuotes`, `permissions.evaluateAnyway`) for an actor of the
current procurement step holding `update-facility-procurement-request-step`, while the request is
`order`, the step is `waiting-quotes` and a shortfall is recorded; otherwise **403**.
`can_request_more_quotes` / `can_evaluate_anyway` say whether the state allows it. The officer may
also rebase a request held for a shortfall.

**Requests opened before this rule existed.** A request already in quotes review with fewer quotes
than the minimum can be put back to waiting for quotes, shortfall recorded, so the officer gets the
same two choices. On the server:

```
php artisan procurement:hold-short-quotes-review --dry-run   # lists what would move
php artisan procurement:hold-short-quotes-review [--company=ID]
```

Only a review the chain has not acted on qualifies (the procurement step is still current, no quote
is selected, no later step has answered). The system's recommended quote keeps its flag.

### Quote rounds on the request

`GET /requests/{procurementRequest}` adds:

| Field | Type | Notes |
|---|---|---|
| `quote_round` | integer | 1 for the first round, one more after each rebase |
| `is_preliminary` | boolean | The request is with its default supplier alone (see [Default supplier first](#default-supplier-first)) |
| `preliminary_until` | object or null | `{ raw, formatted, diff }`: when the default supplier's window ends; `null` outside the preliminary stage |
| `default_vendor_id` | integer or null | The default supplier the request was sent to first |
| `default_vendor` | object or null | `{ id, name }` of that supplier |
| `quotes_shortfall` | object or null | `{ awaiting: bool, received: int, required: int, since: { raw, formatted } }` while the deadline passed with too few quotes and the officer has not decided; else `null` |
| `rebased` | object or null | `{ at: { raw, formatted }, by: { id, name } or null, reason }` for the latest rebase; else `null` |
| `can_rebase` | boolean | The state allows a rebase (show only) |
| `can_request_more_quotes` | boolean | The state allows requesting more quotes (show only) |
| `can_evaluate_anyway` | boolean | The state allows evaluating anyway (show only) |
| `permissions.rebase`, `permissions.requestMoreQuotes`, `permissions.evaluateAnyway` | boolean | Whether the caller may (state and actor) |

Show the actions when the permission is true. The flags alone carry the state, for disabled
buttons or explanations.

Each entry of `quotes` (and of `GET /requests/{id}/bids`) adds `round` and `shelved`
(`{ at: { raw, formatted }, reason }` or `null`). `status` may be `shelved`
(`{ value: "shelved", color: "warning" }`). A shelved quote cannot be recommended or selected
(`selected_option` naming one returns **422**), is left out of the step form's `meta.quotes`, and
is not rejected again when the LPO is raised.
### Who a step is addressed to

A request step goes to the same people an approval step on the request's property would. The
user named for the role on the property comes first, and allocated holders of the role are the
fallback. A company-wide role goes to its holders allocated to the property. If nobody qualifies,
creating the request, or moving it onto that step, returns **422** keyed on `steps`. The rule
and the error are described in [Approval Steps → Who a step is addressed to](../access-management/approvals.md#who-a-step-is-addressed-to).

## LPOs

Endpoints:

- `GET /lpos`
- `POST /lpos` — raise a **direct** LPO (see below)
- `GET /lpos/{lpo}`
- `PUT /lpos/{lpo}` — edit a **direct** LPO while it is `pending` (see [Edit a direct LPO](#edit-a-direct-lpo))
- `PATCH /lpos/{lpo}/step-fields` — save only the fields an approval step opened
- `DELETE /lpos/{lpo}` — delete an LPO that was never issued (see [Delete an LPO](#delete-an-lpo))
- `PATCH /lpos/{lpo}/cancel` — cancel an LPO the supplier has received (see [Cancel an LPO](#cancel-an-lpo))
- `PUT /lpos/{lpo}/review`

### Permissions

| Permission | Allows |
|---|---|
| `view-facility-procurement-lpo` | List and read LPOs (staff: on allocated properties; suppliers: their own) |
| `create-facility-procurement-lpo` | Raise a direct LPO (`POST /lpos`) |
| `update-facility-procurement-lpo` | Edit a pending direct LPO. The LPO's creator, an approver granted edit rights on the current step, and whoever it was returned to for changes may also edit |
| `delete-facility-procurement-lpo` | Delete a pending or rejected LPO. The LPO's creator may also delete |
| `cancel-facility-procurement-lpo` | Cancel an issued LPO, or a delivered one whose bill is still pending, on an allocated property; shown as `permissions.cancel` |
| `review-document-facility-procurement-lpo` | Accept a supplier's jobcard / delivery note (`PUT /lpos/{lpo}/review`); shown as `permissions.reviewDocument` |
| `submit-doc-facility-procurement-lpo` | Supplier portal: submit the jobcard / delivery note |

### Workflow LPOs and direct LPOs

An LPO comes into being in one of two ways:

| | Workflow LPO | Direct LPO |
|---|---|---|
| Raised by | The procurement request workflow, once a bid (work) or option (purchase) is selected | A staff user, by hand, through `POST /lpos` |
| `is_direct` | `false` | `true` |
| `procurement_request_id` / `procurementRequest` | Set | `null` |
| Items | Copied from the winning bid / selected option | Entered on the form |
| Winning bid document | On the request's bid | `winningBidUpload` |
| Comparable quotes | The request's other bids | `comparables` |
| Approval | Already approved through the request's steps; issued as `lpo` | Goes through the LPO approval template, if one is active (see below) |

Both kinds carry their own `title`, `facility`, `type`, `expenseType` and `expenseSubType`, so lists, filters,
property scoping, bills and the printed LPO treat them the same way.

### List query support

`GET /api/v1/app/{company}/property-management/procurement/lpos`

- Filters:
  - `filter[search]` — free text over id, title, notes, supplier, request title and property name
  - **Exact**: `filter[facility_id]`, `filter[is_direct]` (`1`/`0`), `filter[type]` (`work`, `purchase`)
  - `filter[vendor_id]`, `filter[expense_category_id]`, `filter[expense_type_id]`, `filter[expense_sub_type_id]` (row 169), `filter[status]`, `filter[delivered_at]`, `filter[rating]`,
    `filter[amount]`, `filter[total]`, `filter[created_at]`
- Sort:
  - `sort=id,created_at,delivered_at,amount,total,rating,status`
- Include:
  - `include=items`
- Fields: not supported
- Pagination: `per_page`, `page`

Only LPOs on properties the user is allocated to are listed. `filter[facility_id]` reads the LPO's own
property, so direct LPOs are included.

Example:

`GET /api/v1/app/12/property-management/procurement/lpos?filter[is_direct]=1&filter[type]=work&sort=-created_at`

### Create a direct LPO

`POST /api/v1/app/{company}/property-management/procurement/lpos`

Authorization:

- Requires the `create-facility-procurement-lpo` permission (policy `create`).
- The user must be allocated to `facility_id`, otherwise `403`.

Request body (`CreateDirectLpoData`):

| Field | Required | Type | Allowed Values / Notes |
|---|---|---|---|
| `facility_id` | Yes | integer | Must exist in `facilities.id`. The property the order is for |
| `type` | Yes | string | `work`, `purchase` |
| `title` | No | string | Max 255. Shown on the supplier's jobcard task and the printed LPO |
| `vendor_id` | Yes | integer | Must exist in `users.id`. The supplier |
| `currency_id` | Yes | integer | Must exist in `currencies.id` |
| `expense_type_id` | Yes | integer | Must exist in `facility_expense_types.id` |
| `expense_sub_type_id` | Yes | integer | Must exist in `facility_expense_sub_types.id` and belong to `expense_type_id` |
| `expense_category_id` | No | integer | Must exist in `expense_categories.id`. The LPO's own category - any category may be chosen (e.g. a Repairs & Maintenance order charged to rent rather than service charge). When omitted, the expense type's default category is used |
| `delivery_at` | Yes | string/date | When the work or goods are due |
| `notes` | No | string | - |
| `assigned_technician_id` | No | integer | Must exist in `users.id` |
| `withholding_tax_ids` | No | array of integer | Each must exist in `withholding_taxes.id`, no duplicates. The taxes withheld on the bill generated when the LPO is delivered. When omitted or empty, the bill withholds the property landlord's `withholds` taxes |
| `winning_bid_upload_id` | Yes | integer | Must exist in `uploads.id`. The supplier's winning quotation |
| `comparables` | No | array of integer | Each must exist in `uploads.id` and differ from `winning_bid_upload_id`. The other quotations the winning bid was compared against |
| `items` | Yes | array | At least one item |

`items[]` object — the same shape a supplier sends when submitting a bid:

| Field | Required | Type | Allowed Values / Notes |
|---|---|---|---|
| `type` | Yes | string | `product`, `service` |
| `purchase_item_id` | Yes | integer **or** string | An existing `facility_purchase_items.id`, **or** the name of a new item (see below) |
| `notes` | No | string | - |
| `quantity` | Yes | number | Greater than `0` |
| `stock_keeping_unit_id` | Yes | integer | Must exist in `stock_keeping_units.id` |
| `cost` | Yes | number | Unit cost, minimum `0` |
| `tax_id` | Yes | integer | Must exist in `taxes.id` |
| `discount_amount` | No | number | Discount on the line, minimum `0`. Default `0` |

**Selecting or adding an item.** Send an integer `purchase_item_id` to pick an existing purchase item. Send a
string to add one: if a purchase item with exactly that name already exists it is reused, otherwise a new
purchase item is created (in the first inventory category, with the line's type, unit, tax and cost as its
base price, and the supplier linked to it). The line's `title` is the purchase item's name.

**Line arithmetic** (the same as bids):

- `amount = quantity × cost`
- `amount_after_discount = amount − discount_amount`
- `tax = rate × amount_after_discount` when the tax is discount-deductible, otherwise `rate × amount`
- `total = amount_after_discount + tax`

If the supplier is **not VAT-registered**, every line is stored with the zero-rate tax and `tax = 0`,
exactly as a workflow LPO is. The LPO's `amount`, `discount_amount`, `amount_after_discount`, `tax` and
`total` are the sums of its lines.

**Uploads.**

- `winning_bid_upload_id` is stored on the LPO and returned as `winningBidUpload`.
- `comparables` are attached to the LPO and returned as `comparables`. An upload belongs to one record at a
  time, so attaching one another record holds moves it. An upload created by a different user, or an id that
  does not exist, is rejected with `422` on the `comparables` key.

**Approval.** The LPO is created as `lpo`. If the company has an active approval template for LPOs, and it
forms a chain for this user, the LPO drops to `pending` and waits for that chain:

- While `pending` the supplier cannot see it, and no jobcard task is raised for them.
- On final approval it moves to `lpo` and the supplier gets the "Upload jobcard" task.
- On rejection it moves to `rejected`, which is final: the supplier never sees it, it does not
  count as an award to that supplier, and it can only be viewed or deleted (every other
  `permissions` flag is `false`).

With no template, a bypass role, or no applicable steps, it stays `lpo` and the supplier's jobcard task is
raised at once.

Example request:

```json
{
  "facility_id": 4,
  "type": "work",
  "title": "Replace borehole pump",
  "vendor_id": 118,
  "currency_id": 1,
  "expense_type_id": 7,
  "expense_sub_type_id": 21,
  "delivery_at": "2026-10-10",
  "notes": "Access through the service gate.",
  "assigned_technician_id": 52,
  "winning_bid_upload_id": 9031,
  "comparables": [9032, 9033],
  "items": [
    {
      "type": "product",
      "purchase_item_id": 311,
      "notes": "1.5HP submersible",
      "quantity": 1,
      "stock_keeping_unit_id": 1,
      "cost": 85000,
      "tax_id": 2,
      "discount_amount": 5000
    },
    {
      "type": "service",
      "purchase_item_id": "Pump installation labour",
      "quantity": 1,
      "stock_keeping_unit_id": 3,
      "cost": 15000,
      "tax_id": 2
    }
  ]
}
```

Success response (`DataResource`):

- `message`: `LPO created successfully.`
- `lpo`: the created LPO, with `facility`, `vendor`, `currency`, `expenseType`, `expenseSubType`,
  `expenseCategory`, `assignedTechnician`, `winningBidUpload`, `comparables` and `items` loaded.

```json
{
  "message": "LPO created successfully.",
  "lpo": {
    "id": 412,
    "is_direct": true,
    "type": "work",
    "title": "Replace borehole pump",
    "notes": "Access through the service gate.",
    "amount": "100000.00000",
    "discount_amount": "5000.00000",
    "amount_after_discount": "95000.00000",
    "tax": "15200.00000",
    "total": "110200.00000",
    "status": { "value": "pending", "color": "warning" },
    "delivery_at": { "raw": "2026-10-10T00:00:00.000000Z", "formatted": "10 Oct, 2026", "diff": "2 weeks from now" },
    "facility": { "id": 4, "name": "Kilimani Heights" },
    "vendor": { "id": 118, "name": "Aqua Pumps Ltd" },
    "winningBidUpload": { "id": 9031, "title": "Aqua Pumps quote.pdf" },
    "comparables": [
      { "id": 9032, "title": "Rift Water quote.pdf" },
      { "id": 9033, "title": "Borehole Masters quote.pdf" }
    ],
    "procurementRequest": null,
    "items": [
      { "id": 1201, "title": "Submersible pump", "type": "product", "quantity": "1.000", "cost": "85000.00000", "discount_amount": "5000.00000", "tax": "12800.00000", "total": "92800.00000" },
      { "id": 1202, "title": "Pump installation labour", "type": "service", "quantity": "1.000", "cost": "15000.00000", "discount_amount": "0.00000", "tax": "2400.00000", "total": "17400.00000" }
    ]
  }
}
```

Errors:

- `403` — missing permission, or not allocated to `facility_id`.
- `422` — validation, keyed by field (`items.0.cost`, `comparables.1`, `expense_sub_type_id`, ...).

### Edit a direct LPO

`PUT /api/v1/app/{company}/property-management/procurement/lpos/{lpo}`

Only while the LPO is `pending` (row 138). Only a direct LPO is ever pending — a workflow LPO is issued as
`lpo` straight away — so a workflow LPO is never editable. Once approved the LPO has gone to the supplier
and is not edited; cancel it instead.

Body: **the same payload as [Create a direct LPO](#create-a-direct-lpo)**, sent whole. The items are
replaced by the ones sent and the totals recalculated the same way. `comparables` is the complete list: a
quotation left off is detached. The status stays `pending`.

- An approver whose current step lends them editing rights may edit without the permission; if the step
  limits them to certain fields, a body carrying any other field is refused with `403`, naming the fields
  allowed and the ones refused. Such an approver saves their fields through
  `PATCH /lpos/{lpo}/step-fields` - see
  [approvals](../access-management/approvals.md#saving-only-the-steps-fields-step-fields-row-138).
- Editing an LPO that was returned for changes sends it round its approval chain again.
- The approval chain is told what changed: material fields notify everyone on the chain, the rest notify
  the LPO's creator.

Response `200`:

```json
{
  "data": {
    "message": "LPO updated successfully.",
    "lpo": { "id": 41, "status": { "value": "pending", "color": "warning" }, "...": "as Show an LPO" }
  }
}
```

Errors:

- `403` — `"Only a pending LPO can be edited."`, `"Only a directly raised LPO can be edited."`, no edit
  right, or not allocated to `facility_id`.
- `422` — validation, as on create.

`permissions.update` on the LPO is `true` exactly when this call would be allowed.

### Delete an LPO

`DELETE /api/v1/app/{company}/property-management/procurement/lpos/{lpo}`

Only an LPO that was never issued: `pending` or `rejected`. Its items, its approval chain and the tasks that
chain raised go with it. An LPO the supplier has (`lpo`, `delivered`, `cancelled`) is refused with `403`
`"Only a pending or rejected LPO can be deleted."`.

Response `200`: `{ "data": { "message": "LPO deleted successfully." } }`

`permissions.delete` on the LPO follows the same rule.

### Cancel an LPO

`PATCH /api/v1/app/{company}/property-management/procurement/lpos/{lpo}/cancel`

```json
{ "reason": "Supplier cannot deliver before the end of the month." }
```

`reason` is required (3 to 1000 characters). It is saved on the LPO and sent to the supplier and to the
people notified in-app.

Which LPOs can be cancelled:

| LPO status | Its bill | Cancel |
|---|---|---|
| `lpo` (issued) | none yet | Allowed |
| `delivered` | `pending` | Allowed. The bill is deleted |
| `delivered` | `cancelled` | Allowed. The bill stays as it is |
| `delivered` | `unpaid`, `partially-paid`, `paid` | Refused |
| `delivered` | merged into a bigger bill | Refused |
| `pending`, `rejected` | - | Refused: delete it instead |
| `cancelled` | - | Refused |

How a refusal comes back:

- **The LPO's status** (pending, rejected, already cancelled) is a `403` whose `message` says why.
  `permissions.cancel` is false for these, so the button never shows.
- **The bill** (posted, paid or merged) is a `422` keyed on `status`:

  ```json
  {
    "message": "Bill #41 is unpaid. A delivered LPO can only be cancelled while its bill is pending.",
    "errors": { "status": ["Bill #41 is unpaid. A delivered LPO can only be cancelled while its bill is pending."] }
  }
  ```

  `permissions.cancel` stays true here, so the modal can show why; `cancellation.can_cancel` is false.
- **A missing reason** is a `422` keyed on `reason`.
- **No permission, or not allocated to the property,** is a `403` "This action is unauthorized."

What a cancel does:

- The LPO becomes `cancelled`, with `cancelled_at`, `cancelledBy` and `cancellation_reason`.
- A pending bill is deleted.
- The supplier's "upload jobcard" task is cleared.
- The supplier, the LPO's creator and its approvers are notified (see
  [Notifications when an LPO is cancelled](#notifications-when-an-lpo-is-cancelled)).
- The LPO still prints, under a red **CANCELLED** watermark.

Response `200`:

```json
{
  "data": {
    "message": "LPO cancelled successfully.",
    "lpo": {
      "id": 93,
      "status": { "value": "cancelled", "color": "danger" },
      "cancelled_at": { "raw": "2026-10-20T09:15:00.000000Z", "formatted": "20 Oct, 2026", "diff": "1 second ago" },
      "cancellation_reason": "Supplier cannot deliver before the end of the month.",
      "cancelledBy": { "id": 12, "name": "Jane Wambui" }
    }
  }
}
```

#### The warning before the cancel

`GET /lpos/{lpo}` carries a `cancellation` object, so the confirmation modal can say what will happen to the
bill without another call:

```json
"cancellation": {
  "can_cancel": true,
  "blocked_reason": null,
  "bill": {
    "id": 41,
    "status": { "value": "pending", "color": "warning" },
    "merged_into_bill_id": null,
    "outcome": "deleted"
  },
  "message": "Cancelling this LPO also deletes its pending bill #41. The supplier will be told the order is cancelled."
}
```

- `bill` is `null` when the LPO has no bill (an issued LPO never does).
- `bill.outcome` is `deleted` or `already_cancelled`, and `null` when the cancel is refused.
- `message` is ready to show in the modal: what will happen, or why the cancel is refused
  (the same text as `blocked_reason`).
- Show the Cancel button on `permissions.cancel`: the viewer holds the permission, is allocated to the
  property, and the LPO is issued or delivered. Let the confirm go through only when `can_cancel` is true;
  otherwise show `message`, which says why the bill blocks it.

`cancellation` is on the detail response only, not on list rows.

### Notifications when an LPO is cancelled

| Who | Notification `type` | Channels |
|---|---|---|
| The LPO's supplier | `App\Notifications\PropertyManagement\Procurement\LpoCancelledNotification` | database, mail, SMS |
| The LPO's creator, and whoever started or acted on its approval chain | `App\Notifications\PropertyManagement\DocumentCancelledNotification` | database |
| For an LPO from a procurement request: the request's creator and whoever acted on its steps | `App\Notifications\PropertyManagement\DocumentCancelledNotification` | database |

The supplier's email attaches the cancelled copy of the LPO, with the watermark. Whoever cancelled is not
notified. The in-app notice is the same one every cancelled document sends; see
[Document templates](document-templates-frontend.md) for the list.

`data` for `LpoCancelledNotification`: `lpo_id`, `procurement_request_id`, `facility_name`, `total`,
`delivery_at`, `reason`. For `DocumentCancelledNotification`: `document_type`, `document_id`, `reason`
(may be null), `cancelled_by`.

### Notifications when an LPO is issued

An LPO is *issued* when it reaches `lpo`: a workflow LPO on the step that generates it, a direct LPO on
creation (no chain) or on its final approval.

| Who | Notification `type` | When | Channels |
|---|---|---|---|
| The LPO's supplier | `App\Notifications\PropertyManagement\Procurement\LpoIssuedNotification` | Every issued LPO | database, mail, SMS |
| Every other supplier who quoted on the request | `App\Notifications\PropertyManagement\Procurement\BidRejectedNotification` | Workflow **work** LPOs only | database |
| The procurement request's creator | `App\Notifications\PropertyManagement\Procurement\LpoRaisedNotification` | Workflow LPOs only (a direct LPO's creator raised it themselves) | database |

Mail and SMS go only where the recipient has an address / phone number. The LPO is a document sent to the
supplier, so it is the only one of the three that goes out by SMS; the other two are in-app only.

When a workflow work LPO is generated, every other bid on the request that is not already `rejected` is set
to `rejected`, with `rejection_reason` "Another quotation was selected for this request." The winning bid
is left as it is.

`data` payloads (besides `title`, `message`, `resource_type`, `resource_id`, `resource_url`):

- `LpoIssuedNotification` / `LpoRaisedNotification`: `lpo_id`, `procurement_request_id` (null for direct),
  `facility_name`, `total`, `delivery_at`. `resource_url` opens the LPO in the recipient's portal.
- `BidRejectedNotification`: `bid_id`, `procurement_request_id`, `request_title`, `rejection_reason`.
  `resource_url` opens the supplier's quote.

### Show an LPO

`GET /api/v1/app/{company}/property-management/procurement/lpos/{lpo}`

Loads `currency`, `documentUpload`, `facility`, `expenseType`, `expenseSubType`, `expenseCategory`, `vendor`,
`assignedTechnician`, `winningBidUpload`, `comparables`, `procurementRequest` (null for a direct LPO) and
`items` with `purchaseItem`, `stockKeepingUnit` and `taxType`.

### Response fields

`FacilityProcurementLpoResource`:

| Field | Notes |
|---|---|
| `id` | - |
| `public_url` | The LPO's public page, opened without signing in. See [Public LPO Page](#public-lpo-page) |
| `is_direct` | `true` for an LPO raised through `POST /lpos` |
| `type` | `work`, `purchase` |
| `title` | Every LPO: a direct LPO's as entered; a workflow LPO's is copied from its procurement request |
| `notes`, `amount`, `discount_amount`, `amount_after_discount`, `tax`, `total`, `expense_category_id` | - |
| `withholding_tax_ids` | The withholding taxes passed to the bill (array of ids). Direct LPOs: as entered; workflow LPOs: the property landlord's `withholds` when the LPO was raised. `null` when there are none |
| `delivery_at`, `delivered_at`, `created` | `raw`, `formatted`, `diff` |
| `status` | `{ value, color }` — `pending`, `lpo`, `delivered`, `cancelled`, `rejected` |
| `document_number`, `documentUpload` | The supplier's jobcard / delivery note, set when they submit it |
| `work_advice`, `quality_rating`, `speed_rating`, `communication_rating`, `rating`, `comments` | Set on review |
| `facility` | The property |
| `expenseType`, `expenseSubType`, `expenseCategory`, `currency`, `vendor`, `assignedTechnician` | When loaded |
| `winningBidUpload` | Direct LPOs: the winning quotation |
| `comparables` | Direct LPOs: the comparable quotations (array of uploads) |
| `procurementRequest` | Workflow LPOs only |
| `items` | When loaded |
| `permissions` | Instance permissions |

### Review submitted LPO document

`PUT /api/v1/app/{company}/property-management/procurement/lpos/{lpo}/review`

Purpose:

- Used by App users to accept a vendor-submitted jobcard / delivery document. It marks the LPO `delivered`
  and generates the supplier bill for it (once per LPO).

Who: holders of `review-document-facility-procurement-lpo`, and the LPO's creator (`created_by_id` on the
LPO, else on its procurement request) - `403` otherwise (row 138; before, the call had no check and any
staff user could make it). Gate the Review action on `permissions.reviewDocument`, not
`permissions.update`, which is now `true` only while an LPO is pending.

When the supplier submits the document, the creator gets a "Review document and close LPO" pending task
(app portal, Property Management domain, the LPO as its resource). Accepting the document here clears it,
and so do cancelling or deleting the LPO.

#### Handing migrated procurement to the property

Requests and LPOs imported from EPMAS were created by the migration user (id `1`), so nobody owned
them. A console command gives each one to the property's caretaker (the `facility_role` holder of role
`6`), or to its Property Manager (role `7`) when the property has no caretaker. An LPO that is still `lpo`
and already carries the supplier's document also gets the creator's review task. A record whose property
has neither role holder is listed and left as it is.

```bash
php artisan procurement:reassign-migrated-creators --dry-run      # report only
php artisan procurement:reassign-migrated-creators
php artisan procurement:reassign-migrated-creators --company=3 --from-user=1 --role=6 --fallback-role=7
```

Running it again changes nothing and raises no duplicate tasks.

Request body (`CloseLpoData`; `document_number` and `document_upload_id` are taken from the LPO, and
`delivered_at` is set to now):

| Field | Required | Type | Allowed Values / Notes |
|---|---|---|---|
| `work_advice` | No | string | - |
| `quality_rating` | No | integer | Default `0` |
| `speed_rating` | No | integer | Default `0` |
| `communication_rating` | No | integer | Default `0`. `rating` is the average of the three |
| `comments` | No | string | - |

The bill takes its property and expense type / sub-type from the LPO itself, so direct and workflow LPOs
bill the same way.

The bill withholds the LPO's `withholding_tax_ids` - as entered on a direct LPO, or the landlord's `withholds`
stamped on a workflow LPO when it was raised - when it has any; otherwise the property landlord's
`withholds` taxes, as before.

Success response:

- `message`: `Document accepted!`
- `lpo`: the updated LPO with `documentUpload`

### Public LPO Page

Every LPO has a public link that anyone holding it can open without signing in, for example the
supplier the order was sent to. LPO responses carry it as `public_url`:

```json
{
  "id": 412,
  "public_url": "https://api.example.com/lpos/Yx3...40 random chars"
}
```

The link ends in a random 40-character token, not the LPO id, so LPO numbers cannot be guessed.
The token is issued when the LPO is created; LPOs that existed before this feature were given one by
the migration.

These are web pages served by the backend, not JSON endpoints:

| Route | Returns |
|---|---|
| `GET /lpos/{token}` | HTML page. The left column shows the LPO number, title, order date, delivery date, currency and status; the supplier (KRA PIN, phone, email, address) and the property it is delivered to (address, landlord); the line items (quantity, unit price, amount, tax rate, tax, total) with a footer of Amount, Tax and Total; a tax summary per tax type; the approvals (who prepared the LPO and who approved it, with dates); and the notes, when there are any. The right column shows the order total and delivery date, and a **Print LPO** button |
| `GET /lpos/{token}/pdf` | The LPO as a PDF (`inline`, `LPO0412.pdf`) |

Rules:

- An unknown token returns `404`. So does a `pending` LPO, since it has not been approved yet, and a
  `rejected` one, which never will be. `lpo` (badge "Issued"), `delivered` and `cancelled` LPOs are shown with their
  status badge.
- The PDF prints with the company's default LPO template that the LPO's property can use, else its
  newest active LPO template the property can use.
- **Print LPO** is hidden, and `/pdf` returns `404`, when the company has no active LPO template the
  LPO's property can use.
- Both routes are limited to 60 requests a minute per IP, and are marked `noindex`.

## Contracts

Endpoints:

- `GET /contracts`
- `POST /contracts`
- `GET /contracts/{contract}`
- `PUT/PATCH /contracts/{contract}`
- `DELETE /contracts/{contract}`
- `PATCH /contracts/{contract}/status`
- `POST /contracts/{contract}/generate-bill-for-next-period`

List query support:

- Filters:
  - `filter[search]` — free text over the contract's id, title, supplier name and property name
  - **Exact** (`=`): `filter[id]`, `filter[status]`, `filter[facility_id]`, `filter[vendor_id]`,
    `filter[expense_type_id]`, `filter[expense_category_id]`, `filter[expense_sub_type_id]`, `filter[asset_id]`
  - **Partial** (`LIKE`): `filter[title]`, `filter[start_at]`, `filter[end_at]`, `filter[created_at]`
  - Aliases: `filter[property_id]` → `facility_id`, `filter[supplier_id]` → `vendor_id`
  - `filter[landlord_id]` — **exact**, matched through the contract's property. A contract has no
    landlord of its own; it is raised against a property, and the property belongs to a landlord.
    One spelling only, since landlord is the same word in the UI and the schema.

All filters combine, so `filter[landlord_id]=4&filter[property_id]=12&filter[status]=active`
narrows as you would expect, and any of them can sit alongside `filter[search]`.
- Sort:
  - `sort=id,title,status,start_at,end_at,created_at`
- Include:
  - `include=bills`

> **Exact versus partial matters here.** Ids and `status` are exact: they were previously declared as
> bare strings, which Spatie turns into partial filters, so `filter[facility_id]=6` matched every
> property whose id contained a 6, and `filter[status]=active` also returned `inactive` contracts.
>
> `title` and the dates stay partial on purpose — a title is a text match, and a partial date lets
> `filter[created_at]=2026-09` mean "that month".
>
> The UI's *property* and *supplier* are the schema's *facility* and *vendor*. Both spellings are
> accepted, so either can be used; an unknown filter name is a `400`, not an empty list.

Create/Update payload:

| Field | Required | Type | Allowed Values / Notes |
|---|---|---|---|
| `vendor_id` | Yes | integer | Must exist in `users.id` |
| `facility_id` | Yes | integer | Must exist in `facilities.id` |
| `start_at` | Yes | date (`YYYY-MM-DD`) | - |
| `end_at` | Yes | date (`YYYY-MM-DD`) | Inclusive last day of the contract. `period_in_days` is derived from `start_at`/`end_at` by the backend and stored automatically |
| `type` | Yes | string | `fixed`, `variable` |
| `amount` | Yes | integer | - |
| `expense_type_id` | Yes | integer | Must exist in `facility_expense_types.id` |
| `expense_category_id` | Yes | integer | Must exist in `expense_categories.id` |
| `title` | No | string | Generated when omitted — see below |
| `notes` | No | string | Optional |
| `currency_id` | No | integer | Must exist in `currencies.id` |
| `expense_sub_type_id` | No | integer | Must exist in `expense_sub_types.id` |
| `asset_id` | No | integer | Must exist in `assets.id` |
| `agreement_upload_id` | **Yes on create** | integer | Must exist in `uploads.id`. See below |
| `uploads` | No | array of integer | Supporting documents — addendums, extensions. Each must exist in `uploads.id`. See below |
| `has_tax` | No | boolean | Default `false` |
| `tax_id` | No | integer | Must exist in `taxes.id` |
| `creator_id` | No | integer | Must exist in `users.id` |
| `approver_id` | No | integer | Must exist in `users.id` |

### Titles are generated, not typed

The create and edit forms no longer ask for a title. When `title` is omitted or sent empty, one is
generated from the expense type and the property — *"Lift Maintenance Service Contract at Riverside
Plaza"*. A title that **is** sent is used as given, so nothing breaks if you keep sending one.

On update, an omitted or empty `title` **keeps whatever is stored**. It is never cleared. That
matters beyond display: contract sync recognises a contract it has already created **by title**, so
a blank one would make it create a duplicate instead of updating, and the contracts list searches,
filters and sorts on the same field.

A contract that somehow has no stored title gains a generated one the next time it is saved.
### The signed document is required

A contract cannot be **created** without its signed document. Creating one without it returns `422`
with the error keyed on the upload field.

On **update** the rule holds only where it can:

| Situation | Behaviour |
|---|---|
| The contract already has a document | It may be **replaced**, but sending `null` is refused |
| The field is omitted from the payload | The stored document is kept — omitting is not clearing |
| The contract has no document | The update is allowed, so one can be attached from the edit page |

Contracts already in the system predate this rule and mostly carry no document. Requiring one on
every save would make them uneditable until the paperwork was found, which is why they stay
editable instead.

**Contracts the system creates are not affected** and need no upload: a facility's management
contract created at facility setup, and contracts produced by utility and service contract sync.
Do not add an upload step to those flows.

Supporting documents in `uploads[]` — addendums, extensions — remain optional. A contract with
addendums but no signed document is still refused.


`status` is **no longer accepted** on create or update. A contract's status belongs to its
approval chain - see [Approval](#approval) - and the only moves left in a person's hands are
between `active` and `suspended`, through the status endpoint below.

(Previously `status` was accepted here and defaulted to `pending`, which meant an update that
left it out quietly sent a live contract back to pending. It is now ignored entirely.)

### Status update

`PATCH /api/v1/app/{company}/property-management/procurement/contracts/{contract}/status`

Authorized against `update-status-facility-contract` (it previously checked
`update-facility-contract`).

| Field | Required | Type | Allowed Values |
|---|---|---|---|
| `status` | Yes | string | `active`, `suspended` |

Allowed transitions - everything else returns **422**:

| From | To | |
|---|---|---|
| `active` | `suspended` | allowed |
| `suspended` | `active` | allowed |
| anything | `pending` | refused - the chain raises a contract into pending |
| anything | `active` | refused - approving the chain is what makes a contract active |
| anything | `inactive` | refused |
| anything | `rejected` | refused - rejecting the chain is what makes a contract rejected |
| `rejected` | anything | refused - a rejected contract can only be viewed or deleted |
| anything | `expired` | refused - derived from `end_at`, never stored |

`expired` is never written to the database. An `active` contract whose `end_at` has passed
reads as `expired` in reports, computed from the date.

### Approval

`FacilityContract` is a registered approvable model (`config('approvals.models')`) and is
driven by the shared approval framework. Act on a contract's chain through
`POST /access-management/approval-steps/{approvalStep}` - see
[Approval Steps](../access-management/approvals.md). Configure the chain itself under
[Approval Templates](../access-management/approval-templates.md).

Only `POST /contracts` raises a chain. Contracts created by the system - the utility and
service contract sync that runs off a facility's setup, and the EPMAS importers - are in force
the moment they are written and never enter one.

Every contract show response from the staff endpoints (lists only with `?with_approval_steps=1`) carries an `approval_steps` array in the same
shape as every other approvable resource. The **vendor**-facing contract endpoints do not: a
supplier does not see the company's internal approvers.

#### Condition facts

Fields a template step's `conditions` may be written against. Operators and the rule shape are
documented once in
[Approval Templates → Step conditions](../access-management/approval-templates.md#step-conditions).

| Fact | Meaning |
|---|---|
| `amount` | The contracted figure - the usual threshold to escalate on |
| `type` | `fixed` or `variable` |
| `billing_cycle` / `billing_period` | How often a fixed contract bills |
| `facility_id` | The property the contract is for |
| `vendor_id` | The supplier |
| `expense_type_id` / `expense_category_id` | What the spend is classified as |
| `currency_id` | The contract currency |
| `has_tax` / `tax_id` | Whether the contract is taxed, and at what rate |

#### Status transitions

| Action | The contract |
|---|---|
| Created, company has no active template | `active` immediately, no chain |
| Created, all steps fall away on conditions | `active` immediately, no chain |
| Created by a holder of a bypass role | `active` immediately, no chain |
| Created, a chain applies | `pending`, chain raised, first step prompted |
| `approve`, not the last step | stays `pending`, next step prompted |
| `approve`, the last step | `active`, `FacilityContractApprovedEvent` fires |
| `review` | stays `pending`, the previous step is reopened |
| `reject` | `rejected` - final; view and delete only |
| `PATCH {contract}/status` | `active` ↔ `suspended` only |

A `pending` contract is not yet in force: it does not bill, and
`POST {contract}/generate-bill-for-next-period` refuses it, as does the `contracts:generate-bills`
scheduler. The facility's expenditure budget follows the same line - a contract only counts
towards the contracted floor once its chain has made it `active`, and drops back out if it is
rejected.

### Generate bill for next period

`POST /api/v1/app/{company}/property-management/procurement/contracts/{contract}/generate-bill-for-next-period`

Purpose:

- Manually generate the upcoming period's bill for a contract ahead of the scheduled
  billing date (`next_billing_at`), when operationally necessary. Runs the same generation
  as the `contracts:generate-bills` scheduler.

Request body:

- None.

Eligibility (returns `422` validation error otherwise):

- Contract `type` must be `fixed` with `amount > 0`.
- Contract `status` must be `active`.
- Contract must have a billing schedule (`next_billing_at` set).
- Unlike the scheduler, the "not yet due" check is **skipped** — `next_billing_at` may be in the future.

Behavior:

- The bill `notes` describe the billing **period** being generated for (derived from `next_billing_at` and
  the billing cycle), not the date it was generated.
- `next_billing_at` is advanced by one billing cycle when the bill is created.

Success response:

- `message`: `Bill for next period generated successfully.`
- `bill`: the created `FacilityBill`.


### Supporting documents

`uploads` is a list of `uploads.id` values — **addendums, contract extensions and anything else the
contract picks up over its life**. It sits beside the single signed document above, which is
unchanged: that column still holds the agreement itself, and is never part of this list.

| Behaviour | Result |
|---|---|
| `uploads: [1, 2]` | Those two are attached |
| `uploads: [1]` on a contract holding 1 and 2 | **2 is released** — the list is the complete set, not an addition |
| `uploads` **omitted entirely** | Attachments are left untouched — safe for a partial update |
| `uploads: []` | All are released |

Released means **unowned, not deleted**: the file still exists and can be attached elsewhere.

An upload may belong to one record at a time, so attaching one that another contract holds moves it.
An upload created by a different user is rejected with `422` on the `uploads` key, as is an id that
does not exist.

Documents keep the name they were uploaded with — there is no document type to set.

The response exposes them as `uploads`, an array of the standard upload resource.

## Inventories and Purchase Items

Inventories:

- Implemented endpoint:
  - `GET /inventories`
- Note:
  - Other inventory routes exist but are currently not implemented in the controller.

Inventory list query support:

- Filters:
  - `filter[facility_id]`, `filter[inventory_category_id]`, `filter[name]`
  - `filter[current_stock]`, `filter[storage_room_id]`, `filter[shelf_id]`

Purchase items:

- Implemented endpoints:
  - `GET /inventories/purchase-items`
  - `POST /inventories/purchase-items`
  - `GET /inventories/purchase-items/{purchase_item}`
  - `PUT/PATCH /inventories/purchase-items/{purchase_item}`
  - `DELETE /inventories/purchase-items/{purchase_item}`
  - `PUT/PATCH /inventories/purchase-items/{item}/prices/{price}`

Purchase item list query support:

- Filters:
  - `filter[search]` - free text over the item name, its inventory category name and its SKU
    name; a whole-number word also matches the item id. Several words must all match; several
    comma-separated terms match any of them.
  - `filter[id]`, `filter[inventory_category_id]`, `filter[name]`, `filter[currency_id]`,
    `filter[tax_id]`, `filter[base_price]`
  - `filter[vendor_id]`, `filter[facility_id]` - items with a price row for that vendor / property
- Sort: always `-created_at, -id`.

Purchase item create payload:

| Field | Required | Type | Allowed Values / Notes |
|---|---|---|---|
| `type` | Yes | string | `service`, `product` |
| `name` | Yes | string | Must be unique |
| `inventory_category_id` | Yes | integer | Must exist in `inventory_categories.id` |
| `stock_keeping_unit_id` | No | integer | Must exist in `stock_keeping_units.id` |
| `tax_id` | No | integer | Must exist in `taxes.id` |
| `currency_id` | No | integer | Must exist in `currencies.id` |
| `is_regulated` | No | boolean | Default `false` |
| `base_price` | No | number | Default `0`. On update, a changed value is written to **every** supplier's price row at every property |
| `vendors` | No | array | Supplier user ids. Each must belong to the supplier group and must not be **terminated** |

**Who may be attached.** `vendors` was only checked against `users.id`, so any user could be named -
a tenant, a landlord, a member of staff - and a price row was then created for them on every
active property. Each id must now be a supplier, and a terminated supplier is refused with
*"This supplier has been terminated and cannot be given new work."*

Errors are keyed by position, so the offending row can be marked: `vendors.0`, `vendors.2`.

A **suspended** supplier is still accepted. Suspension is a pause rather than an ending, and the
same holds on bills, contracts and LPOs.

Purchase item update payload:

- Same shape and validation rules as create payload.
- Success response message: `Purchase Item updated successfully.`
- Error message (server error): `Error updating Purchase Item`

Purchase item delete response:

- Success response message: `Purchase Item deleted successfully.`
- Error message (server error): `Error deleting Purchase Item`

Purchase item price update payload:

| Field | Required | Type |
|---|---|---|
| `price` | Yes | number |

- Success response message: `Purchase Item Price updated successfully.`
- Error message (server error): `Error updating Purchase Item Price`

### Purchase item prices

A purchase item has **one price row per (item, supplier, property)**. The database enforces it with
a unique key on (`purchase_item_id`, `vendor_id`, `facility_id`), so a supplier can never appear
twice at a property, and never twice as an option on a purchase request.

| Behaviour | Result |
|---|---|
| A supplier is added to `vendors` | A row is created for them at every active property, priced at the item's `base_price` (`0` when none) in the item's currency |
| A supplier stays in `vendors` | Their rows and prices are left alone; sending the same list again creates nothing |
| A supplier is removed from `vendors` | Their rows are deleted **at every property**, and their option disappears from purchase requests still waiting on the procurement step |
| `vendors: []` | Every price row of the item is deleted |
| `vendors` omitted on update | Suppliers and prices are left untouched |
| `base_price` changes on update | Every price row of the item, for every supplier at every property, is set to the new base price (regulated or not); options of purchase requests still waiting on the procurement step follow. Sending the same `base_price` back changes nothing |

`vendors` is the **complete list**, not an addition, the same as `uploads` on contracts.

`price` is a number in currency units (up to 5 decimals); the response returns it as a decimal
string, e.g. `"82.50000"`. A regulated item refuses a price above its `base_price` with `422` on the
`price` key.

### Maintenance commands

#### Removing duplicate price rows

Price rows created before the unique key existed could repeat (item, supplier, property). The
migration that adds the key removes the duplicates itself; the command below repeats that, and
also repairs rows stamped with the wrong company and lists rows of suppliers no longer on the item.

```
php artisan purchase-items:dedupe-prices --dry-run                  # list everything, change nothing
php artisan purchase-items:dedupe-prices                            # dedupe and restamp
php artisan purchase-items:dedupe-prices --company=3                # one company's items
php artisan purchase-items:dedupe-prices --dry-run --purge-orphans  # list the orphans that would go
php artisan purchase-items:dedupe-prices --purge-orphans            # also delete the orphans
```

- **Duplicates.** A group keeps the row with the latest `updated_at` (ties to the highest id, a row
  without `updated_at` losing to any dated row) and deletes the rest. The kept row is the one
  touched last, not the cheapest; the *Prices differ* column flags groups whose rows disagreed.
- **Restamp.** A row whose `company_id` is empty or differs from its item's gets the item's
  company, so it shows up under the company that owns the item.
- **Orphans.** A row whose supplier is not in the item's `vendors` list is always listed, and is
  deleted only with `--purge-orphans`. Prices imported per property and supplier lists imported
  from supplier links were loaded separately, so read the dry run before purging: a purge deletes
  real prices if a supplier is missing from `vendors` by mistake.
- `--company` limits every phase to the purchase items of that company (by the item's company, not
  the price row's). Each phase runs in its own transaction, and a second run reports nothing.

#### Recomputing purchase price options

Brings the price options of every purchase request whose `handle_procurement` step is current and
`pending` back in line with the purchase item prices, the same reconcile the step fetch runs (see
[Price options follow the catalogue](#price-options-follow-the-catalogue)). Use it after prices were
changed outside the app, or to refresh options built before options followed the catalogue.

```
php artisan procurement:recompute-purchase-options --dry-run              # list what would change, write nothing
php artisan procurement:recompute-purchase-options                        # every pending purchase request
php artisan procurement:recompute-purchase-options --request=12 --request=15  # only these requests
php artisan procurement:recompute-purchase-options --company=3            # one company's requests
```

- Prints a table of `Request | Property | Options before | Added | Updated | Removed | Pick`, then
  totals. `Pick` is `rerun` (no option selected and the options changed, so the system picked
  again), `kept` (an option is selected, or nothing changed), or `none` (no options).
- Each request is reconciled in its own transaction; a dry run rolls each one back.
- A `--request` id that is not a purchase request waiting on its procurement step is listed as
  skipped. Requests whose step has moved on are never touched.

## Errors

Use shared behavior in `docs/frontend/app/README.md`:

- Render `4xx` messages and validation errors.
- Show generic fallback for `5xx`.

## LPO dates

**`requested_at` and `created_at`.** `requested_at` is when the LPO was entered. `created_at` is
when it was processed, which is the day of its final approval when it needs one. Filter on entry
with `filter[requested_between]=Y-m-d,Y-m-d`. See [Document dates](document-dates.md). The LPO resource names its processing date `created`.
