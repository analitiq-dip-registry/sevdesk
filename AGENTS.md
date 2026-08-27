---
name: sevdesk
description: >
  Connector for the sevdesk cloud accounting and invoicing API
type: api
---

# sevdesk

sevdesk is a cloud-based accounting and invoicing platform for small businesses and freelancers in
the DACH region. The API exposes contacts, invoices, credit notes, vouchers, orders, bank
transactions and the reference data those documents reference.

This definition was re-authored from scratch against sevdesk's complete first-party OpenAPI
document (`https://api.sevdesk.de/openapi.yaml`, `info.version: 2.0.0`, 119 paths, 76 component
schemas) and validated against the published **rc23** contract.

## Authentication

### API Key
- Client app required: no
- Header: `Authorization: <api_token>` — the **raw token, no `Bearer` prefix**. The spec's own
  `securitySchemes` block is `api_key: {type: apiKey, name: Authorization, in: header}`, and the
  string `Bearer` does not occur anywhere in the document.
- Token is a 32-character hexadecimal string with infinite lifetime ("api tokens have an infinite
  lifetime and, in other words, exist as long as the sevdesk user exists").
- Obtained from my.sevdesk.de under Extensions (Erweiterungen) > API.
- Token-as-query-parameter auth was removed by sevdesk on 2025-04-29. Header only.

`api_token` is declared at `phase: auth` with `storage: secrets`, and is deliberately **not** listed
in `required_for_activation` — the input's own `required: true` already guarantees it. (Released
3.0.1 used `phase: pre_auth` plus a `required_for_activation` entry; both changed here.)

## Post-Auth Steps

No user-facing selection — the API token is account-scoped (one token per sevdesk account /
SevClient). One automatic discovery runs after auth: `GET /Tools/bookkeepingSystemVersion` resolves
the tenant's bookkeeping system version (`1.0` or `2.0`) into
`connection.discovered.bookkeeping_system_version` via `value_path: objects.version`. It is
informational and deliberately does not gate activation, so a hiccup in that probe cannot make a
valid token unusable.

## Available Endpoints

23 endpoints. Every read is `GET <collection>` with records at `response.body.objects`.

Each `endpoint_id` (and its file basename) equals the handle derived from its `request.path`.

### Contacts and contact metadata
- `contact` — `/Contact` — read (**incremental**), insert, upsert
- `contactaddress` — `/ContactAddress` — read, insert, upsert
- `communicationway` — `/CommunicationWay` — read, insert, upsert
- `communicationwaykey` — `/CommunicationWayKey` — **read only** (reference data)
- `contactcustomfield` — `/ContactCustomField` — read, insert, upsert
- `contactcustomfieldsetting` — `/ContactCustomFieldSetting` — read, insert, upsert
- `accountingcontact` — `/AccountingContact` — read, insert, upsert

### Sales documents
- `invoice` — `/Invoice` — read (**incremental**), **insert only**
- `invoicepos` — `/InvoicePos` — **read only**
- `creditnote` — `/CreditNote` — read, insert, upsert
- `creditnotepos` — `/CreditNotePos` — **read only**
- `order` — `/Order` — read, insert, upsert
- `orderpos` — `/OrderPos` — read, **upsert only (no insert)**

### Purchase / bookkeeping documents
- `voucher` — `/Voucher` — read, insert, upsert
- `voucherpos` — `/VoucherPos` — **read only**

### Catalog and inventory
- `part` — `/Part` — read, insert, upsert (**upsert conflict key is `partNumber`**, not `id`)

### Banking
- `checkaccount` — `/CheckAccount` — read, insert, upsert
- `checkaccounttransaction` — `/CheckAccountTransaction` — read, insert, upsert
- `privatetransactionrule` — `/PrivateTransactionRule` — read, **insert only**

### Classification and reference data
- `category` — `/Category` — **read only**, requires `objectType`
- `tag` — `/Tag` — read, insert, upsert
- `tagrelation` — `/TagRelation` — **read only**
- `staticcountry` — `/StaticCountry` — **read only**

### Write routes are not uniform — check before assuming

`/Invoice`, `/CreditNote`, `/Order`, `/Voucher` and `/CheckAccount` expose **no collection POST**.
Document creation goes through `/Factory/save*` RPC routes that write the document together with its
positions:

| endpoint | insert route | upsert route |
|---|---|---|
| `invoice` | `POST /Invoice/Factory/saveInvoice` | **none** — `/Invoice/{invoiceId}` is GET-only |
| `creditnote` | `POST /CreditNote/Factory/saveCreditNote` | `PUT /CreditNote/{id}` |
| `order` | `POST /Order/Factory/saveOrder` | `PUT /Order/{id}` |
| `voucher` | `POST /Voucher/Factory/saveVoucher` | `PUT /Voucher/{id}` |
| `checkaccount` | `POST /CheckAccount/Factory/fileImportAccount` | `PUT /CheckAccount/{id}` |
| `tag` | `POST /Tag/Factory/create` | `PUT /Tag/{id}` |
| `orderpos` | **none** — created via `saveOrder` | `PUT /OrderPos/{id}` |
| `privatetransactionrule` | `POST /PrivateTransactionRule` | **none** — `/{id}` is DELETE-only |

For the four factory-backed document endpoints the **record is the whole factory envelope**
(`{invoice, invoicePosSave[], …}`), not the bare document. That is deliberate: the `*pos` endpoints
are read-only, so the envelope is the only way positions can ever be written.

> **Spec bug to know about.** All four `save*` schemas name `invoicePos` / `creditNotePos` /
> `orderPos` / `voucherPos` in their `required` list, while the property they actually declare is
> `invoicePosSave` / `creditNotePosSave` / `orderPosSave` / `voucherPosSave`. The definitions author
> the **actual** property names.

### Endpoints removed in 4.0.0

`accountingtype`, `layout`, `staticcurrency` and `unity` were removed. None has a `paths:` entry in
sevdesk's OpenAPI document **and** none has any first-party prose documenting a route — they appear
only as nested object *types* inside other resources' field descriptions. `layout` in particular is
not a collection at all: layouts are read via `/DocServer/*` and changed via
`/{document}/{id}/changeParameter`.

`staticcountry` and `category` are **kept** despite also having no `paths:` entry, because sevdesk's
own field descriptions document their routes verbatim: *"For all countries, send a GET to
/StaticCountry"* and *"For all categories, send a GET to /Category?objectType=Part"*. Their response
**field lists** are not published anywhere in the spec and are carried forward from released 3.0.1 —
that is a known gap, closable only with a live payload.

## Pagination

Offset-based on every read: `limit` (default `runtime.batch_size`) and `offset` (initial 0),
stepping by `response.record_count` because sevdesk's offset counts **records**, not pages. Reads
stop when the `objects` array comes back empty.

sevdesk **enforces** `limit` as an integer in 1..1000 — anything outside returns HTTP 400 (it
accepted up to 10000 before 2025-05-30). Note that `limit`/`offset`/`countAll` are documented only in
the spec's `info.description`; they are not declared as parameters on the collection GETs, but they
are the real wire contract.

## Replication

Only **two** endpoints have grounded incremental support:

- `contact` and `invoice` — cursor field `update`, pushed down via `updateAfter` (`gt`,
  `epoch_seconds`).

Everything else is `full_refresh`, and that is a fact rather than a scoping decision: `updateAfter`
is documented **only** in the info-description filter tables for contacts and invoices, and is not a
declared parameter anywhere in `paths:`. The `startDate`/`endDate` filters on invoice, order,
creditnote and voucher window the **document** date, not modification time, so they cannot back a
cursor — they are authored as ordinary stream filters instead. The credit-note filter list
explicitly does not contain `updateAfter`.

`checkaccounttransaction` is the one resource whose `startDate`/`endDate` are declared
`string`/`date-time` rather than epoch integers; it still windows `valueDate`, so still no cursor.

**Epoch unit.** `updateAfter` is authored as `epoch_seconds`. sevdesk never states the unit for it
directly; the grounding is that the Export family declares `startDate`/`endDate` as `integer` with
10-digit samples (`1641032867`), and `Model_InvoiceResponse.accountNextInvoice` samples
`'1647259198'`. The `01.01.2020`-style examples on `/CreditNote` and `/Voucher` `startDate` params
are spec bugs, not evidence of a date format.

## Rate Limits

Not documented by sevdesk anywhere — no `429`, no quota, no headers. No rate limit is configured
(omitted rather than guessed). The adjacent real constraint is the 1..1000 `limit` cap above.

## Type mapping

sevdesk serialises almost every scalar as a **JSON string** on the read path — ids, money, tax rates,
quantities — while its write models declare genuine JSON types for the same fields. Read/write
asymmetry is real and intentional; do not "fix" it by symmetrising.

The read map has 8 rules covering the complete response vocabulary:

| native | canonical |
|---|---|
| `STRING` | `Utf8` |
| `DATE-TIME` | `Timestamp(MICROSECOND, UTC)` |
| `DATE` | `Date32` |
| `INTEGER` | `Int64` |
| `NUMBER` | `Float64` |
| `BOOLEAN` | `Boolean` |
| `OBJECT` | `Json` |
| `ARRAY` | `Json` |

Native derivation: a temporal `format` wins over the `type` token; otherwise the `type` token is the
native. `string+format:float` and `string+format:binary` therefore land as `string` → `Utf8` — one
native cannot render both `Utf8` and `Float64`, and a non-temporal format adds no Arrow information.

**Datetimes are zone-aware on evidence** (RULE-SHRD-002): every real ISO sample in the spec carries
an offset (`2024-05-10T00:00:00+02:00`, `2023-04-18T15:45:38+02:00`, `2026-01-16T12:51:22+01:00`).
Values spelled `01.01.2020` are a reused placeholder, not a wire sample.

**No read field is `Boolean`.** sevdesk's response models declare `type: boolean` on twelve fields
(`smallSettlement`, `showNet`, `net`, `isAsset`, `optional`, `stockEnabled`, `mapAll`), but the wire
sends the strings `"0"` / `"1"` — and the spec contradicts itself, giving those same fields quoted
string examples (`'0'`, `'1'`, `'true'`, `'false'`). Arrow refuses `str -> bool` deterministically, so
a `Boolean` declaration fails the read before the first batch with `CONFIG_INVALID`. All twelve are
typed `Utf8` on the read path; the write input schemas still take real booleans, which is what
sevdesk accepts on write. **Do not "restore" these to Boolean on the strength of the declared type —
4.0.0 did exactly that and broke every affected stream.**

**sevdesk relation fields** (`{"id": …, "objectName": "Category"}`) are opaque `Json` on the read
path and bare `type: object` in write input schemas — no `properties` sub-schema anywhere. The
expected `objectName` per relation lives in each field's description. Two earlier releases regressed
on exactly this.

## Caveats

- Write responses are **bare** — `POST`/`PUT` return the model at the top level, so write
  `generated_keys` read `response.body.id`. Three exceptions, all verified against the spec:
  `checkaccount` insert is wrapped (`response.body.objects.id`), `contactcustomfieldsetting` insert
  is wrapped **and an array** (`response.body.objects.0.id`), and the four factory routes return a
  composite (`response.body.invoice.id`, `.creditNote.id`, `.order.id`, `.voucher.id`).
  The validator **cannot** resolve write-side `response.body` refs at all — a write mode has no
  `response.schema` — so a wrong ref validates clean and silently yields nothing. Released 3.0.1 had
  `response.body.objects.id` on nearly every write mode, which was wrong in almost every case.
- Path placeholders are snake_case (`/Contact/{contact_id}`), because rc23 constrains them to
  `^[a-z][a-z0-9_]*$`. sevdesk's own spec spells them camelCase; the placeholder is a local binding
  name, not a wire value, so renaming is safe.
- `GET /Contact` needs `depth=1` to return persons as well as organisations; without it only
  organisations come back. The endpoint defaults it to `"1"`, and deliberately declares no
  `operators` on it so a stream cannot reset it to `"0"` and silently drop every person.
- `category` requires an `objectType` value and has **no default** — documented values are
  `ContactAddress` and `Part`, but the full domain is not enumerated anywhere, so no closed enum is
  declared and no default is invented. A stream must supply one.
- Bookkeeping system version 1.0 vs 2.0 changes the tax model: under 2.0 `taxRule` replaces
  `taxType`/`taxSet`, voucher creation is limited to status 50/100, and booking on ACCOUNTING_TYPE is
  unsupported. Several `save*` schemas mark **both** the 1.0 and 2.0 tax fields `required`, which no
  single tenant can satisfy; the definitions leave both optional and state the version in each
  description.
- Action endpoints (`sendBy`, `sendViaEmail`, `bookAmount`, `enshrine`, `resetToDraft`,
  `resetToOpen`, `render`, `getPdf`, `cancelInvoice`) are not modelled — they are side-effects on
  existing documents that do not fit the read/write CRUD operation contract. `DELETE` is likewise
  unmodelable (the write-mode vocabulary is insert/upsert/truncate_insert).
- The Export / Report / ReceiptGuidance / DocServer / Progress / Textparser families are not
  modelled: they return CSV/ZIP payloads or job handles, not paginated record collections.
- **sevdesk's own spec is internally inconsistent about scalar types, and the definitions mirror it
  rather than smoothing it over.** Two consequences are visible in the Arrow schema, and both are the
  provider's, not the connector's:
  - `id` lands as `Utf8` on 21 endpoints but `Int64` on `part` and `contactaddress`, because
    `Model_Part.id` and `Model_ContactAddressResponse.id` are declared `integer` while
    `Model_ContactResponse.id`, `Model_InvoiceResponse.id` and the rest are declared `string`.
  - Money lands as `Utf8` on the document endpoints but `Float64` on `part`, `invoice.paidAmount` and
    `voucher.paidAmount` — `Model_InvoiceResponse.sumNet` is declared `string` while
    `Model_InvoiceResponse.paidAmount`, in the same model, is declared `number/float`.

  Authoring these as declared is deliberate (RULE-CTOR-026/041: a fact the docs do not establish must
  be reported as a gap, not inferred). Given that sevdesk demonstrably serialises most scalars as
  JSON strings on the wire, the `integer`/`number` declarations are the ones more likely to be wrong.
  A live probe against a real account is the only way to settle it; if it shows strings, those fields
  become `Utf8` and that is another major bump.
- Known spec defects carried as declared and flagged in field descriptions:
  `Model_InvoicePosResponse.quantity` is declared `boolean` (it is numeric on the wire — authored as
  a string like every sibling position model); `Model_Part.id` and `Model_ContactAddressResponse.id`
  are declared `integer` while most siblings declare `string`; `/CommunicationWayKey` misspells its
  update field as `upadate` (authored verbatim — it is the wire name);
  `Model_VoucherPosResponse.update` has no `format` and only a placeholder example, so its
  zone-awareness is unestablished and it stays `Utf8`.
- Upcoming: from 2026-04-07 the render / `changeParameter` / `sendByWithRender` endpoints return a
  single full `pdf` property instead of Base64 page-image arrays. Read-side collection endpoints are
  unaffected. The published spec still lags this change.
