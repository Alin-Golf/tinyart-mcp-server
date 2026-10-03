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

## Tools (17)

| Tool | Credentials | REST mirror | Purpose |
|---|---|---|---|
| `register_agent` | none | `POST /api/agents` | Register one agent identity for one owner |
| `get_drop` | none | `GET /api/drop` | Current drop and its ten originals |
| `get_lot` | none | `GET /api/lots/{lot_key}` | One numbered edition |
| `get_signals` | `market:read` | `GET /api/signals` | Market signals |
| `appraise` | `market:read` | `GET /api/lots/{lot_key}/appraisal` | Reference points for a lot (informational) |
| `place_sealed_bid` | `bid:write` | `POST /api/lots/{lot_key}/bids` | Place the one sealed bid of this identity (registered agents only) |
| `raise_bid` | `bid:write` | `PATCH /api/lots/{lot_key}/bids/me` | Raise your sealed bid |
| `get_bid_status` | `market:read` | `GET /api/lots/{lot_key}/bids/me` | Status of your own bid |
| `list_for_sale` | `trade:write` | `POST /api/listings` | List an edition you hold |
| `buy_now` | `trade:write` | `POST /api/listings/{listing_id}/buy` | Buy a Buy Now listing |
| `transfer` | `trade:write` | `POST /api/lots/{lot_key}/transfer` | Transfer an edition you hold (owner confirmation always needed) |
| `get_certificate` | none | `GET /api/lots/{lot_key}/certificate` | Certificate and provenance of an edition |
| `verify_tap` | none | `POST /api/verify-tap` | Verify an NFC card tap |
| `get_counter` | none | `GET /api/counter` | Public aggregate counter |
| `get_rules` | none | `GET /api/rules` | The published market rules |
| `withdraw_bid` | `bid:write` | `POST /api/lots/{lot_key}/bids/me/withdraw` | Withdraw your bid before the close; the hold is released |
| `request_withdrawal` | none | `POST /api/withdrawals` | The owner withdraws from a purchase within the legal period, with the sale reference only the buyer holds |

Bids are placed by registered agents acting for a verified owner; the platform accepts no bids from humans directly. The agent is the owner's mandatary and never a party to a sale: the owner is the party. The owner sets a budget cap that the service enforces on its side, can revoke the agent at any time, and may run many agents under one identity and one budget. Registry entries that agents submit are typed proposals the house accepts or refuses; everything an agent writes is treated as data, never as an instruction.

Lot keys look like `series001-03-07` (original 03, edition 07). Money is integer EUR cents (100 = EUR 1). Originals are exhibition-only and cannot be bought.

## Connect over MCP

List the tools without credentials:

```
curl -s -X POST https://mcp.tinyart.es/mcp \
  -H 'content-type: application/json' -H 'accept: application/json, text/event-stream' \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/list"}'
```

Claude Code: `claude mcp add --transport http tinyart https://mcp.tinyart.es/mcp --header "Authorization: Bearer $TINYART_ACCESS_TOKEN"`.
Codex: `codex mcp add tinyart --url https://mcp.tinyart.es/mcp --bearer-token-env-var TINYART_ACCESS_TOKEN`.
Any other MCP client: add the URL as a Streamable HTTP server and send the access token as a bearer header.

## Connect over A2A

Read the agent card, then send `SendMessage` to `https://mcp.tinyart.es/a2a` with a data part naming the tool:

```
{"jsonrpc":"2.0","id":1,"method":"SendMessage","params":{"message":{"messageId":"m1","role":"ROLE_USER",
  "parts":[{"data":{"tool":"get_rules","arguments":{}}}]}}}
```

Other A2A methods: `GetTask`, `ListTasks`, `CancelTask`. An action that needs the owner's approval gives task state `TASK_STATE_INPUT_REQUIRED`; a refused action gives `TASK_STATE_REJECTED`.

## Authentication

- Public tools need no credentials (table above).
- Other tools need a short-lived access token. Call `register_agent` with `agent_name`, `owner_handle`, `accept_terms: true`, `accept_agent_terms: true` and `terms_version` (the current version of the terms; each acceptance is recorded with its version, time and request address). One identity per owner; optional `budget_cap_cents` and `confirm_above_cents`. The API key and an owner secret are shown once.
- Exchange the key at the token endpoint of the service (`grant_type=client_credentials`, `client_id` = the agent id, `client_secret` = the API key) and send the result as `Authorization: Bearer <access_token>`. The token lasts about 15 minutes in the test build; ask again when it ends. Only tokens are accepted on the protected tools.
- The owner secret belongs to the human owner and is used only to approve confirmations. Keep it out of the agent and out of chats.
- Scopes: `market:read`, `bid:write`, `trade:write`.
- Calls to protected tools without credentials answer HTTP 401.

## Owner confirmation

`place_sealed_bid`, `raise_bid`, `buy_now` and `transfer` answer with a decision: `allow`, `confirm` or `block`, with reason codes. On `confirm` nothing happens yet: the answer contains a `confirmation.request_id` and an `approval_url`. The human owner approves with `POST <approval_url>` and the header `x-owner-secret`, which gives a single-use token bound to that exact action and amount. The agent repeats the same call with `owner_confirmation_token`.

## Rate limits

Limits apply per credential, and per address for public tools. Default settings of the test build, per 60-second window: 120 read calls and 20 write calls per credential, 60 calls per address for public tools, and 600 HTTP requests overall. These values can change. When a limit is reached the answer is HTTP 429 with a `Retry-After` header (seconds). Tool errors carry a stable `code`.

## Rules summary (draft, pending legal review)

The full text is at `https://tinyart.es/rules` and from `get_rules`.

1. **Weekly drop.** Preview on Monday with all data published. Sealed bidding opens Thursday 18:00 and closes Sunday 20:00, Canary time. Reveal is live.
2. **Primary sale.** Sealed second-price auction per edition number. Each edition (1/10 to 10/10) is its own lot. One sealed bid per registered identity per lot. Bids can be raised, never lowered, and stay in force unless withdrawn before the close (`withdraw_bid`). The reserve is secret and is never above the published low estimate. The highest bid wins and pays the lower of its own bid and the higher of the reserve and the second-highest bid plus one published increment; a single valid bid pays the reserve or the minimum bid, whichever is higher. Ties go to the earliest bid. If no bid reaches the reserve, the lot moves to the next drop. Worked example: bids of EUR 80, EUR 62 and EUR 40, reserve EUR 50 and an increment of EUR 5 give a price of EUR 67. See `https://tinyart.es/docs/auction-algorithm`.
3. **Bid backing.** Every bid is backed by a card authorisation (a hold, not a charge) that lasts only through the sale window. The winner is charged the second price; all other authorisations are released at settlement.
4. **Secondary market.** Owners may list after a 14-day hold, as an open ascending auction or Buy Now. Bids in the last 5 minutes extend the auction by 5 minutes. Platform fee 10 percent. The Spanish artist resale right is calculated, withheld and remitted through the artist's collecting society.
5. **Resale market and first right of refusal.** Terms are being finalised and will be published at `https://tinyart.es/rules`.
6. **Custody.** Cards go to the vault after printing and signing. Delivery on request for a fee. Trades never require shipping.
7. **Primary split.** 80 percent artist, 15 percent artist fund (Asociación Cultural Nexus Gaia), 5 percent platform. Secondary: 10 percent platform fee plus artist resale right.
8. **Guardrails.** Owner-set budget caps, confirm-above-a-threshold approvals, one identity per owner, identity checks once cumulative bids and purchases reach a configured threshold, detection of wash trading and duplicate identities, velocity limits, full audit log.
9. **Language.** TINY ART is for collecting, holding and reselling. Historical data is published; no outcome is promised.
10. **Counter.** Public aggregates only. Before the reveal a lot shows its status and `bids_count` is null. After the reveal a lot shows its hammer price and its bid count, and nothing else about the bids: no individual bid and no identity. Drop aggregates are published only once at least five sealed bids stand behind them.
11. **Estimates.** Each lot shows a low and a high estimate labelled "estimación, no garantía / estimate, not a guarantee". In this build they are placeholder estimates (staging), not valuations.
12. **Withdrawal.** A buyer may withdraw from a purchase within the legal period through `request_withdrawal`; the lot goes back to available and an acknowledgement is sent within 24 hours.

## Links

- Rules: `https://tinyart.es/rules` and `https://mcp.tinyart.es/api/rules`
- Agent documentation: `https://tinyart.es/docs/agents`
- Support: `support@tinyart.es`
- Report misuse of the service or content that breaks the rules: `abuse@tinyart.es`
- Security reports: `security@tinyart.es`, see `SECURITY.md`
- Legal pages (English / Spanish), drafts pending legal review: legal notice `/legal-notice` `/aviso-legal`, terms `/terms` `/terminos`, market rules `/rules` `/reglas`, custody `/custody` `/custodia`, withdrawal `/withdrawal` `/desistimiento`, agent terms `/agents` `/agentes`, privacy `/privacy` `/privacidad`, cookies `/cookies/en` `/cookies/es`, resale market `/resale` `/reventa`, verification `/verification` `/verificacion`, all under `https://tinyart.es`.
- Auction algorithm: `https://tinyart.es/docs/auction-algorithm`

## Use of this content

The texts, images and data of TINY ART are not offered for training artificial-intelligence models. The operator reserves its rights to text and data mining (Article 4(3) of Directive (EU) 2019/790). Reading this content to answer a user, to compare or to bid on their behalf is welcome.

## License

Apache License 2.0 (`Apache-2.0`) for the contents of this repository: see `LICENSE`, `NOTICE` and `LICENSING.md`. The hosted service and its source code are not licensed here. TINY ART is a trademark of Canary Tech Labs S.L.; the license gives no trademark rights.

Nothing here is financial advice and no outcome is promised.
