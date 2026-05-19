# Vendor Facilities API

Base route:

`/api/v1/vendor`

## Accessible Facilities

- `GET /accessible-facilities`

Behavior:

- Uses authenticated vendor context (`vendor()` helper).
- Reads `facility_expense_sub_type_vendors` by `vendor_id`.
- Flattens all values in `facilities` JSON field and removes duplicates.
- Returns only distinct facilities (`id`, `name`) sorted by `name`.

Success response (`DataResource`):

- `facilities` (array)
  - `id` (integer)
  - `name` (string)

Sample response:

```json
{
  "data": {
    "facilities": [
      {
        "id": 3,
        "name": "Garden Heights"
      },
      {
        "id": 9,
        "name": "Riverside Plaza"
      }
    ]
  }
}
```
