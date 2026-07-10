---
name: sevdesk
description: >
  Connector for the sevdesk cloud accounting and invoicing API
type: api
---

# sevdesk

sevdesk is a cloud-based accounting and invoicing platform for small businesses and freelancers in the DACH region. The API provides access to contacts, invoices, vouchers, bank transactions, and other accounting data.

## Authentication

### API Key
- Client app required: no
- Header format: `Authorization: ${api_token}` (raw token, no Bearer prefix)
- Token is a 32-character hexadecimal string with infinite lifetime
- Obtained from my.sevdesk.de under Extensions (Erweiterungen) > API (reveal with your account password)

## Post-Auth Steps

None required. The API token is account-scoped (one token per sevdesk account / SevClient).

## Available Endpoints

All endpoints expose `read` (GET list) plus `write.insert` (POST) and `write.upsert` (PUT/{id}) where the API supports them. Reference-data endpoints are read-only.

Each endpoint's `endpoint_id` (and its file basename) equals the handle derived from its `request.path` — lowercased, path-params dropped — per the api-endpoint `endpoint-id-locator` rule.

### Contacts and contact metadata
- `contact` — `/Contact` — read, insert, upsert
- `contactaddress` — `/ContactAddress` — read, insert, upsert
- `communicationway` — `/CommunicationWay` — read, insert, upsert
- `contactcustomfield` — `/ContactCustomField` — read, insert, upsert
- `accountingcontact` — `/AccountingContact` — read, insert, upsert

### Sales documents
- `invoice` — `/Invoice` — read, insert, upsert
- `invoicepos` — `/InvoicePos` — read, insert, upsert
- `creditnote` — `/CreditNote` — read, insert, upsert
- `creditnotepos` — `/CreditNotePos` — read, insert, upsert
- `order` — `/Order` — read, insert, upsert
- `orderpos` — `/OrderPos` — read, insert, upsert

### Purchase / bookkeeping documents
- `voucher` — `/Voucher` — read, insert, upsert
- `voucherpos` — `/VoucherPos` — read, insert, upsert

### Catalog and inventory
- `part` — `/Part` — read, insert, upsert

### Banking
- `checkaccount` — `/CheckAccount` — read, insert, upsert
- `checkaccounttransaction` — `/CheckAccountTransaction` — read, insert, upsert
- `privatetransactionrule` — `/PrivateTransactionRule` — read, insert, upsert

### Classification and templates
- `category` — `/Category` — read, insert, upsert
- `tag` — `/Tag` — read, insert, upsert
- `tagrelation` — `/TagRelation` — read, insert (no PUT — recreate to change)
- `layout` — `/Layout` — read, insert, upsert

### Reference data (read-only)
- `staticcountry` — `/StaticCountry`
- `staticcurrency` — `/StaticCurrency`
- `unity` — `/Unity`
- `accountingtype` — `/AccountingType`

## Pagination

Offset-based: `limit` (max 1000, defaults to `runtime.batch_size`) and `offset` (initial 0). Read operations stop when the `objects` array in the response is empty.

## Replication

Currently authored as `full_refresh` for all resources. Several resources (Invoice, CreditNote, Order, Voucher, CheckAccountTransaction) accept `startDate`/`endDate` filters that could be used for incremental replication; this can be added per-endpoint when needed.

## Rate Limits

Not documented by sevdesk. No rate limit configuration is applied.

## Caveats

- The `Authorization` header uses the raw API token value without a `Bearer` prefix. This differs from most API key connectors.
- The API base URL is `https://my.sevdesk.de/api/v1` — all endpoint paths are relative to this.
- All sevdesk POST endpoints wrap the created/updated record in `{ "objects": <Resource> }`. Endpoint definitions extract `id` via `response.body.objects.id`.
- Many writable fields are sevdesk relations: `{ "id": ..., "objectName": "Category" }` style. Each endpoint's `input.schema` documents the expected `objectName` per relation.
- POST `/Contact`, `/Invoice`, `/CreditNote`, `/Order`, `/Voucher` create base records only. For atomic create-with-positions, sevdesk exposes `/Invoice/Factory/saveInvoice`, `/CreditNote/Factory/saveCreditNote`, `/Order/Factory/saveOrder`, `/Voucher/Factory/saveVoucher` — not currently modeled here; positions can be created separately via the `*pos` position endpoints (`invoicepos`, `creditnotepos`, `orderpos`, `voucherpos`).
- Action endpoints (`sendBy`, `sendViaEmail`, `bookAmount`, `enshrine`, `resetToDraft`, `resetToOpen`) are not modeled — they are POST side-effects on existing documents that don't fit the read/write CRUD operation contract.
