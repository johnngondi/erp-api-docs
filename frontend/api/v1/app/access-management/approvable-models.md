# Access Management: Approvable Models

Base prefix:

`/api/v1/app/{company}/access-management`

## Endpoint

Implemented:

- `GET /approvable-models`

## Purpose

`GET /approvable-models` returns all configured approvable resources from `config/approvals.php`.

Use this response to populate:

- model dropdown in template form
- status dropdowns (`initial_status`, `final_status`)

## Response Shape

Each item contains:

- `class` (FQCN)
- `label` (human-readable model label)
- `statuses` (allowed status values)

Example:

```json
{
  "data": {
    "models": [
      {
        "class": "App\\Models\\FacilityInvoice",
        "label": "Facility Invoice",
        "statuses": ["pending", "unpaid", "partially paid", "paid", "cancelled"]
      }
    ]
  }
}
```

