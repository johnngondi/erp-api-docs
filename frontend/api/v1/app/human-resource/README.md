# Human Resource API (Frontend)

Base prefix:

`/api/v1/app/{company}/human-resource`

`{company}` is the current app/company context. Shared auth, tenancy, query and error handling
guidance: `docs/frontend/api/v1/app/README.md`.

## Two spaces

| Space | Prefix | Who | Gate |
| --- | --- | --- | --- |
| Admin (supervisors, HR) | `/human-resource/admin` | Staff with HR permissions | Every route needs its own permission (`403` without it). |
| Staff (every employee) | `/human-resource/me` | Any employee of the company who is **on probation** or **active** | **No permission at all.** |

### The staff space needs no permissions

Every working employee uses their own self-service (profile, leave, attendance), even with no
roles. Do not hide the staff space behind a permission; show it to anyone the API lets in.

The API answers `403` when the caller cannot use it:

| Case | `message` |
| --- | --- |
| No employee record in this company | `You do not have an employee record in this company.` |
| Employee is a draft, suspended or terminated | `Your employee record is not active, so the staff space is closed to you.` |

Every `me/*` endpoint answers for the caller's own employee record only. There is no id in the
URL to change.

### Acting for another employee (impersonation)

A holder of `impersonate-employee` can open the staff space as another employee and act for them.
Send the header on every `me/*` request:

```
X-Acting-Employee: 45
```

- `45` is the **employee id** (`employees.id`), not a user id.
- The employee must belong to the same company and be on probation, active or suspended. A draft,
  a terminated employee, an unknown id or another company's employee answers `404`
  (`The employee you are acting for was not found in this company.`).
- Without `impersonate-employee` the header answers `403`
  (`You are not allowed to act on behalf of another employee.`), even for a caller who is an
  employee.
- The impersonator does not need an employee record of their own.
- Every change made this way is recorded in the audit trail under the real user, with the employee
  acted for (`on_behalf_of_employee_id`).
- `GET /human-resource/me` returns `acting_on_behalf: true` while acting, so the UI can show a
  banner ("You are acting as …").

`impersonate-employee` is in `GET /me/permissions` under `modules["Human Resource"]`, so the
"Act as employee" action can be gated on it.

## Permissions (tag `Human Resource`)

| Permission | Gates |
| --- | --- |
| `view-human-resource-dashboard` | `GET /human-resource/admin` |
| `view-human-resource-setting` | `GET /human-resource/admin/settings` |
| `update-human-resource-setting` | `PUT /human-resource/admin/settings` (not in `me/permissions`; read `can_update` from the settings response) |
| `impersonate-employee` | The `X-Acting-Employee` header on `me/*` |
| `view-employee` | Employee list, summary and profile; status history; manager options |
| `create-employee` | `POST /human-resource/admin/employees`; with `update-employee`, either opens `employees/form-options` |
| `update-employee` | `PATCH /human-resource/admin/employees/{employee}`, employee documents |
| `delete-employee` | `DELETE /human-resource/admin/employees/{employee}` (drafts only) |
| `suspend-employee` | `PATCH /human-resource/admin/employees/{employee}/suspend` and `/suspension` (change the last day) |
| `activate-employee` | `PATCH /human-resource/admin/employees/{employee}/activate` |
| `terminate-employee` | `PATCH /human-resource/admin/employees/{employee}/terminate` |

`GET /me/permissions` returns the `view-*`, `create-*` and `impersonate-*` names under `modules["Human Resource"]`.
Suspend, activate and terminate are not in it: read them from the employee's own `permissions`,
which also say whether the action would succeed for that employee.

## Module access

Every route under `/human-resource` also checks the caller's module access (`module.access`).
Human Resource is open to every staff member, so in practice nobody is refused here; the same
check will guard the other modules later. `GET /me/permissions` returns the caller's modules as
`module_access`.

## Modules

- Dashboards (Admin and Staff): `docs/frontend/api/v1/app/human-resource/dashboard.md`
- HR General Settings: `docs/frontend/api/v1/app/human-resource/settings.md`
- Employees (list, summary, profile, create, update, delete, manager options, documents): `docs/frontend/api/v1/app/human-resource/employees.md`
- My profile (staff space): `docs/frontend/api/v1/app/human-resource/profile.md`
- Attendance channels (biometric and QR setup, service catalogue, test connection, device IDs), with
  their four permissions: `docs/frontend/api/v1/app/human-resource/attendance-channels.md`

Attendance recording and leave are added by the next tickets of this batch.

## Shared values

Employee `status` and `employment_type` are returned as `{ value, color, label }`:

| `status` | Meaning |
| --- | --- |
| `draft` | Saved without a user yet. Never in headcount. |
| `on_probation` | Working, within the probation period. |
| `active` | Working. |
| `suspended` | Temporarily out; no access to PropERP or the staff space. |
| `terminated` | Gone. Final. |

| `employment_type` |
| --- |
| `permanent`, `contract`, `intern`, `casual` |
