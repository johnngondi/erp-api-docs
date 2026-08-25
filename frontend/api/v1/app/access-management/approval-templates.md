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
