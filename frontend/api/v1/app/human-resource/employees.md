# Employees (Frontend)

HR creates and edits employees in the Admin space. Saving a completed employee also creates (or
links) their PropERP account, company membership, roles and modules.

Base: `/api/v1/app/{company}/human-resource/admin/employees`

| Endpoint | Permission |
| --- | --- |
| `GET /employees` | `view-employee` |
| `GET /employees/summary` | `view-employee` |
| `GET /employees/{employee}` | `view-employee` |
| `POST /employees` | `create-employee` |
| `PATCH /employees/{employee}` | `update-employee` |
| `DELETE /employees/{employee}` | `delete-employee` (drafts only) |
| `GET /employees/manager-options` | `view-employee`, `create-employee` or `update-employee` |
| `POST /employees/{employee}/documents` | `update-employee` |
| `DELETE /employees/{employee}/documents/{document}` | `update-employee` |

The employee's own profile in the staff space is in `profile.md`.

## List

`GET /api/v1/app/{company}/human-resource/admin/employees`

Paginated (`per_page`, `page`), newest first. Every status is listed, drafts and terminated
included; use the filters for a tab or a status.

| Filter | Match | Notes |
| --- | --- | --- |
| `filter[search]` | text | Staff number, names (first, middle, last, or full name), job title, work email. A comma list of numbers is a list of employee ids. |
| `filter[employment_type]` | exact | `permanent`, `contract`, `intern`, `casual`. Comma list allowed. Use it for the tabs. |
| `filter[status]` | exact | `draft`, `on_probation`, `active`, `suspended`, `terminated`. Comma list allowed. |
| `filter[company_department_id]` | exact | |
| `filter[company_office_id]` | exact | |
| `filter[manager_id]` | exact | Everyone who reports to that employee. |

Sorts: `sort=staff_no`, `first_name`, `date_joined`, `created_at` (prefix `-` for descending).
Default: newest first.

Includes: `include=companyDepartment,companyOffice,manager,user`. Department, office and manager
are always returned; `user` adds nothing the row needs, so leave it out unless you need it.

Each row carries the employee fields (see the profile below, without `access`, `documents` and
`profile_completeness`) and its own `permissions` (`view`, `update`, `delete`, `suspend`, `activate`, `terminate`).

```json
{
  "data": [
    {
      "id": 75,
      "staff_no": "NWR-075",
      "name": "Amina Hassan",
      "job_title": "Property Officer",
      "company_department": { "id": 6, "name": "Property Management" },
      "company_office": { "id": 3, "name": "Field / Sites" },
      "manager": { "id": 88, "staff_no": "NWR-088", "name": "Mercy Chebet", "job_title": "Property Manager" },
      "employment_type": { "value": "contract", "color": "primary", "label": "Contract" },
      "date_joined": "2023-08-01",
      "status": { "value": "active", "color": "success", "label": "Active" },
      "permissions": { "view": true, "update": true, "delete": false, "suspend": true, "activate": false, "terminate": true }
    }
  ],
  "links": { "...": "..." },
  "meta": { "current_page": 1, "per_page": 15, "total": 130 }
}
```

(Rows abridged.) "On leave" on the design's status column comes with the Leave tickets. Export is
done in the browser from this list; there is no export endpoint.

## Summary

`GET /api/v1/app/{company}/human-resource/admin/employees/summary`

The tiles and tab counts above the list.

```json
{
  "data": {
    "employees": {
      "total": 130,
      "change_this_month": { "count": 6, "percent": 4.8, "previous_total": 124 }
    },
    "on_leave_today": null,
    "on_probation": { "count": 6, "reviews_due_this_month": 2 },
    "contracts_ending": { "count": 4, "within_days": 60 },
    "missing_documents": { "count": 9, "required_types": ["employment_contract", "national_id", "kra_pin_certificate"] },
    "tabs": { "all": 130, "permanent": 96, "contract": 28, "intern": 6, "casual": 0 }
  }
}
```

- `employees`: headcount today and its change since the end of last month, the same figures as
  the HR dashboard.
- `on_leave_today`: `null` until the Leave tickets.
- `on_probation.reviews_due_this_month`: probation ends this month.
- `contracts_ending`: contracts ending from today to 60 days ahead.
- `missing_documents`: employees whose file lacks at least one of the required documents (HR
  Settings › Required Documents, `settings.md`).
- `tabs`: employees per employment type; `all` is their sum.
- Every count except the headcount is of employees on probation, active or suspended: drafts and
  terminated employees are left out.
- So `tabs.all` can be higher than `employees.total`: someone hired with a join date still ahead
  is in the tabs (they are on the list) but not yet in the headcount.

## Profile

`GET /api/v1/app/{company}/human-resource/admin/employees/{employee}`

Every step 1 field, plus:

```json
{
  "data": {
    "employee": {
      "id": 75,
      "staff_no": "NWR-075",
      "name": "Amina Hassan",
      "...": "every step 1 field",
      "manager": { "id": 88, "staff_no": "NWR-088", "name": "Mercy Chebet", "job_title": "Property Manager" },
      "direct_reports_count": 0,
      "access": {
        "modules": ["property-management", "human-resource"],
        "roles": [{ "id": 12, "name": "Property Officer" }],
        "facilities": [{ "id": 3, "name": "Kimathi House" }]
      },
      "documents": [
        { "id": 501, "type": { "value": "national_id", "color": "info", "label": "National ID" }, "title": "ID copy", "upload": { "id": 3301, "source_url": "…" } }
      ],
      "profile_completeness": {
        "percent": 78,
        "missing": [
          { "key": "date_of_birth", "kind": "detail", "label": "Date of birth" },
          { "key": "employment_contract", "kind": "document", "label": "Employment Contract" }
        ],
        "documents": { "required": 3, "uploaded": 2 }
      },
      "leave_balance": null,
      "attendance": null,
      "permissions": { "view": true, "update": true, "delete": false, "suspend": true, "activate": false, "terminate": true }
    }
  }
}
```

- `profile_completeness` counts the personal and job details (photo, gender, date of birth,
  national ID number, KRA PIN, nationality, SHIF number, NSSF number, phone, residence, emergency
  contact name and phone, job title, department), the Immediate Manager (not for an executive)
  and the required documents. `missing[].kind` is `detail`, `manager` or `document`; `key` is the
  field name or the document type. Use `documents` for "2 of 3 uploaded".
- `leave_balance` and `attendance` are `null` until the Leave and Attendance tickets.
- While suspended: `suspended_at`, `suspension_reason`, `suspended_until` (the last day, or `null`
  for until further notice) and `days_suspended` (working days so far, today included; see the
  status history). All are `null` otherwise. Past suspensions are in the status history.
- Another company's employee answers `404`.
- No payroll, performance or pay figures are returned in this batch.

## The two steps

The form has two steps sent as one body: step 1 is the employee's details, step 2 is `access`.

### Draft or completed

| `status` | What is saved |
| --- | --- |
| `draft` | Step 1 only. Needs `first_name`, `last_name`, `work_email`, `date_joined`; everything else is optional. No account is created and `access` is ignored. The staff number is given at once. |
| `on_probation`, `active` | Both steps. Every required detail (below) and `access` the first time. Creates or links the account. |

A draft becomes an employee by `PATCH`ing its `status` to `on_probation` or `active` with the
missing details and `access`. A completed employee cannot go back to draft. `suspended` and
`terminated` are set by their own actions, never by this form.

### Step 1: details

Required for a completed employee: `first_name`, `last_name`, `gender`, `work_email`, `phone`,
`job_title`, `company_department_id`, `company_office_id`, `employment_type`, `date_joined`, and
`manager_id` (see below). The office is required because attendance check-in is per office.

`staff_no` is not a field: the API gives it (see below), so the form has no input for it.

| Field | Type | Notes |
| --- | --- | --- |
| `photo_upload_id` | integer | An upload id. |
| `first_name`, `middle_name`, `last_name` | string | Up to 100 characters each. |
| `gender` | string | `male`, `female`, `other` |
| `date_of_birth` | date `Y-m-d` | Before today. |
| `marital_status` | string | `single`, `married`, `divorced`, `widowed`, `other` |
| `national_id_no`, `kra_pin`, `nationality` | string | |
| `shif_number`, `nssf_number` | string | Member numbers only. |
| `work_email` | email | Becomes the account's login email. |
| `personal_email` | email | |
| `phone` | string | Becomes the account's phone. Any format: `0712345678`, `+254 712 345 678`. |
| `residence`, `emergency_contact_name`, `emergency_contact_phone` | string | |
| `job_title` | string | |
| `company_department_id` | integer | A department of this company. |
| `company_department_division_id` | integer | A division of that department. |
| `company_office_id` | integer | An office of this company. Required for a completed employee. |
| `manager_id` | integer | The Immediate Manager, an employee id. See below. |
| `job_group_id` | integer | Optional, never required. |
| `employment_type` | string | `permanent`, `contract`, `intern`, `casual` |
| `date_joined` | date | |
| `contract_ends_at`, `probation_ends_at` | date | Not before `date_joined`. |
| `status` | string | `draft`, `on_probation`, `active` |

**Payroll details are not part of this form** (payroll group, salary, allowances, bank,
contributions). Fields like those are ignored if sent.

### Immediate Manager

- Required, unless one of the employee's roles is an **executive role** (HR Settings ›
  General › Executive Roles). Executives report to no one.
- When no executive roles are set yet, saving without a manager answers
  `Choose the Immediate Manager. To save someone who reports to no one, set the executive roles in HR Settings first.`
  Set the CEO/executive role in HR Settings before adding the first employees.
- The manager must be an employee of this company who is on probation or active, not the
  employee themself, not someone who already reports to them, and either in the same department
  or holding an executive role.
- Changing the department re-checks the current manager.
- Fill the picker from `GET /employees/manager-options` (below): it applies the same rule.

### Step 2: access

```json
"access": {
  "modules": ["property-management"],
  "roles": [12, 15],
  "facilities": [3, 9],
  "send_welcome_email": true
}
```

| Key | Notes |
| --- | --- |
| `modules` | Any of `access-management`, `property-management`, `facility-management`, `project-management`, `human-resource`. **`human-resource` is always added**, whether sent or not. |
| `roles` | Staff role ids of this company. Replaces the person's roles in this company. May be empty: Human Resource needs no role. Every module chosen other than Human Resource needs at least one chosen role that gives access to it (`Choose at least one role that gives access to Property Management.` on `access.roles`). |
| `facilities` | Property ids of this company. Replaces property access. Leave the key out to keep property access as it is. |
| `send_welcome_email` | Default `true`. Sends the login details to a new account. |

On `PATCH`, send `access` only to change access; without it roles, modules and properties stay
as they are.

### The account behind the employee

- If an account already uses the `work_email`, that account is linked (an existing staff member
  becoming an employee). Its name and phone are updated to the employee's.
- `422` on `work_email` when that account is already an employee of this company.
- `422` on `phone` when the phone belongs to a different account.
- Later edits to the name, work email or phone update the account; another account's email or
  phone is refused the same way.

### Create

`POST /api/v1/app/{company}/human-resource/admin/employees`

```json
{
  "first_name": "Jane",
  "middle_name": "Wanjiku",
  "last_name": "Mwangi",
  "gender": "female", // male, female, other
  "date_of_birth": "1994-05-17",
  "marital_status": "single", // single, married, divorced, widowed, other
  "national_id_no": "30123456",
  "kra_pin": "A012345678Z",
  "nationality": "Kenyan",
  "shif_number": "SHIF-0099887",
  "nssf_number": "NSSF-552211",
  "work_email": "jane.mwangi@example.com",
  "personal_email": "jane@example.net",
  "phone": "0712345678",
  "residence": "Kilimani, Nairobi",
  "emergency_contact_name": "Peter Mwangi",
  "emergency_contact_phone": "0722000111",
  "job_title": "Accountant",
  "company_department_id": 4,
  "company_department_division_id": null,
  "company_office_id": 2,
  "manager_id": 18,
  "job_group_id": null,
  "employment_type": "permanent", // permanent, contract, intern, casual
  "date_joined": "2026-12-01",
  "contract_ends_at": null,
  "probation_ends_at": "2027-03-01",
  "photo_upload_id": null,
  "status": "on_probation", // draft, on_probation, active
  "access": {
    "modules": ["property-management"], // access-management, property-management, facility-management, project-management, human-resource
    "roles": [12],
    "facilities": [3, 9],
    "send_welcome_email": true
  }
}
```

Response `200`:

```json
{
  "data": {
    "message": "Employee created successfully.",
    "employee": {
      "id": 41,
      "staff_no": "NWR-041",
      "name": "Jane Wanjiku Mwangi",
      "first_name": "Jane",
      "status": { "value": "on_probation", "color": "info", "label": "On Probation" },
      "employment_type": { "value": "permanent", "color": "success", "label": "Permanent" },
      "manager": { "id": 18, "staff_no": "NWR-018", "name": "Peter Otieno", "job_title": "Finance Manager" },
      "company_department": { "id": 4, "name": "Finance" },
      "user_id": 233,
      "access": {
        "modules": ["property-management", "human-resource"],
        "roles": [{ "id": 12, "name": "Accountant" }],
        "facilities": [{ "id": 3, "name": "Kimathi House" }, { "id": 9, "name": "Westlands Plaza" }]
      },
      "documents": [],
      "permissions": { "view": true, "update": true, "delete": false, "suspend": true, "activate": false, "terminate": true }
    }
  }
}
```

(Abridged: every step 1 field is returned as well.) For a draft, `user_id` and `access` are
`null`.

The staff number is given as soon as the employee is saved, a draft included, and shown on the
profile. It is the HR Settings prefix plus one more than the highest number already given under
that prefix, at least three digits: `NWR-001`, and after an import ending at `NWR-128` the next is
`NWR-129`. It counts from the staff numbers, never from the employee id. A draft saved before
drafts were numbered gets its number on its next save.

### Update

`PATCH /api/v1/app/{company}/human-resource/admin/employees/{employee}`

Send only what changes. Same rules as create; for keys not sent the stored values count (so
completing a draft only needs what the draft lacks). Answers like create, with
`Employee updated successfully.`

### Delete

`DELETE /api/v1/app/{company}/human-resource/admin/employees/{employee}`

Drafts only. A completed employee answers `422` on `employee`:
`Only a draft employee can be deleted. Terminate a completed employee instead.`
`permissions.delete` on the employee is `true` only for a draft the caller may delete.

## Suspend, reactivate, terminate

Three actions on a completed employee. Each also changes the person's company membership, so
their PropERP access follows on their next request. The staff space (`me/*`) is closed to a
suspended or terminated employee.

Show each button from the employee's own `permissions`: `suspend`, `activate` and `terminate` are
`true` only when the caller holds the permission and the action would succeed for this employee
(its status, not the caller's own record, not the company owner). `GET /me/permissions` does not
list these three names.

### Suspend

`PATCH /api/v1/app/{company}/human-resource/admin/employees/{employee}/suspend`

Permission `suspend-employee`.

```json
{ "reason": "Disciplinary", "suspended_until": "2026-12-05" }
```

| Field | Rules |
| --- | --- |
| `reason` | Required, up to 1000 characters |
| `suspended_until` | Optional, `Y-m-d`, today or later. The **last day** of the suspension. Left out or `null`: until further notice. |

- Allowed from on probation or active. The employee becomes `suspended`, with `suspended_at`,
  `suspension_reason` and `suspended_until`; their membership becomes `suspended`, so PropERP
  refuses them (`403`) from their next request.
- **With `suspended_until`**, the system reactivates them the day after it (checked hourly, in the
  company's timezone), exactly as the Reactivate button would. HR can still reactivate earlier.
  Label the field as the last day, e.g. "Suspended until (last day)": `2026-12-05` means they are
  back on 6 Dec.
- **Without it**, they stay suspended until someone reactivates them.
- The end date can be changed while the suspension runs (below).

### Change the end of a suspension

`PATCH /api/v1/app/{company}/human-resource/admin/employees/{employee}/suspension`

Permission `suspend-employee`. Show the button from `permissions.editSuspension`.

```json
{ "suspended_until": "2026-12-12" }
```

| Field | Rules |
| --- | --- |
| `suspended_until` | Must be sent. `Y-m-d`, today or later: the new last day. `null`: until further notice. |

- Only while the employee is suspended. Moves the last day, gives an open-ended suspension one, or
  makes it until further notice. The reason and start stay as they were.
- The status history gets no new row: the running suspension's row shows the new date.
- The Immediate Manager is told in-app (`change`: `suspension_changed`) when the date actually
  changes.
- Answers `200` with `Suspension updated successfully.` and the employee, like the actions below.
- `403` without the permission, or on one's own record (`You cannot change your own suspension.`);
  `422` on `status` when not suspended
  (`Only a suspension that is running can be changed. This employee is Active.`), on
  `suspended_until` when missing or before today.

### Reactivate

`PATCH /api/v1/app/{company}/human-resource/admin/employees/{employee}/activate`

Permission `activate-employee`. No body.

- Allowed from suspended only. A terminated employee is refused: termination is final.
- The employee returns to the status they had when suspended (on probation or active), read from
  the status history. With none on record: on probation while `probation_ends_at` is still ahead,
  active otherwise.
- The suspension fields are cleared and the membership becomes `active` again. How long the
  suspension lasted stays in the status history.

### Terminate

`PATCH /api/v1/app/{company}/human-resource/admin/employees/{employee}/terminate`

Permission `terminate-employee`.

```json
{
  "reason": "Resigned",
  "last_working_date": "2026-12-31",
  "reassign_reports_to_id": 45
}
```

| Field | Rules |
| --- | --- |
| `reason` | Required, up to 1000 characters |
| `last_working_date` | Required, `Y-m-d`. Not before the employee joined. May be in the past (a termination recorded late). |
| `reassign_reports_to_id` | Required only when someone still reports to this employee: an employee id |

- Allowed from on probation, active or suspended. The employee becomes `terminated`, with
  `terminated_at`, `termination_reason` and `last_working_date`.
- **Access ends at once**, whatever the last working date: the membership becomes `inactive`. Until
  the last working date the dashboard still counts them, as `leaving`.
- Access Management may later turn their login back on; that never brings the employee back.
- **Direct reports.** Everyone who still reports to this employee (on probation, active, suspended
  and drafts; not terminated) must move. Without `reassign_reports_to_id` the answer is `422`:
  `3 employees report to Mercy Chebet. Choose who they report to now.` With it, the replacement must
  pass the Immediate Manager rule for **each** report (same department or executive, on probation
  or active, no reporting cycle); otherwise `422` lists one message per report that fails, as
  `Amina Hassan: The Immediate Manager must be in the same department or hold an executive role.`,
  and nothing is changed. A replacement who is one of the reports fails for their own record
  (`An employee cannot be their own Immediate Manager.`): give them a manager in step 1 first, or
  pick someone else. Use `GET employees?filter[manager_id]={employee}` to show who will move, and
  `manager-options` with the reports' department to offer replacements.

### Answers

All three answer like this, with `Employee suspended successfully.`,
`Employee reactivated successfully.` or `Employee terminated successfully.`:

```json
{
  "data": {
    "message": "Employee suspended successfully.",
    "employee": {
      "id": 75,
      "status": { "value": "suspended", "color": "warning", "label": "Suspended" },
      "suspended_at": "2026-11-26T09:14:02+03:00",
      "suspension_reason": "Disciplinary",
      "suspended_until": "2026-12-05",
      "days_suspended": 1,
      "terminated_at": null,
      "termination_reason": null,
      "last_working_date": null,
      "...": "every step 1 field"
    }
  }
}
```

| Status | Key | Message |
| --- | --- | --- |
| `403` | | Without the action's permission (`This action is unauthorized.`) |
| `403` | | `You cannot suspend your own employee record.` / `You cannot terminate your own employee record.` |
| `404` | | Another company's employee |
| `422` | `suspended_until` | `The last day of the suspension cannot be before today.` |
| `422` | `status` | `Only an employee on probation or active can be suspended. This employee is Draft.` |
| `422` | `status` | `Only a suspended employee can be reactivated. This employee is Active.` |
| `422` | `status` | `A terminated employee cannot be reactivated.` |
| `422` | `status` | `A draft employee cannot be terminated. Delete the draft instead.` / `This employee is already terminated.` |
| `422` | `employee` | `The company owner cannot be suspended: their access to the company does not depend on a membership.` (same for terminated) |
| `422` | `last_working_date` | `The last working date cannot be before the employee joined (02 Mar 2026).` |
| `422` | `reassign_reports_to_id` | See Direct reports above |

The employee's Immediate Manager gets an in-app notification for each action (no email or SMS),
with `employee_id`, `employee_name`, `change` (`suspended`, `suspension_changed`, `reactivated`,
`terminated`), `reason`,
`suspended_until` and `last_working_date`, under the Human Resource domain. A reactivation by the
system at the end of a suspension sends the same `reactivated` notification.

### Status history

`GET /api/v1/app/{company}/human-resource/admin/employees/{employee}/status-history`

Permission `view-employee`. Every status the employee moved to, latest first, not paginated:
created (as a draft, on probation or active), completed, confirmed, suspended, reactivated,
terminated. Employees created before the history existed start from their next change.

```json
{
  "data": [
    {
      "id": 31,
      "from_status": { "value": "suspended", "color": "warning", "label": "Suspended" },
      "to_status": { "value": "active", "color": "success", "label": "Active" },
      "reason": null,
      "suspended_until": null,
      "last_working_date": null,
      "changed_at": "2026-12-06T00:15:03+03:00",
      "changed_by": null,
      "suspension_ended_at": null,
      "days_suspended": null
    },
    {
      "id": 30,
      "from_status": { "value": "active", "color": "success", "label": "Active" },
      "to_status": { "value": "suspended", "color": "warning", "label": "Suspended" },
      "reason": "Disciplinary",
      "suspended_until": "2026-12-05",
      "last_working_date": null,
      "changed_at": "2026-12-01T09:14:02+03:00",
      "changed_by": { "id": 12, "name": "Grace Wanjiru" },
      "suspension_ended_at": "2026-12-06T00:15:03+03:00",
      "days_suspended": 5
    }
  ]
}
```

- `from_status` is `null` on the row for the employee's creation.
- `changed_by` is `null` when the system made the change (the end of a suspension, records
  created by `hr:backfill-employees`). Show it as "System".
- `reason` is the suspension or termination reason; `suspended_until` is set on suspension rows,
  `last_working_date` on termination rows.
- On suspension rows only: `suspension_ended_at` is when it ended (reactivated or terminated),
  `null` while it is still running, and `days_suspended` is the company's **working days** (HR
  setting `hr_working_days`, Monday to Friday by default) from the day it started up to the day
  before the employee was back. Suspended Monday, back the next Monday: 5. Friday to Tuesday: 2.
  Over a weekend only: 0. While running, it counts today; lifted the day it started, it counts
  that day if it is a working day.
- Public holidays are not left out yet (there is no holiday list). The count follows the current
  working-days setting, so changing it recounts past suspensions too.
- Another company's employee answers `404`.

## Form options

`GET /api/v1/app/{company}/human-resource/admin/employees/form-options`

Everything the employee form lists, in one call. Open to anyone with `create-employee` or
`update-employee` (`403` otherwise), so the form does not need the Access Management or HR Settings
endpoints and their permissions.

```json
{
  "data": {
    "departments": [
      { "id": 1, "name": "Information Technology (IT)", "divisions": [
        { "id": 4, "name": "Networking and Infrastructure" },
        { "id": 3, "name": "Software Development" }
      ] }
    ],
    "offices": [{ "id": 2, "name": "Head Office" }],
    "roles": [
      { "id": 7, "name": "Property Manager", "modules": ["access-management", "property-management", "facility-management"] }
    ],
    "executive_role_ids": [25],
    "document_types": [
      { "value": "employment_contract", "color": "primary", "label": "Employment Contract", "required": true }
    ]
  }
}
```

- `departments` and their `divisions`, and `offices`: active ones only, by name.
- `roles`: the company's staff roles. `modules` lists the modules the role's permissions reach,
  so step 2 can show which roles cover each chosen module (every module except `human-resource`
  needs one).
- `executive_role_ids`: an employee holding one of these roles needs no Immediate Manager; for
  anyone else `manager_id` is required.
- `document_types`: every type, `required` when HR Settings › Required Documents lists it.

## Manager options

`GET /api/v1/app/{company}/human-resource/admin/employees/manager-options`

| Param | Notes |
| --- | --- |
| `company_department_id` | The department chosen in step 1. Without it only executives are offered. |
| `employee_id` | When editing: leaves out the employee and everyone under them. |
| `search` | Name, staff number or job title. |

```json
{
  "data": {
    "options": [
      {
        "id": 18,
        "staff_no": "NWR-018",
        "name": "Peter Otieno",
        "job_title": "Finance Manager",
        "company_department": { "id": 4, "name": "Finance" },
        "is_executive": false
      }
    ],
    "executives_configured": true
  }
}
```

Up to 50 options. `executives_configured: false` means HR Settings has no executive roles yet:
show a hint to set them, since nobody can be saved without a manager until then.

## Documents

`POST /api/v1/app/{company}/human-resource/admin/employees/{employee}/documents`

Upload the file first (the usual upload endpoint), then file it on the employee:

```json
{
  "upload_id": 3301,
  "type": "employment_contract", // employment_contract, national_id, kra_pin_certificate, academic_certificate, nssf_card, other
  "title": "Contract 2026"
}
```

Answers `200` with `message` and `document` (`id`, `type` as `{ value, color, label }`, `title`,
`upload`). No verification step yet.

`DELETE /api/v1/app/{company}/human-resource/admin/employees/{employee}/documents/{document}`
takes the document off the employee's file. The upload itself is kept.

## Module access

The modules a person may open are kept on their company membership. `GET /me/permissions`
returns them as `module_access` (see `docs/frontend/api/v1/app/inbox-and-permissions.md`). Human
Resource is always in it. Only the Human Resource routes check it so far.
