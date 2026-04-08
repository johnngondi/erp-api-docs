# Access Management API (Frontend)

Base prefix:

`/api/v1/app/{company}/access-management`

`{company}` is the current app/company context.

## Modules

- User Groups:
  - `docs/frontend/api/v1/app/access-management/user-groups.md`
- Permissions:
  - `docs/frontend/api/v1/app/access-management/permissions.md`
- Roles:
  - `docs/frontend/api/v1/app/access-management/roles.md`
- Companies:
  - `docs/frontend/api/v1/app/access-management/companies.md`
- Company Offices:
  - `docs/frontend/api/v1/app/access-management/company-offices.md`
- Company Departments:
  - `docs/frontend/api/v1/app/access-management/company-departments.md`
- Company Department Divisions:
  - `docs/frontend/api/v1/app/access-management/company-department-divisions.md`
- Company Users:
  - `docs/frontend/api/v1/app/access-management/company-users.md`

Apply shared auth, tenancy, query, and error handling guidance from:

- `docs/frontend/api/v1/app/README.md`

## Create Payload Tables

| Module | Create Endpoint | Payload Table Location |
| --- | --- | --- |
| User Groups | `POST /user-groups` | `user-groups.md` (currently N/A; endpoint not implemented) |
| Permissions | `POST /permissions` | `permissions.md` (N/A; no create endpoint) |
| Roles | `POST /roles` | `roles.md` |
| Companies | `POST /companies` | `companies.md` |
| Company Offices | `POST /company-offices` | `company-offices.md` |
| Company Departments | `POST /company-departments` | `company-departments.md` |
| Company Department Divisions | `POST /company-departments/{companyDepartment}/divisions` | `company-department-divisions.md` |
| Company Users | `POST /company-users` | `company-users.md` |
