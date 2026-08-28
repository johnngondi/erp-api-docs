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
| `steps[].editable_fields` | array[string] \| null | No | `null` | Which fields that grant covers — see [The edit grant](#the-edit-grant) |
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
    { "step_order": 1, "role_id": 21 },
    { "step_order": 2, "role_id": 22 }
  ]
}
```

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
does not reshape a chain somebody is part-way through; re-submitting it does.

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

## Attempts

A resource may be in only one chain at a time; initiating over an open chain is a
**422**. Rejecting sends the resource to its `FINAL_STATUS_ON_REJECTION` and ends
that chain — re-submitting afterwards opens a new **`attempt`** rather than
overwriting the old one, so who rejected attempt 1 and why stays on the record.

`attempt` appears on every `approval_steps` row. Everything that means "the chain"
— the embedded `approval_steps` array on a resource, `is_current`, `can_act` —
refers to the **highest** attempt. A re-submitted chain is also re-evaluated
against the resource as it now stands, so a figure corrected after a rejection can
change which steps apply.
