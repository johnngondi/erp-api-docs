# Projects API

Domain: `Project Management`

Base prefix:

`/api/v1/app/{company}/project-management/projects`

## Endpoints

Projects:

- `GET /projects`
- `POST /projects`
- `GET /projects/{project}`
- `PUT/PATCH /projects/{project}`
- `DELETE /projects/{project}`
- `PATCH /projects/{project}/cancel`

Milestones:

- `GET /projects/{project}/milestones`
- `POST /projects/{project}/milestones`
- `GET /projects/{project}/milestones/{milestone}`
- `PUT/PATCH /projects/{project}/milestones/{milestone}`
- `DELETE /projects/{project}/milestones/{milestone}`
- `PUT /projects/{project}/milestones/{milestone}/update-status`

Tasks:

- `GET /projects/{project}/milestones/{milestone}/tasks`
- `POST /projects/{project}/milestones/{milestone}/tasks`
- `GET /projects/{project}/milestones/{milestone}/tasks/{task}`
- `PUT/PATCH /projects/{project}/milestones/{milestone}/tasks/{task}`
- `DELETE /projects/{project}/milestones/{milestone}/tasks/{task}`
- `PUT /projects/{project}/milestones/{milestone}/tasks/{task}/update-status`

Updates:

- `GET /projects/{project}/updates`
- `POST /projects/{project}/updates`
- `GET /projects/{project}/updates/{update}`
- `PUT/PATCH /projects/{project}/updates/{update}`
- `DELETE /projects/{project}/updates/{update}`

## Project Status Enums

Allowed `status` input values for project create/update:

- `pending` (color: `warning`)
- `in-progress` (color: `primary`)
- `completed` (color: `success`)
- `cancelled` (color: `secondary`)
- `stalled` (color: `danger`)

## List Projects

`GET /api/v1/app/{company}/project-management/projects`

Supported query params:

- Filters:
  - `filter[id]`
  - `filter[title]`
  - `filter[status]`
  - `filter[facility_id]`
  - `filter[total_budget]`
  - `filter[created_at]`
  - `filter[initiated_at]`
  - `filter[estimated_start_date]`
  - `filter[completed_at]`
  - `filter[estimated_end_date]`
- Sort:
  - `sort=id,title,total_budget,created_at,initiated_at,estimated_start_date,completed_at,estimated_end_date`
- Include:
  - Not supported on this endpoint.
- Select fields:
  - Not supported on this endpoint.
- Pagination:
  - `per_page`, `page`

Examples:

- `GET /api/v1/app/12/project-management/projects?filter[status]=in-progress&sort=-created_at`
- `GET /api/v1/app/12/project-management/projects?filter[facility_id]=14&per_page=20&page=2`

## Create/Update Project

`POST /api/v1/app/{company}/project-management/projects`  
`PUT/PATCH /api/v1/app/{company}/project-management/projects/{project}`

Request body:

| Field | Required | Type | Allowed Values / Notes |
|---|---|---|---|
| `facility_id` | Yes | integer | Must exist in `facilities.id` |
| `title` | Yes | string | - |
| `description` | Yes | string | - |
| `estimated_start_date` | Yes | date (`YYYY-MM-DD`) | - |
| `estimated_duration_in_days` | Yes | number | Numeric |
| `currency_id` | No | integer | Must exist in `currencies.id` |
| `total_budget` | No | number | Default `0` |
| `project_manager_id` | No | integer | Must exist in `users.id` |
| `initiated_at` | No | date (`YYYY-MM-DD`) | - |
| `completed_at` | No | date (`YYYY-MM-DD`) | - |
| `status` | No | string | `pending`, `in-progress`, `completed`, `stalled`, `cancelled` |

Notes:

- `estimated_end_date` is computed by backend from `estimated_start_date + estimated_duration_in_days`.
- Do not send `estimated_end_date` in request payload.

Example create payload:

```json
{
  "facility_id": 14,
  "title": "Tower A Elevator Upgrade",
  "description": "Replace legacy elevator control boards and cabin panels.",
  "estimated_start_date": "2026-03-01",
  "estimated_duration_in_days": 45,
  "currency_id": 1,
  "total_budget": 2500000,
  "project_manager_id": 57,
  "status": "pending"
}
```

Example response:

```json
{
  "message": "Project created successfully.",
  "project": {
    "id": 1201,
    "title": "Tower A Elevator Upgrade",
    "status": {
      "value": "pending",
      "color": "warning"
    },
    "estimated_start_date": "2026-03-01",
    "estimated_end_date": "2026-04-15 00:00:00",
    "total_budget": 2500000
  }
}
```

## Cancel Project

`PATCH /api/v1/app/{company}/project-management/projects/{project}/cancel`

Request body:

- none

Behavior notes:

- Backend returns `422` when project is already `completed` or `cancelled`.
- Backend may also return `403` if user lacks permission.

## Milestones

### List Milestones

`GET /api/v1/app/{company}/project-management/projects/{project}/milestones`

Supported query params:

- Filters:
  - `filter[title]`
  - `filter[status]`
- Sort:
  - `sort=title,status,estimated_start_date,started_at,completed_at,created_at`
- Include:
  - `include=projectTasks`
- Select fields:
  - Not supported
- Pagination:
  - `per_page`, `page`

Milestone status values:

- `pending` (color: `secondary`)
- `in-progress` (color: `primary`)
- `late` (color: `warning`)
- `stuck` (color: `danger`)
- `completed` (color: `success`)

### Create/Update Milestone

`POST /api/v1/app/{company}/project-management/projects/{project}/milestones`  
`PUT/PATCH /api/v1/app/{company}/project-management/projects/{project}/milestones/{milestone}`

| Field | Required | Type | Allowed Values / Notes |
|---|---|---|---|
| `title` | Yes | string | - |
| `description` | No | string | Optional |
| `percentage_of_budget` | Yes | number | Numeric |
| `estimated_start_date` | Yes | date (`YYYY-MM-DD`) | - |
| `estimated_duration_in_days` | Yes | number | Numeric |
| `responsible_person_id` | No | integer | Must exist in `users.id` |
| `started_at` | No | date (`YYYY-MM-DD`) | - |
| `completed_at` | No | date (`YYYY-MM-DD`) | - |

Notes:

- `estimated_end_date` is computed by backend.

### Update Milestone Status

`PUT /api/v1/app/{company}/project-management/projects/{project}/milestones/{milestone}/update-status`

| Field | Required | Type | Allowed Values |
|---|---|---|---|
| `status` | Yes | string | `pending`, `in-progress`, `late`, `stuck`, `completed` |

## Tasks

### List Tasks

`GET /api/v1/app/{company}/project-management/projects/{project}/milestones/{milestone}/tasks`

Supported query params:

- Filters:
  - `filter[title]`
  - `filter[status]`
  - `filter[priority]`
  - `filter[assigned_to_id]`
  - `filter[estimated_duration_in_days]`
- Sort:
  - `sort=title,status,estimated_start_date,started_at,completed_at,created_at`
- Include:
  - Not supported
- Select fields:
  - Not supported
- Pagination:
  - `per_page`, `page`

Task status values:

- `pending` (color: `secondary`)
- `in-progress` (color: `primary`)
- `late` (color: `warning`)
- `stuck` (color: `danger`)
- `done` (color: `success`)

Task priority values:

- `low` (color: `success`)
- `normal` (color: `primary`)
- `high` (color: `danger`)

### Create/Update Task

`POST /api/v1/app/{company}/project-management/projects/{project}/milestones/{milestone}/tasks`  
`PUT/PATCH /api/v1/app/{company}/project-management/projects/{project}/milestones/{milestone}/tasks/{task}`

| Field | Required | Type | Allowed Values / Notes |
|---|---|---|---|
| `title` | Yes | string | - |
| `estimated_start_date` | Yes | date (`YYYY-MM-DD`) | - |
| `estimated_duration_in_days` | Yes | number | Numeric |
| `description` | No | string | Optional |
| `priority` | No | string | `low`, `normal`, `high` |
| `assigned_to_id` | No | integer | Must exist in `users.id` |

Notes:

- `estimated_end_date` is computed by backend.

### Update Task Status

`PUT /api/v1/app/{company}/project-management/projects/{project}/milestones/{milestone}/tasks/{task}/update-status`

| Field | Required | Type | Allowed Values |
|---|---|---|---|
| `status` | Yes | string | `pending`, `in-progress`, `late`, `stuck`, `done` |

Implementation note:

- Backend currently does not explicitly validate this field in controller.
- Frontend should always submit only allowed values above to avoid unexpected server errors.

## Project Updates

### List Updates

`GET /api/v1/app/{company}/project-management/projects/{project}/updates`

Supported query params:

- Filters:
  - `filter[title]`
  - `filter[description]`
- Sort:
  - `sort=created_at,updated_at`
- Include:
  - Not supported
- Select fields:
  - Not supported
- Pagination:
  - `per_page`, `page`

### Create/Update Update Entry

`POST /api/v1/app/{company}/project-management/projects/{project}/updates`  
`PUT/PATCH /api/v1/app/{company}/project-management/projects/{project}/updates/{update}`

| Field | Required | Type | Allowed Values / Notes |
|---|---|---|---|
| `description` | Yes | string | - |
| `title` | No | string | Optional |
| `attachment_upload_id` | No | integer | Must exist in `uploads.id` |
| `added_by` | No | string | `user`, `system` |
| `added_by_id` | No | integer | Must exist in `users.id` |
| `project_item_id` | No | integer | Optional |
| `project_item_type` | No | string | Optional |

Example create payload:

```json
{
  "title": "Site Inspection Complete",
  "description": "Electrical and safety inspections completed. No major blockers found.",
  "attachment_upload_id": 9021,
  "added_by": "user",
  "added_by_id": 57
}
```

## Frontend Error Handling

Use shared behavior in `docs/frontend/app/README.md`:

- Show backend `4xx` messages and validation errors to users.
- Show generic fallback for `5xx` responses.
