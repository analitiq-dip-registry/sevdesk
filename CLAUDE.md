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

The `api_token` input is declared at `phase: auth` (it was `pre_auth` before 4.0.0) and is not
listed in `required_for_activation` — the input's own `required: true` already guarantees it.

## Post-Auth Steps

No user-facing selection is required — the API token is account-scoped (one token per sevdesk
account / SevClient). One automatic discovery runs after auth: `GET /Tools/bookkeepingSystemVersion`
resolves the tenant's bookkeeping system version (`1.0` or `2.0`) into
`connection.discovered.bookkeeping_system_version`. It is informational and deliberately does not
gate activation, so a hiccup in that probe cannot make a valid token unusable.

## Available Endpoints

24 endpoints. Operations below reflect what sevdesk's **current** OpenAPI spec documents — several
resources that once accepted standalone writes no longer do, and document creation now goes through
`/Factory/save*` RPC routes rather than a plain `POST /<Resource>`.

Each endpoint's `endpoint_id` (and its file basename) equals the handle derived from its
`request.path` — lowercased, path-params dropped — per the api-endpoint `endpoint-id-locator` rule.

### Contacts and contact metadata
- `contact` — `/Contact` — read, insert, upsert
- `contactaddress` — `/ContactAddress` — read, insert, upsert
- `communicationway` — `/CommunicationWay` — read, insert, upsert
- `contactcustomfield` — `/ContactCustomField` — read, insert, upsert
- `accountingcontact` — `/AccountingContact` — read, insert, upsert

### Sales documents
- `invoice` — `/Invoice` — read, insert, upsert (both writes via `POST /Invoice/Factory/saveInvoice`)
- `invoicepos` — `/InvoicePos` — **read only** (positions are written through `saveInvoice`)
- `creditnote` — `/CreditNote` — read, insert, upsert
- `creditnotepos` — `/CreditNotePos` — **read only** (positions are written through `saveCreditNote`)
- `order` — `/Order` — read, insert, upsert
- `orderpos` — `/OrderPos` — read, upsert (**no insert**; creation only via `saveOrder`)

### Purchase / bookkeeping documents
- `voucher` — `/Voucher` — read, insert, upsert (both writes via `POST /Voucher/Factory/saveVoucher`)
- `voucherpos` — `/VoucherPos` — **read only** (positions are written through `saveVoucher`)

> The pre-4.0.0 `voucherpos` document declared `insert` as `POST /VoucherPos` and `upsert` as
> `PUT /VoucherPos/{id}`. Neither path has ever existed in the spec — `/VoucherPos` declares `get`
> only, and there is no `/VoucherPos/{id}` path item at all. Those two write modes were fabrications
> and are now gone; the read surface is unchanged.

### Catalog and inventory
- `part` — `/Part` — read, insert, upsert (upsert conflict key is `partNumber`)

### Banking
- `checkaccount` — `/CheckAccount` — read, insert (**no upsert**; insert targets
  `POST /CheckAccount/Factory/fileImportAccount` — bank/PayPal accounts must be created in the UI)
- `checkaccounttransaction` — `/CheckAccountTransaction` — read, insert (**no upsert**; `id` is
  read-only so no client-suppliable conflict key exists)
- `privatetransactionrule` — `/PrivateTransactionRule` — read, insert (**no upsert**; no PUT exists)

### Classification and templates
- `category` — `/Category` — read, insert, upsert
- `tag` — `/Tag` — read, insert, upsert (insert is `POST /Tag/Factory/create` and **requires**
  attaching to an existing Invoice/Voucher/Order/CreditNote — a standalone tag cannot be created)
- `tagrelation` — `/TagRelation` — **read only** (relations are created automatically with a tag)
- `layout` — `/Layout` — read, insert, upsert

### Reference data (read-only)
- `staticcountry` — `/StaticCountry`
- `staticcurrency` — `/StaticCurrency`
- `unity` — `/Unity`
- `accountingtype` — `/AccountingType`

> **Reference data is deliberately not re-authored from docs.** None of these six
> (plus `category` and `layout`) has a path entry in sevdesk's current OpenAPI file, yet they are
> demonstrably live — the spec's own prose says "For all countries, send a GET to /StaticCountry",
> and no deprecation exists anywhere in the api_news archive. Their endpoint documents are carried
> forward from the previous release because re-deriving them from documentation alone would narrow
> them to `id` + `objectName`. Treat the released field lists as the better source until a live
> probe says otherwise.

## Pagination

Offset-based: `limit` (defaults to `runtime.batch_size`) and `offset` (initial 0). Read operations
stop when the `objects` array in the response is empty. Since 2025-05-30 sevdesk **enforces**
`limit` as an integer in 1..1000 — anything outside that range returns HTTP 400 (it accepted up to
10000 before). `countAll=true` adds a `total` field, which sevdesk serializes as a string.

## Replication

Mostly `full_refresh`. Three endpoints declare incremental replication:

- `invoice`, `order` — cursor `update`, pushed down via the `updateAfter` filter (`gt`).
- `checkaccounttransaction` — cursor `valueDate`, via `startDate` (`gte`). Append-only: sevdesk
  exposes no update/modified-after filter here, so status transitions and edits to
  already-replicated rows are never re-emitted.

`creditnote` and `voucher` deliberately stay `full_refresh` — their `startDate`/`endDate` filters
window on the *document* date, not modification time, so they cannot back a cursor.

**Open caveat:** sevdesk never documents the epoch unit for `updateAfter`. It is authored as
`epoch_seconds`, inferred from the sibling `startDate`/`endDate` params being declared integer Unix
timestamps. If a live probe shows milliseconds, that is a one-token fix per endpoint.

## Rate Limits

Not documented by sevdesk. No rate limit configuration is applied.

## Type mapping

sevdesk serializes almost every scalar as a **JSON string** on the read path — ids, money, tax
rates, quantities and even booleans (`'0'`/`'1'`) — while its write models declare genuine JSON
types for the same fields. Read/write type asymmetry is therefore real and intentional; do not
"fix" it by symmetrizing.

As of 4.0.0 those wire strings map to `Utf8` rather than the previous `Decimal128(18, 4)`, because
the response models declare them as strings. This changes the downstream type of every money column
— the single largest semantic change in the release.

All `format: date-time` fields use native `timestamp` → `Timestamp(MICROSECOND, UTC)`. There is no
naive/zoneless temporal token by design: sevdesk emits Europe/Berlin offsets, and the zoneless-looking
doc examples (`01.01.2020`) are a reused placeholder, not a wire sample — the same field name appears
offset-bearing on one resource and zoneless on another, which tracks example quality rather than
serialization.

## Caveats

- The `Authorization` header uses the raw API token value without a `Bearer` prefix. This differs from most API key connectors. Token-as-URL-parameter auth was removed by sevdesk on 2025-04-29.
- The API base URL is `https://my.sevdesk.de/api/v1` — all endpoint paths are relative to this. There is no v2 REST API; the spec's `info.version: 2.0.0` is a document version.
- GET responses wrap records in `{ "objects": [...] }`. **Write responses do not** — `POST`/`PUT` return the bare model, so write `generated_keys` read `response.body.id`, not `response.body.objects.id`. Note the validator cannot resolve write-side `response.body` refs at all (a write mode has no `response.schema`), so a wrong ref validates clean and silently yields nothing.
- Many writable fields are sevdesk relations: `{ "id": ..., "objectName": "Category" }` style. These are opaque `Json` on the read path; each endpoint's `input.schema` documents the expected `objectName` per relation.
- Plain `POST /Invoice`, `/CreditNote`, `/Order`, `/Voucher` **no longer exist**. Document creation and update both go through `/Factory/saveX`, which takes a wrapper body (`{invoice, invoicePosSave[], invoicePosDelete}`) and writes the document together with its positions. First-party spec bug to be aware of: `saveInvoice`'s `required` list names `invoicePos` while the actual property is `invoicePosSave` (same mismatch on `saveOrder`).
- `GET /Contact` needs `depth=1` to return persons as well as organisations; without it only organisations come back.
- Bookkeeping system version (1.0 vs 2.0) is discovered post-auth into `connection.discovered.bookkeeping_system_version`. Under 2.0, `taxRule` replaces `taxType`/`taxSet`, voucher creation is limited to status 50/100, and booking on ACCOUNTING_TYPE is unsupported.
- Action endpoints (`sendBy`, `sendViaEmail`, `bookAmount`, `enshrine`, `resetToDraft`, `resetToOpen`) are not modeled — they are side-effects on existing documents that do not fit the read/write CRUD operation contract. `DELETE` is likewise unmodelable (the write-mode vocabulary is insert/upsert/truncate_insert).
- Upcoming: from 2026-04-07 the render/`changeParameter`/`sendByWithRender` endpoints return a single full `pdf` property instead of Base64 page-image arrays. Read-side collection endpoints are unaffected.
