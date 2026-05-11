# Finance API

Domain: `Property Management`

Base prefix:

`/api/v1/app/{company}/property-management/finance`

## Resources

- [Bills](./finance/bill.md)
- [Expenses](./finance/expense.md)
- [Payment Vouchers](./finance/payment-voucher.md)
- [Remittances](./finance/remittance.md)
- [Settlements](./finance/settlement.md)

## Notes

- All list endpoints support pagination via `per_page` and `page`.
- Search-enabled list endpoints use `filter[search]` (Scout-backed custom filters).
- Sort support is documented per resource.

## Errors

Use shared behavior in `docs/frontend/app/README.md`:

- Render `4xx` errors/messages directly.
- Show generic fallback for `5xx`.
