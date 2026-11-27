# HR General Settings (Frontend)

One set of settings per company, under HR Settings › General.

| Endpoint | Permission |
| --- | --- |
| `GET /api/v1/app/{company}/human-resource/admin/settings` | `view-human-resource-setting` |
| `PUT /api/v1/app/{company}/human-resource/admin/settings` | `update-human-resource-setting` |

`update-human-resource-setting` is not in `me/permissions` (only `view-*`, `create-*` and a few
other prefixes are). Show the Save button from `can_update` in the `GET` response instead.

## Settings

| Key | Type | Default | Meaning |
| --- | --- | --- | --- |
| `hr_executive_role_ids` | array of role ids | `[]` | Holders of these roles are executives: they may be anyone's Immediate Manager, whatever the department, and need no manager themselves. |
| `hr_manager_role_id` | role id or `null` | `null` | The HR manager role: leave requests go to holders of this role after the Immediate Manager approves them. One role only. Until it is set, leave cannot complete its approval. |
| `hr_staff_no_prefix` | string | `EMP-` | Put before the running number of new staff numbers (`NWR-` gives `NWR-001`). Imported staff numbers count towards the next one only when they use this prefix. |
| `hr_work_start_time` | `HH:MM` (24-hour) | `08:00` | When the working day starts. |
| `hr_work_end_time` | `HH:MM` (24-hour) | `17:00` | When the working day ends. Must be after the start. |
| `hr_late_grace_minutes` | integer | `15` | Minutes after the start time that still count as on time. |
| `hr_working_days` | array of day names | Monday to Friday | The days the company works. Attendance, leave and days suspended count only these days. |
| `hr_required_document_types` | array of document types | `employment_contract`, `national_id`, `kra_pin_certificate` | The documents every employee's file must hold. Drives Missing Documents on the Employees list and profile completeness. `[]` requires none. |

## Read

`GET /api/v1/app/{company}/human-resource/admin/settings`

```json
{
  "data": {
    "settings": {
      "hr_executive_role_ids": [12],
      "hr_manager_role_id": 18,
      "hr_staff_no_prefix": "NWR-",
      "hr_work_start_time": "08:00",
      "hr_work_end_time": "17:00",
      "hr_late_grace_minutes": 15,
      "hr_working_days": ["monday", "tuesday", "wednesday", "thursday", "friday"],
      "hr_required_document_types": ["employment_contract", "national_id", "kra_pin_certificate"]
    },
    "executive_roles": [
      { "id": 12, "name": "Managing Director" }
    ],
    "hr_manager_role": { "id": 18, "name": "HR Manager" },
    "fields": [
      {
        "key": "hr_executive_role_ids",
        "name": "Executive Roles",
        "description": "Holders of these roles count as executives: ...",
        "type": "multiselect",
        "options": ["https://api.example.com/api/v1/app/1/access-management/roles?filter[user_group_id]=3"]
      },
      {
        "key": "hr_manager_role_id",
        "name": "HR Manager Role",
        "description": "Leave requests go to holders of this role after the Immediate Manager approves them. ...",
        "type": "select",
        "options": ["https://api.example.com/api/v1/app/1/access-management/roles?filter[user_group_id]=3"]
      },
      {
        "key": "hr_working_days",
        "name": "Working Days",
        "description": "The days of the week the company works. ...",
        "type": "multiselect",
        "options": { "monday": "Monday", "tuesday": "Tuesday", "wednesday": "Wednesday", "thursday": "Thursday", "friday": "Friday", "saturday": "Saturday", "sunday": "Sunday" }
      }
    ],
    "can_update": true
  }
}
```

- `settings`: the values, typed. Lists are arrays, the grace period is a number.
- `executive_roles`: id and name of each executive role, to show the chosen chips without a
  second call.
- `hr_manager_role`: id and name of the HR manager role, or `null` while none is set (always
  present).
- `fields`: label, help text, input `type` (`multiselect`, `select`, `text`, `time`, `number`) and `options`
  for each key, in display order. For the executive roles and the HR manager role, `options` holds
  the URL that lists the company's staff roles; for working days it is the value → label map.
- `can_update`: whether the caller may save.

## Update

`PUT /api/v1/app/{company}/human-resource/admin/settings`

Send only the keys that change. A key left out keeps its value; any key that is not an HR setting
is ignored.

```json
{
  "hr_executive_role_ids": [12, 15],
  "hr_manager_role_id": 18,
  "hr_staff_no_prefix": "NWR-",
  "hr_work_start_time": "08:00",
  "hr_work_end_time": "17:00",
  "hr_late_grace_minutes": 15,
  "hr_working_days": ["monday", "tuesday", "wednesday", "thursday", "friday"], // any of monday … sunday
  "hr_required_document_types": ["employment_contract", "national_id"] // employment_contract, national_id, kra_pin_certificate, academic_certificate, nssf_card, other
}
```

Answers `200` with `message` plus the same `settings`, `executive_roles`, `hr_manager_role`,
`fields` and `can_update` as the `GET`.

Rules (`422` with field errors otherwise):

- `hr_executive_role_ids`: an array (`[]` clears it). Each id must be a staff role of this company
  (`Each executive role must be a staff role of this company.` on `hr_executive_role_ids.N`).
- `hr_manager_role_id`: one staff role of this company, or `null` to clear it
  (`The HR manager role must be a staff role of this company.`).
- `hr_staff_no_prefix`: 1 to 20 characters, letters, numbers, `-`, `/` and `_` only.
- `hr_work_start_time`, `hr_work_end_time`: `HH:MM`. The end must be after the start, comparing
  the values sent with the stored ones for a key not sent
  (`The work end time must be after the work start time.`).
- `hr_late_grace_minutes`: 0 to 240.
- `hr_working_days`: at least one day, no repeats, each one of `monday` … `sunday`. Saved Monday
  first whatever order they are sent in.
- `hr_required_document_types`: an array (`[]` requires none), no repeats, each a document type.
  Saved in the document types' order.

`403` without `update-human-resource-setting`, checked before the body is validated.
