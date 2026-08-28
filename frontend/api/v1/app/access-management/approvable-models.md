# Access Management: Approvable Models

Base prefix:

`/api/v1/app/{company}/access-management`

## Endpoint

Implemented:

- `GET /approvable-models`

## Purpose

`GET /approvable-models` returns all configured approvable resources from `config/approvals.php`.

Use this response to populate:

- the model dropdown in the template form
- the field picker for a step's `editable_fields`

`statuses` is informational only (e.g. for display). Templates no longer accept `initial_status`/`final_status`; the workflow statuses are fixed per model via the `INITIAL_STATUS_ON_CREATE`, `FINAL_STATUS_ON_APPROVAL`, and `FINAL_STATUS_ON_REJECTION` constants.

## Response Shape

Each item contains:

- `class` (FQCN)
- `label` (human-readable model label)
- `statuses` (allowed status values)
- `fields` (the editable surface — what a step's `editable_fields` may name)

Each entry in `fields` carries:

- `name` — the field as it is submitted and as it must appear in `editable_fields`
- `label` — a human-readable rendering of the name, for the picker
- `type` — the underlying column type, or `unknown` for a field that is not a
  column (a model may expose an accessor its update endpoint accepts)

The list is the model's own answer, not the raw table. Most models derive it from
their columns minus the ones nobody edits through an approval step — `id`, the
timestamps, `company_id`, `created_by_id`, and `status`, which is the chain's own
to move — while a model with a curated surface states it directly.

`editable_fields` is validated against this list, so a step cannot grant editing
over something the resource does not expose. `"*"` is always accepted and means
every field.

Example:

```json
{
  "data": {
    "models": [
      {
        "class": "App\\Models\\FacilityInvoice",
        "label": "Facility Invoice",
        "statuses": ["pending", "unpaid", "partially paid", "paid", "cancelled"],
        "fields": [
          { "name": "lease_id", "label": "Lease", "type": "bigint" },
          { "name": "due_at", "label": "Due At", "type": "timestamp" },
          { "name": "notes", "label": "Notes", "type": "varchar" },
          { "name": "total", "label": "Total", "type": "decimal" }
        ]
      }
    ]
  }
}
```

