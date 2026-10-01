<!-- SPDX-License-Identifier: Apache-2.0 -->
# TINY ART: agent-facing art drops (public description of the hosted service)

License: Apache-2.0 (documents and API description only; see `LICENSING.md`).

TINY ART is a weekly drop of numbered art editions that AI agents, and the people who own them, can browse, appraise, bid on and verify. Operated by Canary Tech Labs S.L. for Asociación Cultural Nexus Gaia. Series 001 is by Miguel González Rodríguez: 10 originals, each in 10 numbered editions.

**Status: test mode.** Series 001 data in this build is placeholder data (placeholder titles, placeholder prices). No real payment is taken and nothing is sold. The market rules are a draft pending legal review. The live domains are being set up; until `mcp.tinyart.es` resolves, the endpoints below are not reachable.

This repository holds the public description of the service: this README, `openapi.json`, `server.json` (Model Context Protocol registry entry), `llms.txt` and the security and contribution notes. The service's source code is not in this repository.

## Endpoints

| What | URL |
|---|---|
| MCP (Streamable HTTP, stateless, POST only) | `https://mcp.tinyart.es/mcp` |
| A2A 1.0 agent card | `https://mcp.tinyart.es/.well-known/agent-card.json` |
| A2A JSON-RPC | `https://mcp.tinyart.es/a2a` |
| REST mirror of the tools | `https://mcp.tinyart.es/api/...` (see `openapi.json`) |
| Guide for agents | `https://mcp.tinyart.es/agents.md` |
| Index for language models | `https://mcp.tinyart.es/llms.txt` |
| Website and rules | `https://tinyart.es` and `https://tinyart.es/rules` |

## Tools (15)

| Tool | Credentials | REST mirror | Purpose |
|---|---|---|---|
| `register_agent` | none | `POST /api/agents` | Register one agent identity for one owner |
| `get_drop` | none | `GET /api/drop` | Current drop and its ten originals |
| `get_lot` | none | `GET /api/lots/{lot_key}` | One numbered edition |
| `get_signals` | `market:read` | `GET /api/signals` | Market signals |
| `appraise` | `market:read` | `GET /api/lots/{lot_key}/appraisal` | Reference points for a lot (informational) |
| `place_sealed_bid` | `bid:write` | `POST /api/lots/{lot_key}/bids` | Place the one sealed bid of this identity |
| `raise_bid` | `bid:write` | `PATCH /api/lots/{lot_key}/bids/me` | Raise your sealed bid |
| `get_bid_status` | `market:read` | `GET /api/lots/{lot_key}/bids/me` | Status of your own bid |
| `list_for_sale` | `trade:write` | `POST /api/listings` | List an edition you hold |
| `buy_now` | `trade:write` | `POST /api/listings/{listing_id}/buy` | Buy a Buy Now listing |
| `transfer` | `trade:write` | `POST /api/lots/{lot_key}/transfer` | Transfer an edition you hold (owner confirmation always needed) |
| `get_certificate` | none | `GET /api/lots/{lot_key}/certificate` | Certificate and provenance of an edition |
| `verify_tap` | none | `POST /api/verify-tap` | Verify an NFC card tap |
| `get_counter` | none | `GET /api/counter` | Public aggregate counter |
| `get_rules` | none | `GET /api/rules` | The published market rules |

Lot keys look like `series001-03-07` (original 03, edition 07). Money is integer EUR cents (100 = EUR 1). Originals are exhibition-only and cannot be bought.

## Connect over MCP

List the tools without credentials:

```
curl -s -X POST https://mcp.tinyart.es/mcp \
  -H 'content-type: application/json' -H 'accept: application/json, text/event-stream' \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/list"}'
```

Claude Code: `claude mcp add --transport http tinyart https://mcp.tinyart.es/mcp --header "x-api-key: $TINYART_API_KEY"`.
Codex: `codex mcp add tinyart --url https://mcp.tinyart.es/mcp --bearer-token-env-var TINYART_API_KEY`.
Any other MCP client: add the URL as a Streamable HTTP server and send the key as a header.

## Connect over A2A

Read the agent card, then send `SendMessage` to `https://mcp.tinyart.es/a2a` with a data part naming the tool:

```
{"jsonrpc":"2.0","id":1,"method":"SendMessage","params":{"message":{"messageId":"m1","role":"ROLE_USER",
  "parts":[{"data":{"tool":"get_rules","arguments":{}}}]}}}
```

Other A2A methods: `GetTask`, `ListTasks`, `CancelTask`. An action that needs the owner's approval gives task state `TASK_STATE_INPUT_REQUIRED`; a refused action gives `TASK_STATE_REJECTED`.

## Authentication

- Public tools need no credentials (table above).
- Other tools need an API key (`tak_test_...`). Call `register_agent` with `agent_name` and `owner_handle` (one identity per owner; optional `budget_cap_cents` and `confirm_above_cents`). The API key and an owner secret are shown once. Send the key as `x-api-key: <key>` or `Authorization: Bearer <key>`.
- The owner secret belongs to the human owner and is used only to approve confirmations. Keep it out of the agent and out of chats.
- Scopes: `market:read`, `bid:write`, `trade:write`.
- Sign-in with OAuth for MCP clients that cannot send an API key is not offered yet.
- Calls to protected tools without credentials answer HTTP 401.

## Owner confirmation

`place_sealed_bid`, `raise_bid`, `buy_now` and `transfer` answer with a decision: `allow`, `confirm` or `block`, with reason codes. On `confirm` nothing happens yet: the answer contains a `confirmation.request_id` and an `approval_url`. The human owner approves with `POST <approval_url>` and the header `x-owner-secret`, which gives a single-use token bound to that exact action and amount. The agent repeats the same call with `owner_confirmation_token`.

## Rate limits

Limits apply per credential, and per address for public tools. Default settings of the test build, per 60-second window: 120 read calls and 20 write calls per credential, 60 calls per address for public tools, and 600 HTTP requests overall. These values can change. When a limit is reached the answer is HTTP 429 with a `Retry-After` header (seconds). Tool errors carry a stable `code`.

## Rules summary (draft, pending legal review)

The full text is at `https://tinyart.es/rules` and from `get_rules`.

1. **Weekly drop.** Preview on Monday with all data published. Sealed bidding opens Thursday 18:00 and closes Sunday 20:00 (Europe/Madrid). Reveal is live.
2. **Primary sale.** Sealed second-price auction per edition number. Each edition (1/10 to 10/10) is its own lot. One sealed bid per registered identity per lot. Bids can be raised, never lowered. The reserve is secret. The highest bid wins and pays the second-highest bid. If no bid reaches the reserve, the lot moves to the next drop.
3. **Bid backing.** Every bid is backed by a card authorisation (a hold, not a charge) that lasts only through the sale window. The winner is charged the second price; all other authorisations are released at settlement.
4. **Secondary market.** Owners may list after a 14-day hold, as an open ascending auction or Buy Now. Bids in the last 5 minutes extend the auction by 5 minutes. Platform fee 10 percent. The Spanish artist resale right is calculated, withheld and remitted through the artist's collecting society.
5. **Resale market and first right of refusal.** Terms are being finalised and will be published at `https://tinyart.es/rules`.
6. **Custody.** Cards go to the vault after printing and signing. Delivery on request for a fee. Trades never require shipping.
7. **Primary split.** 80 percent artist, 15 percent artist fund (Asociación Cultural Nexus Gaia), 5 percent platform. Secondary: 10 percent platform fee plus artist resale right.
8. **Guardrails.** Owner-set budget caps, confirm-above-a-threshold approvals, one identity per owner, identity checks once cumulative bids and purchases reach a configured threshold, detection of wash trading and duplicate identities, velocity limits, full audit log.
9. **Language.** TINY ART is for collecting, holding and reselling. Historical data is published; no outcome is promised.
10. **Counter.** Public aggregates only. No single bid amount is shown before the full reveal after close.

## Links

- Rules: `https://tinyart.es/rules` and `https://mcp.tinyart.es/api/rules`
- Agent documentation: `https://tinyart.es/docs/agents`
- Support: `support@tinyart.es`
- Report misuse of the service or content that breaks the rules: `abuse@tinyart.es`
- Security reports: `security@tinyart.es`, see `SECURITY.md`
- Privacy policy and terms: planned at `https://tinyart.es`, not published yet. [PLACEHOLDER: add the live URLs here]

## License

Apache License 2.0 (`Apache-2.0`) for the contents of this repository: see `LICENSE`, `NOTICE` and `LICENSING.md`. The hosted service and its source code are not licensed here. TINY ART is a trademark of Canary Tech Labs S.L.; the license gives no trademark rights.

Nothing here is financial advice and no outcome is promised.
