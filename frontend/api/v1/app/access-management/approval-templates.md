# Access Management: Approval Templates

Base prefix:

`/api/v1/app/{company}/access-management`

## Endpoints

Implemented:

- `GET /approval-templates`
- `POST /approval-templates`
- `GET /approval-templates/{approvalTemplate}`
- `PUT/PATCH /approval-templates/{approvalTemplate}`
- `DELETE /approval-templates/{approvalTemplate}`

## List Templates

`GET /api/v1/app/{company}/access-management/approval-templates`

Supported query params:

- Filters:
  - `filter[name]` (partial)
  - `filter[model_type]` (exact FQCN)
  - `filter[is_active]` (`true|false`)
  - `filter[created_at]` (date: `YYYY-MM-DD`)
- Sorts:
  - `sort=name`
  - `sort=model_type`
  - `sort=is_active`
  - `sort=created_at`
- Pagination:
  - `per_page`, `page`

## Create Template

`POST /api/v1/app/{company}/access-management/approval-templates`

Create payload table:

| Field | Type | Required | Default | Notes |
| --- | --- | --- | --- | --- |
| `model_type` | string | Yes | - | Must be one of `/approvable-models` classes |
| `steps` | array[object] | Yes | - | Ordered approval steps |
| `steps[].step_order` | int | Yes | - | Must be sequential starting at `1` |
| `steps[].role_id` | int | Yes | - | Company app-role ID to act at this step |
| `steps[].conditions` | object \| null | No | `null` | When this step applies — see [Step conditions](#step-conditions) |
| `steps[].allowed_to_edit` | boolean | No | `false` | Lets this step's actor edit the resource while it waits on them |
| `steps[].can_edit_approvable` | boolean | No | `false` | The newer name for `allowed_to_edit`; send either. When both are sent, this one wins |
| `steps[].editable_fields` | array[object \| string] \| null | No | `null` | Which fields that grant covers, each `{ "field": "notes", "is_material": false }` (a bare `"notes"` still works) — see [The edit grant](#the-edit-grant) |
| `steps[].is_checkpoint` | boolean | No | `false` | A later step's push-back lands here — see [Checkpoints](#checkpoints) |
| `always_requires_approval` | boolean | No | `false` | If `true`, bypass roles are ignored |
| `bypass_role_ids` | array[int] | No | `[]` | Roles allowed to bypass workflow |
| `post_approval_event` | string \| null | No | `null` | FQCN event class; must exist |
| `is_active` | boolean | No | `true` | Only one template per model/company |

Notes:

- `name` is **not** accepted in the request. It is derived from `model_type` and stored as `"<Model Label> Approval Template"` (e.g. `"Facility Invoice Approval Template"`). It is still returned in responses.
- The resource statuses are **not** configured on the template. Each approvable model declares them as constants (`INITIAL_STATUS_ON_CREATE`, `FINAL_STATUS_ON_APPROVAL`, `FINAL_STATUS_ON_REJECTION`) and the system applies them automatically as the resource is created, approved, or rejected.

Example request:

```json
{
  "model_type": "App\\Models\\FacilityInvoice",
  "always_requires_approval": false,
  "bypass_role_ids": [11],
  "steps": [
    { "step_order": 1, "role_id": 21, "is_checkpoint": true },
    {
      "step_order": 2,
      "role_id": 22,
      "can_edit_approvable": true,
      "editable_fields": [
        { "field": "due_at", "is_material": true },
        { "field": "notes", "is_material": false }
      ]
    }
  ]
}
```

Each step in the response (on the template and on a resource's `approval_steps`) carries:

| Field | Notes |
| --- | --- |
| `allowed_to_edit` | Whether the step lends its actor the right to edit |
| `can_edit_approvable` | Same value as `allowed_to_edit` |
| `editable_fields` | Always objects `{ field, is_material }`, whatever shape was sent; `null` when no list was given |
| `editable_field_names` | Just the names; `null` means **every field** (no list, `[]`, or `*`) |
| `is_checkpoint` | Whether a later step's push-back lands here |

## Update Template

`PUT/PATCH /api/v1/app/{company}/access-management/approval-templates/{approvalTemplate}`

Update payload accepts the create fields except `model_type` (partial update supported). `name` is derived and cannot be set; resource statuses live on the model, not the template.

Important:

- `model_type` is immutable after template creation, so the derived `name` never changes.
- If `steps` are updated, only future approvals use the new template steps.
- Existing `approval_steps` already created for in-flight approvables remain unchanged.

## Delete Template

`DELETE /api/v1/app/{company}/access-management/approval-templates/{approvalTemplate}`

Behavior:

- Soft delete only.
- Delete is blocked if template has active pending approval steps (`422`).

## Step conditions

`steps[].conditions` decides whether a step is in a given resource's chain. It is
evaluated **once, when the chain is initiated** — editing the resource afterwards
does not reshape a chain somebody is part-way through.

```json
{
  "mode": "bypass",
  "match": "all",
  "rules": [
    { "field": "monthly_rent",  "operator": "less_than", "value": 50000 },
    { "field": "facility_type", "operator": "equals",    "value": "Residential" }
  ]
}
```

Hang that on a CEO step and a residential offer under 50,000 a month is written
without one. A bare list of rules is accepted as shorthand for the same thing with
the defaults filled in. `null` or no rules means an unconditional step.

| Key | Values | Default | Meaning |
| --- | --- | --- | --- |
| `mode` | `bypass` \| `require` | `bypass` | matched means **skip** the step / matched means **include** it |
| `match` | `all` \| `any` | `all` | how the rules combine |
| `rules[].field` | see below | - | what is being asked about |
| `rules[].operator` | see below | `equals` | the comparison |
| `rules[].value` | scalar, or a list for `in`/`not_in` | - | what it is compared against |

Both readings of "conditional step" are in circulation, which is why `mode` is
stored rather than inferred — a step that does not say which it means is a chain
that silently comes out wrong.

**Operators** (`App\Enums\ApprovalConditionOperator`) — `equals`, `not_equals`,
`greater_than`, `greater_than_or_equal`, `less_than`, `less_than_or_equal`, `in`,
`not_in`. Equality is the way a settings screen means it: `50000` equals
`"50000"`, `true` equals `"true"`, and `"Residential"` equals `"residential"`.

**Fields** — every scalar column on the resource, plus whatever derived facts the
model exposes. `GET /approvable-models` lists the column names per model in its
`fields` array; derived facts are documented with the model (see, for example,
the [LOO's condition facts](../../property-management/lease-management/loo/loo.md#condition-facts)).

Two deliberate asymmetries, both erring towards **asking for the signature**:

- a rule naming a field that does not exist **keeps** the step, in either mode. A
  typo in configuration must not delete an approval nobody notices is missing.
- a null on the resource's side **never matches**. An unpriced document has no
  monthly rent, and "not known" is not "under the threshold".

### What the chain looks like

Only applicable steps are written. A bypassed step leaves **no row at all**, so
`approval_steps` reads as the signatures still owed rather than a mixture of those
and ones nobody will ever be asked for. `step_order` is copied from the template
and may therefore have gaps — steps 1, 2, 3 with the second bypassed materialise
as 1 and 3, and `is_last` is set on the last *materialised* step.

If **no** step applies, the resource is treated as fully approved — the same
outcome as a bypass role. A chain with nothing in it cannot withhold approval.

## The edit grant

`steps[].allowed_to_edit` hands this step's actor the right to update the resource
while the chain is waiting on them, **even without the resource's own update
permission**. The right is borrowed, not kept: it arrives when the document
reaches that person and is gone the moment they approve, review or reject it.

`steps[].editable_fields` scopes that right. `null`, `[]` and `["*"]` all mean
"every field"; anything else is the allow-list. A submitted field outside it is
refused with **403**, naming the offending fields — the request is rejected whole
rather than silently half-applied.

The allow-list scopes only the **borrowed** right. Somebody who already holds the
resource's update permission edits as they always could.

Field names are validated on the way in against `GET /approvable-models` → the
model's `fields` array, so a step cannot grant editing over something the resource
does not expose.

### Shape and the `can_edit_approvable` alias

Each `editable_fields` entry is an object, `{ "field": "due_at", "is_material": true }`.
`is_material` defaults to `false`. A bare string (`"due_at"`) is still accepted and
reads as a non-material field, so templates saved before materiality keep working;
responses always come back as objects. Anything else - a number, an object without
`field`, an `is_material` that is not a boolean - is a **422** on that entry.

`can_edit_approvable` is the same switch as `allowed_to_edit` (it is stored in
`allowed_to_edit`). Send either; both come back.

### Edit notifications and materiality

Whenever a resource is edited through its update endpoint while its chain is open
(a step `pending` or sent back for changes), the people behind the chain are told
with an in-app **"Edited during approval: {resource}"** notification. Its `data`
carries `fields` (the columns that changed) and `is_material`.

| Who | When |
| --- | --- |
| The resource's creator | Every edit |
| Whoever submitted it for approval, and everybody who has already answered a step | Only when the edit is **material** |

An edit is material when a changed field is marked `is_material: true` on **any**
step of the chain (`{ "field": "*", "is_material": true }` makes every field
material). The editor is never told of their own edit. A value that only changed
shape (`100` written over `100.00000`) is not a change, and `status` and the
timestamps are never counted. This applies to every approvable, whoever made the
edit - a step-granted editor, a permission holder, or a creator answering a
push-back.

## Checkpoints

`steps[].is_checkpoint` decides where a push-back lands. When a step answers
`review` (`POST approval-steps/{step}` with `status: review` and a `comment`):

- **A checkpoint before it** - the chain goes back to the **nearest** one. That step
  and every step after it, up to the reviewer, are reset to `pending` (their
  `acted_by`, `acted_at` and `comment` cleared, actors re-resolved) and the
  checkpoint is put in front of **everybody eligible** for its role on the
  property. The reviewer's own step stays `review` with their comment; when the
  chain is approved back up to it, it is asked again rather than skipped.
- **No checkpoint before it** - the resource goes back to its **creator and
  submitter**, from whichever step asked (not only the first). Each gets a
  "Changes requested" task and notification. The earlier steps keep their
  approvals for now, the reviewer's step is the current one with `is_returned:
  true`, and nobody may act until the resource is edited.
- **Resubmission** - editing a returned resource through its update endpoint resets
  every step up to and including the one that asked, and starts the chain again
  from the first step, put in front of everybody eligible (not only whoever
  approved it last time). Their task reads "Resubmitted after {reviewer} asked for
  changes: {comment}".

A checkpoint the resource's [conditions](#step-conditions) bypassed left no row,
so it cannot be landed on - the next checkpoint before it is used, or the
resource goes back to its creator.

Reset steps lose `acted_by` and `comment` on the row; who answered them before is
kept in the activity log.

The flag is copied onto each `approval_steps` row when the chain starts, like the
edit grant, so changing the template does not move a chain already in flight.

## Attempts

A resource may be in only one chain at a time; initiating over an open chain is a
**422**. Rejecting sends the resource to `rejected` and ends that chain for good:
a rejected resource cannot be initiated again (also a **422**), and the corrected
document is raised as a new record with a chain of its own.

`attempt` appears on every `approval_steps` row. Everything that means "the chain"
— the embedded `approval_steps` array on a resource, `is_current`, `can_act` —
refers to the **highest** attempt. Records rejected before `rejected` became
terminal may carry more than one attempt; new chains only ever have attempt `1`.
