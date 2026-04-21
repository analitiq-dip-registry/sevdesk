# sevdesk Connector

## What is this?

This is an [Analitiq](https://analitiq.io) connector definition for **sevdesk**, a cloud-based accounting and invoicing platform. A connector defines how to authenticate and communicate with an external system. Users create connections from this template by filling in their own credentials.

This connector is part of the [Analitiq DIP Registry](https://github.com/analitiq-dip-registry) and can be used with Analitiq Cloud or the open-source analitiq-core pipeline engine.

## Prerequisites

- A sevdesk account at [my.sevdesk.de](https://my.sevdesk.de)
- An API token (32-character hexadecimal string) generated from Settings > Users > select a user

## Authentication

sevdesk uses API token authentication. No OAuth app registration is required.

1. Log in to [my.sevdesk.de](https://my.sevdesk.de).
2. Navigate to **Settings > Users**.
3. Select the user whose API token you want to use.
4. Copy the 32-character hexadecimal API token.
5. Paste the token into the "API Token" field when creating a connection.

The token is sent as-is in the `Authorization` header (no "Bearer" prefix). The token has infinite lifetime and does not expire unless manually revoked.

## Available Endpoints

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/CheckAccountTransaction` | GET | Retrieve check account transactions (bank payments) |

## Limitations

- **Rate limits:** Not documented by sevdesk. No rate limit configuration is applied.
- **Pagination:** Offset-based with a maximum page size of 1000 records.
- **Timeout:** 30 seconds per request.

## How to use this connector

**Analitiq Cloud:** Import this connector from the DIP registry and create a connection by providing your API token.

**Open-source (analitiq-core):** Download this connector definition and use it with the analitiq-core pipeline engine. See [analitiq-core documentation](https://github.com/analitiq-dip-registry/analitiq-core) for setup instructions.

## For AI agents

See [CLAUDE.md](CLAUDE.md) (for Claude Code) or [AGENTS.md](AGENTS.md) (for other agent frameworks) for machine-readable connector reference including auth details, endpoints, and caveats.

## Contributing

To improve this connector (add endpoints, fix auth details, update documentation), see the [Analitiq Connector Builder](https://github.com/analitiq-dip-registry/analitiq-connector-builder) plugin.

## Links

- [sevdesk API Documentation](https://api.sevdesk.de/)
- [sevdesk API News](https://tech.sevdesk.com/api_news/)
- [Analitiq Cloud](https://analitiq.io)
- [analitiq-core](https://github.com/analitiq-dip-registry/analitiq-core)