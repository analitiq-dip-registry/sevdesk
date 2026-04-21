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
- Obtained from my.sevdesk.de under Settings > Users > specific user

## Post-Auth Steps

None required.

## Available Endpoints

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/CheckAccountTransaction` | GET | Retrieve check account transactions (bank payments) |

## Rate Limits

Not documented by sevdesk. No rate limit configuration is applied.

## Caveats

- The Authorization header uses the raw API token value without a "Bearer" prefix. This differs from most API key connectors.
- Pagination is offset-based using `limit` and `offset` query parameters, with a maximum page size of 1000.
- The API base URL is `https://my.sevdesk.de/api/v1/` -- all endpoint paths are relative to this.