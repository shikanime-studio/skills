# NestJS + Prisma contract boundary notes

Session lesson:

- Prisma model fixtures in tests should use the generated `@prisma/client`
  types, not the shared API contract types.
- For `AdminRole`, the generated model uses `permissions: bigint`, while the
  shared `@cpn-console/shared` contract serializes permissions as a string.
- Raw DB fixtures should also follow the generated scalar types for dates
  (`Date`) and enum-like fields.
- Request-body fixtures should use the shared contract type
  (`typeof adminRoleContract.*.body._type`) so API payloads stay aligned with
  the public shape.
- Service return values that go through `toAdminRole`/similar mappers should be
  asserted as contract-shaped values (string permissions), even when the Prisma
  mock uses bigint internally.
- If the generated client shape is unclear, inspect
  `node_modules/.prisma/client/index.d.ts` (or run `prisma generate`) instead of
  guessing from the schema file.
