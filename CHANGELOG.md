# Changelog

## [3.0.0] - 2026-07-13

### Changed
- fix: type sevdesk relation fields as Json instead of Object (#13)

## [2.0.0] - 2026-07-10

### Changed
- fix: conform sevdesk endpoints to api-endpoint contract validator (#12)

## [1.2.1] - 2026-07-05

### Fixed
- fix: conform sevdesk relation fields to the Object marker rule (#10)

## [1.2.0] - 2026-06-30

### Added
- Update sevdesk connector to latest contract schemas (#9)

## [1.1.2] - 2026-06-22

### Fixed
- bug: retrigger version-bump to refresh registry webhook (#6)

## [1.1.1] - 2026-05-15

### Fixed
- bug: retrigger version-bump to retest webhook after connector.json migration (#5)

## [1.1.0] - 2026-05-08

### Added
- 24 new endpoint definitions covering the full sevdesk API for read/write
  operations: contacts, contact_addresses, communication_ways, contact_fields,
  accounting_contacts, invoices, invoice_positions, credit_notes,
  credit_note_positions, orders, order_positions, vouchers, voucher_positions,
  parts, check_accounts, categories, tags, tag_relations, layouts,
  private_transaction_rules, static_countries (read-only), static_currencies
  (read-only), unities (read-only), accounting_types (read-only).
- Write operations (`insert` via POST, `upsert` via PUT) on all CRUD-capable
  resources, plus on the existing check_account_transactions endpoint.

### Changed
- check_account_transactions: added insert/upsert write operations and aligned
  fields with the new api-endpoint schema (dropped `kind` and
  `endpoint_schema_version`, fixed `$schema` URL).
- connector.json: removed reserved server-managed fields (`connector_id`,
  `connector_schema_version`, `created_at`, `updated_at`); these are stamped
  by the registry.

## [0.0.3] - 2026-05-15

### Fixed
- bug: match Analitiq webhook API Gateway schema exactly (#4)

## [0.0.2] - 2026-04-27

### Added
- feat: consolidate manifest into connector.json and add type map (#2)

## [0.0.1] - 2026-04-21

### Fixed
- feat: initial sevdesk connector with /CheckAccountTransaction endpoint (#1)
