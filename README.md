# WhatsApp Data MCP Server

Hosted [Model Context Protocol](https://modelcontextprotocol.io) server for the
[WhatsApp Data API](https://whatsapp.checkleaked.cc) by CheckLeaked. It gives AI assistants and agents read-only tools
to check whether a phone number is on WhatsApp, read its public profile (display name, about text, profile picture,
business profile), run bulk checks, search a database of WhatsApp business and profile records, and add phone OSINT
enrichments (carrier, Telegram, Google results, data-breach check, AI face analysis).

- **Hosted:** nothing to install. Point your MCP client at the URL below.
- **Two ways to pay:** a prepaid API key, or no account at all with [x402](https://www.x402.org) pay-per-call in USDC on Base.
- **Read-only:** no messages are sent and nothing is written. Looking up a number does not notify its owner.
- **Official MCP Registry:** listed as [`com.checkleaked/whatsapp-data`](https://registry.modelcontextprotocol.io/v0.1/servers?search=com.checkleaked/whatsapp-data).

This repository holds the documentation, client configuration examples and the registry manifest
([`server.json`](server.json)). The server itself runs as a hosted service; its source code is not published here.

## Contents

- [Endpoints](#endpoints)
- [Authentication](#authentication)
- [Tools](#tools)
- [Client setup](#client-setup)
- [Paying per call with x402](#paying-per-call-with-x402)
- [Try it with curl](#try-it-with-curl)
- [Example prompts](#example-prompts)
- [Privacy, opt-out and acceptable use](#privacy-opt-out-and-acceptable-use)
- [Links](#links)

## Endpoints

| Transport | URL |
| --- | --- |
| Streamable HTTP (stateless, recommended) | `https://whatsapp-mcp.checkleaked.com/mcp` |
| SSE (legacy clients) | `https://whatsapp-mcp.checkleaked.com/sse` |

Plain HTTP helpers on the same host:

| Path | What it returns |
| --- | --- |
| [`GET /health`](https://whatsapp-mcp.checkleaked.com/health) | Liveness, server version, tool count, x402 status |
| [`GET /pricing`](https://whatsapp-mcp.checkleaked.com/pricing) | Live x402 price list (JSON) |
| [`GET /`](https://whatsapp-mcp.checkleaked.com/) | Service description |

## Authentication

The server picks the payment mode per connection, from the headers your client sends.

### Option 1: API key (best for chat apps and volume)

Send your key in the `x-rapidapi-key` header (`Authorization: Bearer <key>` also works). Keys are prepaid credit
packs from [whatsapp.checkleaked.cc/pricing](https://whatsapp.checkleaked.cc/pricing). Calls draw from the key's
quota under the same rules as the REST API: only completed lookups count, and errors (invalid input, rate limits,
timeouts, server errors) never do. See the [API docs](https://whatsapp.checkleaked.cc/docs) for quotas and rate limits.

With a key you get the 18 data tools and never see x402.

### Option 2: x402 pay-per-call (no account, for agents)

Connect **without** a key. Listing tools is free, and so are `whatsapp_system_status` and `whatsapp_pricing`. Every
other tool call is paid in USDC on Base (`eip155:8453`) with the x402 v2 MCP transport:

1. The agent calls a tool with no payment. The result is `isError: true` with an x402 `PaymentRequired` object in
   `structuredContent` (also JSON-encoded in `content[0].text`): price, asset, network and recipient.
2. The agent signs an EIP-3009 USDC authorization and repeats the call with it in `params._meta["x402/payment"]`.
3. The server verifies the payment, runs the tool, settles on-chain and returns the answer with the receipt
   (transaction hash) in `result._meta["x402/payment-response"]`.

**You pay only for answers.** Settlement happens only when the call returns a definitive result (found, not on
WhatsApp, no picture, no results). Invalid input, rate limits, timeouts and server errors are never charged, and one
signed payment buys exactly one answer.

Chat apps such as Claude Desktop, Cursor and VS Code cannot sign x402 payments, so use an API key there. x402 is for
agents built with an x402 client (see [below](#paying-per-call-with-x402)).

## Tools

Prices are the x402 per-call prices in USD, settled in USDC. With an API key, calls use your key's quota instead.
The live list is always available from the free `whatsapp_pricing` tool or [`/pricing`](https://whatsapp-mcp.checkleaked.com/pricing).

### Number lookups

| Tool | What it does | x402 price |
| --- | --- | --- |
| `whatsapp_number_lookup` | Full lookup: registration, display name, about text, business profile and picture history, with optional Telegram, carrier, Google, leak check, LinkedIn match, AI face analysis, reverse image search and AI report | $0.02, plus enrichments (below) |
| `whatsapp_number_simple_check` | Fast check of whether a number is registered on WhatsApp, without profile data | $0.01 |
| `whatsapp_osint_info` | Registration status, device information and business profile through a second detection method | $0.02 |

`whatsapp_number_lookup` enrichment surcharges: `includeTelegram` +$0.005, `includeCarrier` +$0.005,
`includeGoogle` +$0.01, `includeFaceAnalysis` +$0.03, `includeReverseImageSearch` +$0.03,
`includeLeakCheckPro` +$0.03, `includeFullAiReport` +$0.05.

### Bulk checks

| Tool | What it does | x402 price |
| --- | --- | --- |
| `whatsapp_bulk_check_live` | Live registration check of up to 50 numbers, with ban status and business info | $0.01 per number |
| `whatsapp_bulk_check_db` | Checks up to 1,000 numbers against the database, without live WhatsApp queries | $0.002 per number (min $0.01) |

### Database search

| Tool | What it does | x402 price |
| --- | --- | --- |
| `whatsapp_phone_details` | Stored record for one number, optionally with its modification history | $0.01 |
| `whatsapp_phone_pagination` | Browse the database by date or number, with text, country and date-range filters | $0.01 |
| `whatsapp_phone_search` | Advanced search by country, business status, username, email, website, address, categories, profile picture, dates and face-analysis attributes | $0.01 |
| `whatsapp_phone_stats` | Aggregate statistics: total records, country distribution, business vs personal accounts | $0.01 |
| `whatsapp_business_address_search` | WhatsApp Business accounts with a registered address, including description, categories, email and website | $0.01 |
| `whatsapp_proximity_search` | WhatsApp Business accounts within a radius (km) of a latitude/longitude | $0.01 |

### Enrichment and OSINT

| Tool | What it does | x402 price |
| --- | --- | --- |
| `whatsapp_profile_picture` | Current profile picture: a URL with an API key, the image itself in x402 mode | $0.02 |
| `whatsapp_carrier_lookup` | Carrier/operator, line type and country | $0.01 |
| `whatsapp_telegram_check` | Telegram registration and available profile data for up to 20 numbers | $0.01 per number |
| `whatsapp_google_search` | Public web results that mention the number | $0.02 |
| `whatsapp_face_analysis` | AI estimate of age, gender and visible attributes from the stored profile picture | $0.05 |
| `whatsapp_leak_check` | Known data breaches that contain the number, with sources and exposed data types | $0.05 |

### System

| Tool | What it does | x402 price |
| --- | --- | --- |
| `whatsapp_system_status` | Service health: availability, working instances and error rates | Free |
| `whatsapp_pricing` | The live x402 price list and how payment works (keyless connections only) | Free |

Phone numbers go in international format with the country code, for example `+1 202 555 0143` or `12025550143`.

## Client setup

Replace `YOUR_API_KEY` with your key from [whatsapp.checkleaked.cc/pricing](https://whatsapp.checkleaked.cc/pricing).
Keep keys out of files you commit.

### Claude Code

```bash
claude mcp add --transport http whatsapp-data https://whatsapp-mcp.checkleaked.com/mcp \
  --header "x-rapidapi-key: YOUR_API_KEY"
```

Add `--scope user` to make it available in every project. Check it with `claude mcp list` or `/mcp`.

### Claude Desktop

Claude Desktop's JSON config starts local servers, so bridge the remote endpoint with
[`mcp-remote`](https://www.npmjs.com/package/mcp-remote) (needs Node.js). Edit `claude_desktop_config.json`
(macOS: `~/Library/Application Support/Claude/`, Windows: `%APPDATA%\Claude\`):

```json
{
  "mcpServers": {
    "whatsapp-data": {
      "command": "npx",
      "args": [
        "-y",
        "mcp-remote",
        "https://whatsapp-mcp.checkleaked.com/mcp",
        "--header",
        "x-rapidapi-key:${WHATSAPP_API_KEY}"
      ],
      "env": {
        "WHATSAPP_API_KEY": "YOUR_API_KEY"
      }
    }
  }
}
```

Restart Claude Desktop after saving.

### Cursor

`~/.cursor/mcp.json` (all projects) or `.cursor/mcp.json` (one project):

```json
{
  "mcpServers": {
    "whatsapp-data": {
      "url": "https://whatsapp-mcp.checkleaked.com/mcp",
      "headers": {
        "x-rapidapi-key": "YOUR_API_KEY"
      }
    }
  }
}
```

### VS Code (GitHub Copilot)

`.vscode/mcp.json`. VS Code asks for the key once and stores it securely:

```json
{
  "inputs": [
    {
      "type": "promptString",
      "id": "whatsapp-api-key",
      "description": "WhatsApp Data API key",
      "password": true
    }
  ],
  "servers": {
    "whatsapp-data": {
      "type": "http",
      "url": "https://whatsapp-mcp.checkleaked.com/mcp",
      "headers": {
        "x-rapidapi-key": "${input:whatsapp-api-key}"
      }
    }
  }
}
```

### Other clients

Any client that supports Streamable HTTP (or SSE) with a custom header works: use the endpoint URL and send
`x-rapidapi-key: YOUR_API_KEY`. A client that cannot send headers connects in x402 mode, where only the free tools
answer without payment.

## Paying per call with x402

Any x402 v2 MCP client works, for example [`@x402/mcp`](https://www.npmjs.com/package/@x402/mcp) in TypeScript.
The payer needs a wallet holding USDC on Base.

```typescript
import { createx402MCPClient } from '@x402/mcp'
import { ExactEvmScheme } from '@x402/evm/exact/client'
import { StreamableHTTPClientTransport } from '@modelcontextprotocol/sdk/client/streamableHttp.js'
import { privateKeyToAccount } from 'viem/accounts'

const client = createx402MCPClient({
  name: 'my-agent',
  version: '1.0.0',
  schemes: [
    {
      network: 'eip155:8453', // Base mainnet
      client: new ExactEvmScheme(privateKeyToAccount(process.env.PAYER_PRIVATE_KEY as `0x${string}`)),
    },
  ],
  autoPayment: true,
})

await client.connect(new StreamableHTTPClientTransport(new URL('https://whatsapp-mcp.checkleaked.com/mcp')))

const result = await client.callTool('whatsapp_number_simple_check', { phoneNumber: '+12025550143' })
console.log(result.content, result.paymentResponse?.transaction)
```

Call `whatsapp_pricing` first to budget a task. Prepaid API keys are cheaper at volume.

## Try it with curl

List the tools (no key needed):

```bash
curl -s https://whatsapp-mcp.checkleaked.com/mcp \
  -H 'Content-Type: application/json' \
  -H 'Accept: application/json, text/event-stream' \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/list"}'
```

Call a tool with your key:

```bash
curl -s https://whatsapp-mcp.checkleaked.com/mcp \
  -H 'Content-Type: application/json' \
  -H 'Accept: application/json, text/event-stream' \
  -H "x-rapidapi-key: $WHATSAPP_API_KEY" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"whatsapp_number_simple_check","arguments":{"phoneNumber":"+12025550143"}}}'
```

## Example prompts

- "Is +1 202 555 0143 on WhatsApp? If it is, show the public profile and whether it is a business account."
- "Check these 30 numbers live on WhatsApp and give me a table of which ones are registered."
- "Find WhatsApp Business accounts within 5 km of 40.7128, -74.0060 and list their categories and websites."
- "Which carrier does this number use, and does it appear in any known data breach?"
- "How many records does the database hold per country?"

## Privacy, opt-out and acceptable use

- WhatsApp profile fields are what the number's owner makes visible on WhatsApp. Picture and about text come back
  empty when the owner has restricted them in WhatsApp's privacy settings.
- Phone number owners can hide their data from every channel and block future lookups at
  [whatsapp.checkleaked.cc/opt-out](https://whatsapp.checkleaked.cc/opt-out). Opted-out numbers return no data.
- You need a lawful basis (and the person's consent where the law requires it) for every lookup. Do not use results
  for stalking, harassment, discrimination, unsolicited marketing or eligibility decisions (employment, credit,
  housing, insurance). See the [Terms](https://whatsapp.checkleaked.cc/terms) and
  [Privacy Policy](https://whatsapp.checkleaked.cc/privacy).

WhatsApp is a trademark of WhatsApp LLC. This service is independent and is not affiliated with, endorsed by or
sponsored by WhatsApp LLC or Meta Platforms, Inc.

## Links

- Website: [whatsapp.checkleaked.cc](https://whatsapp.checkleaked.cc)
- API docs: [whatsapp.checkleaked.cc/docs](https://whatsapp.checkleaked.cc/docs)
- Pricing and API keys: [whatsapp.checkleaked.cc/pricing](https://whatsapp.checkleaked.cc/pricing)
- API status: [whatsapp.checkleaked.cc/api-status](https://whatsapp.checkleaked.cc/api-status)
- Changelog: [whatsapp.checkleaked.cc/changelog](https://whatsapp.checkleaked.cc/changelog)
- For LLMs: [whatsapp.checkleaked.cc/llms.txt](https://whatsapp.checkleaked.cc/llms.txt)
- MCP Registry entry: [`com.checkleaked/whatsapp-data`](https://registry.modelcontextprotocol.io/v0.1/servers?search=com.checkleaked/whatsapp-data)

Questions or problems with the MCP server: open an issue in this repository.

## License

The documentation and examples in this repository are released under the [MIT License](LICENSE). The hosted service
is governed by its [Terms of Service](https://whatsapp.checkleaked.cc/terms).
