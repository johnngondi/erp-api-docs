# My Profile (Frontend, staff space)

The employee's own record in the staff space. **No permission**: any employee on probation or
active may use it (see `README.md`). There is no id in the URL; it is always the caller's own
record, or the employee named in `X-Acting-Employee` when HR acts for them.

| Endpoint | |
| --- | --- |
| `GET /api/v1/app/{company}/human-resource/me/profile` | Read |
| `PATCH /api/v1/app/{company}/human-resource/me/profile` | Update the five fields the employee owns |

## Read

`GET /api/v1/app/{company}/human-resource/me/profile`

The same `employee` object as the Admin profile (`employees.md` › Profile), without `access` and
without `permissions`: step 1 fields, `manager`, `direct_reports_count`, `documents`,
`profile_completeness`, and `leave_balance` / `attendance` (`null` until the Leave and Attendance
tickets).

```json
{
  "data": {
    "employee": {
      "id": 75,
      "staff_no": "NWR-075",
      "name": "Amina Hassan",
      "job_title": "Property Officer",
      "personal_email": "amina@example.net",
      "residence": "Kileleshwa",
      "emergency_contact_name": "Mary Atieno",
      "emergency_contact_phone": "0722111000",
      "photo_upload_id": 3402,
      "manager": { "id": 88, "staff_no": "NWR-088", "name": "Mercy Chebet", "job_title": "Property Manager" },
      "profile_completeness": { "percent": 86, "missing": [], "documents": { "required": 3, "uploaded": 3 } },
      "leave_balance": null,
      "attendance": null
    }
  }
}
```

(Abridged.)

## Update

`PATCH /api/v1/app/{company}/human-resource/me/profile`

Send any of these five; the rest of the record belongs to HR.

```json
{
  "personal_email": "amina@example.net",
  "residence": "Kileleshwa",
  "emergency_contact_name": "Mary Atieno",
  "emergency_contact_phone": "0722111000",
  "photo_upload_id": 3402
}
```

| Field | Rules |
| --- | --- |
| `personal_email` | email, or `null` to clear |
| `residence` | text, up to 191 characters |
| `emergency_contact_name` | text, up to 191 characters |
| `emergency_contact_phone` | text, up to 20 characters |
| `photo_upload_id` | an upload id (upload the photo first) |

Any other key in the body is a `422` on that key (`Only HR can change this detail.`) and nothing
is saved. Answers `200` with `message` (`Profile updated successfully.`) and the updated
`employee`, shaped as the read.

While HR acts for the employee (`X-Acting-Employee`), the change is saved to that employee and the
audit trail records it as made by HR on their behalf.
