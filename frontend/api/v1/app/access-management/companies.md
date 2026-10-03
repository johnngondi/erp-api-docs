# Access Management: Companies

Base prefix:

`/api/v1/app/{company}/access-management`

## Endpoint Shape

Implemented:

- `GET /companies`
- `POST /companies`
- `GET /companies/{managedCompany}`
- `PUT/PATCH /companies/{managedCompany}`
- `DELETE /companies/{managedCompany}`

`{company}` is request context.
`{managedCompany}` is the target company for show/update/delete.

## List Companies

`GET /api/v1/app/{company}/access-management/companies`

Returns companies accessible to current user (owner or active member).

Supported query params:

- Filters:
  - `filter[search]` (Scout search; supports CSV company ids e.g. `3,8,12`)
  - `filter[status]`
  - `filter[created_at]` (date: `YYYY-MM-DD`)
- Include:
  - `include=companyUsers`
- Sort:
  - `sort=name`, `sort=status`, `sort=created_at`
- Pagination:
  - `per_page`, `page`

Selectable status values for dropdown:

- `active`
- `inactive`

## Create Company

`POST /api/v1/app/{company}/access-management/companies`

Create payload table:

| Field | Type | Required | Default | Notes |
| --- | --- | --- | --- | --- |
| `name` | string | Yes | - | Company name |
| `tax_pin` | string&#124;null | No | `null` | Tax PIN |
| `withholds` | array[int]&#124;null | No | `null` | IDs from `withholding_taxes` |
| `has_vat` | boolean&#124;null | No | `false` | VAT enabled flag |
| `service_managed` | string&#124;null | No | `null` | `rent` or `sc` or `both` |
| `address` | string&#124;null | No | `null` | Address text |
| `postal_address` | string&#124;null | No | `null` | Postal address / P.O. Box code (e.g. `2211-00202`); printed in the document footer |
| `company_phone` | string&#124;null | No | `null` | Public contact phone |
| `company_email` | string&#124;null | No | `null` | Public contact email |
| `company_website` | string&#124;null | No | `null` | Website URL or domain |
| `company_tagline` | string&#124;null | No | `null` | Short tagline / slogan |
| `twitter_handle` | string&#124;null | No | `null` | Twitter handle without the `@`; printed in the document footer |
| `facebook_url` | string&#124;null | No | `null` | Facebook page URL; printed in the document footer |
| `services` | array[string]&#124;null | No | `null` | Services advertised in the document-footer strip (e.g. `["Valuation","Management"]`); the strip balances itself across however many there are |
| `brand_color` | string&#124;null | No | `null` | Hex colour (`#rgb` or `#rrggbb`) of the identity band (landlord / property / period panel) under a report PDF's header; falls back to `COMPANY_BRAND_COLOR` (`#005044`) |
| `text_color_on_brand_bg` | string&#124;null | No | `null` | Hex text colour printed on `brand_color`; falls back to `COMPANY_TEXT_COLOR_ON_BRAND_BG` (`#ffffff`) |
| `accent_color` | string&#124;null | No | `null` | Hex colour of the printout footer's services strip (documents and report PDF exports); falls back to `COMPANY_ACCENT_COLOR` (`#ed1c24`) |
| `text_color_on_accent_bg` | string&#124;null | No | `null` | Hex text colour printed on `accent_color`; falls back to `COMPANY_TEXT_COLOR_ON_ACCENT_BG` (`#ffffff`) |
| `profile_photo_path` | string&#124;int&#124;null | No | `null` | Company logo: the upload `id` from the uploads endpoint, or a storage path. Stored as a string |
| `light_logo_path` | string&#124;int&#124;null | No | `null` | Light logo, drawn on `brand_color` in the header of every email: the upload `id` from the uploads endpoint, or a storage path. Stored as a string. Use a PNG or JPG; mail clients do not show SVG |
| `type_of_properties` | array&#124;null | No | `null` | Optional array |
| `collection_contract` | boolean | No | `false` | Collection contract enabled flag |
| `registration_type` | string&#124;null | No | `null` | `national_id` or `business_license` or `passport` |
| `registration_number` | string&#124;null | No | `null` | Registration number |
| `registration_upload_id` | int&#124;null | No | `null` | Upload ID |
| `tax_pin_cert_upload_id` | int&#124;null | No | `null` | Upload ID |
| `other_docs_upload_id` | int&#124;null | No | `null` | Upload ID |
| `status` | string | No | `active` | `active` or `inactive` |

Compatibility alias:

- `witholds` is accepted and normalized to `withholds`.

Validation note:

- This endpoint builds its payload with `CreateCompanyData::from(...)`, which does not run the declared rules. Values are stored as sent, so the frontend must validate `company_email` / `company_website` formats and field lengths (all string columns are `varchar(255)`).

Logo handling:

- Upload the file through `POST /api/v1/settings/file-management/uploads`, then send the upload's `id` as `profile_photo_path` (a raw storage path is still accepted). An integer id is stored as its string form, e.g. `13016` → `"13016"`.
- Send `null` to clear the logo.
- Responses return both `profile_photo_path` and `profile_photo_url`. For an upload id, `profile_photo_url` is the upload's preview URL; for a path, it is the storage URL; with no logo, it is a generated avatar built from the company name. Document and report PDFs resolve the logo the same way.
- `light_logo_path` is sent and stored the same way, and returned with its resolved `light_logo_url`. Emails draw it on the brand colour of their header. Without one they put `profile_photo_path` on a white chip, and when neither is a PNG, JPG or GIF they write the company name instead.

## Update Payload

`PUT/PATCH /api/v1/app/{company}/access-management/companies/{managedCompany}`

Same fields as create, all optional. Omitted fields are left `unchanged`; send an explicit `null` to clear a nullable field.

## Response Shape

`POST`, `PATCH` and `GET /companies/{managedCompany}` return the company under `data.company`. `GET /companies` returns a paginated collection of the same object under `data`.

Company object keys:

| Key | Type | Notes |
| --- | --- | --- |
| `id` | int | |
| `name` | string | |
| `slug` | string | Generated from `name` on create |
| `tax_pin` | string&#124;null | |
| `withholds` | array[int]&#124;null | |
| `has_vat` | boolean | |
| `service_managed` | string&#124;null | |
| `address` | string&#124;null | |
| `postal_address` | string&#124;null | Always present |
| `company_phone` | string&#124;null | Always present |
| `company_email` | string&#124;null | Always present |
| `company_website` | string&#124;null | Always present |
| `company_tagline` | string&#124;null | Always present |
| `twitter_handle` | string&#124;null | Always present |
| `facebook_url` | string&#124;null | Always present |
| `services` | array[string] | Always present; `[]` when the company lists none |
| `brand_color` | string&#124;null | Always present |
| `text_color_on_brand_bg` | string&#124;null | Always present |
| `accent_color` | string&#124;null | Always present |
| `text_color_on_accent_bg` | string&#124;null | Always present |
| `profile_photo_path` | string&#124;null | Always present; `null` when no logo is set |
| `profile_photo_url` | string | Always present. Resolved by the `HasProfilePhoto` concern: storage URL for `profile_photo_path`, or a generated avatar built from the company name when no logo is set |
| `light_logo_path` | string&#124;null | Always present; `null` when no light logo is set |
| `light_logo_url` | string&#124;null | Always present. The upload's preview URL or the storage URL for `light_logo_path`; `null` when none is set (no avatar fallback) |
| `is_selected` | boolean | |
| `type_of_properties` | array&#124;null | |
| `collection_contract` | boolean | |
| `registration_type` | string&#124;null | |
| `registration_number` | string&#124;null | |
| `status` | object | `{ value, color, label }` |
| `created` / `updated` | object | `raw`, `formatted` (`d M, Y`), `diff` |
| `permissions` | object | Instance permissions for current user |
| `user` | object | Owner; only when `user` is loaded |
| `users` | array | Only with `include=companyUsers` |
| `registration_upload` / `tax_pin_cert_upload` / `other_docs_upload` | object | Only when the relation is loaded |

Key presence:

- The profile keys (`postal_address`, `company_phone`, `company_email`, `company_website`, `company_tagline`, `twitter_handle`, `facebook_url`, `services`, `brand_color`, `text_color_on_brand_bg`, `accent_color`, `text_color_on_accent_bg`, `profile_photo_path`, `light_logo_path`, `light_logo_url`) are always returned, `null` when unset — except `services`, which is `[]`.
- `profile_photo_url` is always returned and never `null` (avatar fallback).
- Other scalar keys use `whenHas`, so they are omitted from the payload when the underlying value is `null`.

## Show / Update / Delete Company

- `GET /companies/{managedCompany}`
- `PATCH /companies/{managedCompany}`
- `DELETE /companies/{managedCompany}`

Access is validated against the target company (`managedCompany`), not only route context company.
