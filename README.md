![Status: Unverified](https://img.shields.io/badge/status-unverified-yellow)
[![Latest release](https://img.shields.io/github/v/release/analitiq-dip-registry/sevdesk)](https://github.com/analitiq-dip-registry/sevdesk/releases)
[![License: Apache 2.0](https://img.shields.io/badge/license-Apache%202.0-blue)](LICENSE)

# sevdesk

Connector for the [sevdesk](https://sevdesk.de) cloud accounting and invoicing API. sevdesk is a
cloud-based accounting and invoicing platform for small businesses and freelancers in the DACH
region. This connector exposes **23 endpoints** covering contacts, invoices, credit notes, orders,
vouchers, bank transactions, products and the reference data those documents reference.

## What is this?

This is a **connector** -- a configuration that defines how to authenticate with sevdesk and what
data endpoints are available for reading and writing. It does not move data by itself. Instead, it is
used by the [Analitiq](https://analitiq-app.com) data integration platform or the open-source
`analitiq-core` engine to set up data pipelines.

## How to use this connector

There are two ways to use this connector:

### Option 1 -- Analitiq Cloud (no setup required)

All connectors from this registry are automatically available on [analitiq-app.com](https://analitiq-app.com). Simply log in, select the connector, and follow the on-screen instructions to connect your account.

### Option 2 -- Open Source (self-hosted)

All connectors are open source and free to use. To get started:

1. Clone the [analitiq-core](https://github.com/analitiq-dip-registry/analitiq-core) repository
2. Install the Claude plugin `analitiq-plugin-dataflow`
3. Launch Claude in the root directory of `analitiq-core`
4. Tell it: *"I need to move data from X to Y"*

The `analitiq-plugin-dataflow` plugin will automatically fetch the required connectors from the [Analitiq DIP Registry](https://github.com/analitiq-dip-registry) and set up the data flow pipeline for you.

## Prerequisites

- A sevdesk account with API access (available to sevdesk administrators).
- Your sevdesk API token.

## Authentication

sevdesk uses a single static **API token** sent in the `Authorization` header. Note that sevdesk does
**not** use a `Bearer` prefix -- the header value is the raw token:

```
Authorization: your32characterhexadecimaltoken
```

The token has an infinite lifetime and stays valid as long as the sevdesk user exists.

### How to get your credentials

1. Log in to [my.sevdesk.de](https://my.sevdesk.de) as an administrator.
2. Open **Extensions** (*Erweiterungen*) > **API**.
3. Reveal the token with your account password and copy it. It is a 32-character hexadecimal string.

Paste that token as the connector's **API Token** input. Nothing else is required -- the token is
account-scoped, so there is no workspace or company to select afterwards.

> Token-as-URL-parameter authentication was removed by sevdesk on 2025-04-29. The header is the only
> supported method.

## Available Endpoints

All reads are paginated and return records under an `objects` array.

| Endpoint | Path | Operations |
|---|---|---|
| `contact` | `/Contact` | read (incremental), insert, upsert |
| `contactaddress` | `/ContactAddress` | read, insert, upsert |
| `communicationway` | `/CommunicationWay` | read, insert, upsert |
| `communicationwaykey` | `/CommunicationWayKey` | read |
| `contactcustomfield` | `/ContactCustomField` | read, insert, upsert |
| `contactcustomfieldsetting` | `/ContactCustomFieldSetting` | read, insert, upsert |
| `accountingcontact` | `/AccountingContact` | read, insert, upsert |
| `invoice` | `/Invoice` | read (incremental), insert |
| `invoicepos` | `/InvoicePos` | read |
| `creditnote` | `/CreditNote` | read, insert, upsert |
| `creditnotepos` | `/CreditNotePos` | read |
| `order` | `/Order` | read, insert, upsert |
| `orderpos` | `/OrderPos` | read, upsert |
| `voucher` | `/Voucher` | read, insert, upsert |
| `voucherpos` | `/VoucherPos` | read |
| `part` | `/Part` | read, insert, upsert |
| `checkaccount` | `/CheckAccount` | read, insert, upsert |
| `checkaccounttransaction` | `/CheckAccountTransaction` | read, insert, upsert |
| `privatetransactionrule` | `/PrivateTransactionRule` | read, insert |
| `category` | `/Category` | read |
| `tag` | `/Tag` | read, insert, upsert |
| `tagrelation` | `/TagRelation` | read |
| `staticcountry` | `/StaticCountry` | read |

### Incremental sync

`contact` and `invoice` support incremental replication using their `update` field, pushed down to
sevdesk via the `updateAfter` filter. Every other endpoint is full-refresh only -- sevdesk documents
no modification-time filter for them. The `startDate`/`endDate` filters on invoices, orders, credit
notes and vouchers window the **document** date rather than the modification time, so they cannot
drive an incremental sync.

## Limitations

- **Invoices cannot be updated.** sevdesk exposes no `PUT /Invoice/<built-in function id>`, so `invoice` supports
  insert only. Use the sevdesk UI or the document action routes to modify an existing invoice.
- **Document positions are written with their parent.** `invoicepos`, `creditnotepos` and
  `voucherpos` are read-only. Positions are created by sending them inside the parent document's
  `save*` payload (for example `invoicePosSave` on an invoice). `orderpos` additionally supports
  updating an existing position.
- **Tags cannot be created standalone.** `POST /Tag/Factory/create` requires the tag to be attached
  to an existing invoice, voucher, order or credit note.
- **`category` requires an `objectType`.** sevdesk documents `/Category` only as
  `GET /Category?objectType=<X>` (for example `Part` or `ContactAddress`), so a stream must supply
  one. There is no default.
- **Page size is capped at 1000.** sevdesk rejects any `limit` outside 1..1000 with HTTP 400.
- **Bank and PayPal accounts must be created in the sevdesk UI.** The connector's `checkaccount`
  insert targets the file-import account route only.
- **No rate limits are documented** by sevdesk, so none are configured.
- **Numbers arrive as strings.** sevdesk serialises ids, money amounts and tax rates as JSON strings
  on the read path, so those columns land as text. Its write models accept real numbers for the same
  fields; this asymmetry is the provider's, and is preserved deliberately.
- Action routes (`sendBy`, `sendViaEmail`, `bookAmount`, `enshrine`, `render`, `getPdf`, ...) and the
  Export / Report / DATEV families are not modelled -- they are side-effects or file downloads rather
  than record collections.
- `staticcountry` and `category` have no entry in sevdesk's OpenAPI `paths:` object; their routes are
  documented only in sevdesk's own field descriptions, and their field lists are carried forward from
  the previous release.

## For AI agents

This connector includes `CLAUDE.md` and `AGENTS.md` files -- machine-readable references used by AI agents and agentic frameworks. They document authentication types, available endpoints, post-auth steps, and any caveats for programmatic use. Both files are kept identical -- `CLAUDE.md` is for Claude Code, `AGENTS.md` is for other agent frameworks.

## Create a connector to any system

You can create a new connector to any API or database using Claude and the Analitiq connector builder plugin:

1. Install [Claude Code](https://claude.ai/code)
2. Install the connector builder plugin:
   ```
   claude plugin add analitiq-dip-registry/analitiq-plugin-connector-builder
   ```
3. Launch Claude and say: *"I want to create a connector for [system name]"*
4. The plugin will interview you about the system, research its API documentation, and generate the full connector with all required files

No coding required -- the plugin handles authentication research, endpoint schema generation, and file creation automatically.

![Example of Claude building a connector](media/example_1.png)

## Contributing

All connectors in this registry are community-maintained and live at [github.com/analitiq-dip-registry](https://github.com/analitiq-dip-registry). To add new endpoints or improve an existing connector, install the [connector builder plugin](https://github.com/analitiq-dip-registry/analitiq-plugin-connector-builder) and follow its instructions.

## Links

- [sevdesk API Documentation](https://api.sevdesk.de/)
- [sevdesk API News](https://tech.sevdesk.com/api_news/)
- [Analitiq Cloud](https://analitiq-app.com)
- [Analitiq Engine (open source)](https://github.com/analitiq-dip-registry/analitiq-engine)

