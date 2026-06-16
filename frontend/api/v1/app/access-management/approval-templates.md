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
| `name` | string | Yes | - | Template display name |
| `model_type` | string | Yes | - | Must be one of `/approvable-models` classes |
| `initial_status` | string | Yes | - | Must be one of selected model statuses |
| `final_status` | string | Yes | - | Must be one of selected model statuses |
| `steps` | array[object] | Yes | - | Ordered approval steps |
| `steps[].step_order` | int | Yes | - | Must be sequential starting at `1` |
| `steps[].role_id` | int | Yes | - | Company app-role ID to act at this step |
| `always_requires_approval` | boolean | No | `false` | If `true`, bypass roles are ignored |
| `bypass_role_ids` | array[int] | No | `[]` | Roles allowed to bypass workflow |
| `post_approval_event` | string \| null | No | `null` | FQCN event class; must exist |
| `is_active` | boolean | No | `true` | Only one template per model/company |

Example request:

```json
{
  "name": "Tenant Invoice Approval",
  "model_type": "App\\Models\\FacilityInvoice",
  "always_requires_approval": false,
  "bypass_role_ids": [11],
  "initial_status": "pending",
  "final_status": "unpaid",
  "steps": [
    { "step_order": 1, "role_id": 21 },
    { "step_order": 2, "role_id": 22 }
  ]
}
```

## Update Template

`PUT/PATCH /api/v1/app/{company}/access-management/approval-templates/{approvalTemplate}`

Update payload accepts any create fields except `model_type` (partial update supported).

Important:

- `model_type` is immutable after template creation.
- If `steps` are updated, only future approvals use the new template steps.
- Existing `approval_steps` already created for in-flight approvables remain unchanged.

## Delete Template

`DELETE /api/v1/app/{company}/access-management/approval-templates/{approvalTemplate}`

Behavior:

- Soft delete only.
- Delete is blocked if template has active pending approval steps (`422`).
