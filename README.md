# Nexus Exchange API

[![API Version](https://img.shields.io/github/v/release/nexus-xyz/nexus-exchange-api?label=api)](https://github.com/nexus-xyz/nexus-exchange-api/releases)
[![License](https://img.shields.io/badge/license-MIT%2FApache--2.0-blue.svg)](#license)

REST and WebSocket API for [Nexus Exchange](https://exchange.nexus.xyz) — a perpetual futures exchange for crypto, equities, FX, and commodities.

- **Base URL:** `https://exchange.nexus.xyz/api/exchange` — the current default, and it serves **testnet** (play funds). See [Networks](#networks) for the per-network bases.
- **Direct-service base:** `https://exchange.nexus.xyz/api/v1` — the market-data, account/trading, and bridge surfaces are also served directly by their backend service under an `/api/v1` prefix (routed by the load balancer). Both bases are live in parallel; the same HMAC signing applies, over the full request path (e.g. `/api/v1/orders`). Every `/api/v1` operation in the spec carries a `servers` override pinning this host, so it resolves correctly regardless of which top-level base a generator picks.
- **OpenAPI spec:** [`openapi.json`](./openapi.json)
- **Changelog:** [`CHANGELOG.md`](./CHANGELOG.md)
- **Interactive docs:** [exchange.nexus.xyz/api-docs](https://exchange.nexus.xyz/api-docs)

## Networks

The API is served per **network**, and the network is the *host* — not a path, not a header, and not a release channel. There are two public networks:

| Network | REST base | WebSocket | Funds |
|---|---|---|---|
| **Testnet** | `https://api.testnet.nexus.xyz/v1` | `wss://api.testnet.nexus.xyz` | play — synthetic USDX from the faucet, no real-world value |
| **Mainnet** | `https://api.nexus.xyz/v1` | `wss://api.nexus.xyz` | **real** — USDX bridged from Ethereum Mainnet |

WebSocket paths are identical on both hosts: market data at `…/stream`, authenticated at `…/ws?token=…`. `local` (`http://localhost:9090`) is a developer convenience, not a public network.

The spec carries this table as machine-readable `servers` entries plus an `x-nexus-networks` map — **the single place to copy the mapping from.** Mainnet is the real-funds exchange running against Ethereum Mainnet via the USDX bridge; it is not a Nexus L1 chain.

> **Status:** the two `api.` hostnames are decided and documented, but DNS/TLS is a separate infra change and they do not resolve yet. Keep pinning `https://exchange.nexus.xyz/api/exchange` until the cutover; the entries are published now so clients can build the network axis against a stable contract.

### Four things to get right

**Mainnet is deliberately off-pattern.** It takes the bare `api.nexus.xyz`, not `api.mainnet.nexus.xyz`. So `api.{network}.nexus.xyz` is wrong for exactly one network — the real-funds one — and it resolves fine in dev, staging and testnet. The failure only ever shows up on the environment you cannot rehearse. Use the explicit map with mainnet as a named case; never interpolate.

**Credentials do not cross networks.** Session tokens, HMAC API keys, and agent keys are minted per network and are invalid on any other, so a key that leaks or is misconfigured on testnet cannot sign for real funds. Switching network means switching credentials. The EIP-712 signing domain is network-scoped too, which is what makes an action signed for one network invalid on the other — never replay a signature, nonce, or agent registration across networks.

**There is no default network, and no blanket redirect.** Select one explicitly. `exchange.nexus.xyz` serves **testnet**, so when it retires its traffic belongs on `api.testnet.nexus.xyz` — never on the bare `api.nexus.xyz`. Redirecting the legacy host at mainnet would silently move play-funds clients onto real funds.

**Changing base changes what you sign.** The request path is part of the [HMAC canonical string](#4-sign-requests). Migration is two independent steps, in order:

1. base path, same host: `https://exchange.nexus.xyz/api/exchange/orders` → `https://exchange.nexus.xyz/api/v1/orders`
2. host: `https://exchange.nexus.xyz/api/v1/orders` → `https://api.testnet.nexus.xyz/v1/orders`

Each step changes the signed path (`/api/exchange/orders` → `/api/v1/orders` → `/v1/orders`). Repointing the base URL without updating the signed path yields `401`, not a routing error. Note the new hosts use `/v1`, not `/api/v1`: the `api.` prefix moved into the hostname rather than being repeated in the path.

### Discovering targets at runtime

Rather than hardcoding the table above, read [`/metadata`](#api-version-support) — it publishes the network the host serves, the per-network REST and WebSocket targets, and the EIP-712 signing domain. The `Metadata` schema in the spec documents the payload.

## Versioning

Pre-1.0 semver — the minor version increments on breaking changes:

| Version bump | Meaning |
|---|---|
| `0.X.0` | Breaking change — removed endpoint, renamed field, new required parameter |
| `0.X.Y` | New feature or backwards-compatible fix |

## Authentication

All endpoints require HMAC-SHA256 request signing, including market data.

### 1. Get a session token (EIP-191)

```bash
# Sign "Sign in to Nexus Exchange" with your wallet, then:
curl -X POST https://exchange.nexus.xyz/api/exchange/auth/login \
  -H 'Content-Type: application/json' \
  -d '{"message": "Sign in to Nexus Exchange", "signature": "0x..."}'
# → {"token": "sess_..."}
```

### 2. Create an API key

```bash
curl -X POST https://exchange.nexus.xyz/api/exchange/keys \
  -H 'Authorization: Bearer sess_...' \
  -H 'Content-Type: application/json' \
  -d '{"label": "my-bot"}'
# → {"key_id": "nx_...", "secret": "...hex..."}
# Save the secret — it is shown exactly once.
export KEY_ID="nx_..."
export SECRET="...hex..."
```

The key is scoped to the [network](#networks) whose host you created it on and is
invalid on any other — mint a separate key per network rather than reusing one.
`POST /keys` takes no network parameter, so the host you call it on *is* the
binding decision. On any other host the key is refused with the same opaque
`401` as a key that never existed, deliberately, so a key id cannot be probed
across networks: a `401` right after a base-URL change means the credential and
the host disagree, and must never be retried against the other network. Listing
and revocation are scoped the same way — `GET /keys` shows only this host's keys,
and a `404` from `DELETE /keys/{key_id}` is not proof that a key is gone. Revoke
on the host that minted it.

### 3. Deposit collateral

Before placing orders you need a balance. On testnet the deposit endpoint
acts as a faucet — the USDX is synthetic and no real funds are required. On
mainnet there is no faucet: collateral is real USDX bridged from Ethereum
Mainnet. See [Networks](#networks).

```bash
TS=$(date +%s%3N)
BODY='{"amount":"100"}'
BODY_HASH=$(printf '%s' "$BODY" | openssl dgst -sha256 | awk '{print $2}')
SIG=$(printf '%s\nPOST\n/api/exchange/account/deposit\n\n%s' "$TS" "$BODY_HASH" \
  | openssl dgst -sha256 -hmac "$SECRET" | awk '{print $2}')

curl -X POST https://exchange.nexus.xyz/api/exchange/account/deposit \
  -H "X-API-Key: $KEY_ID" \
  -H "X-Timestamp: $TS" \
  -H "X-Signature: $SIG" \
  -H 'Content-Type: application/json' \
  -d "$BODY"
# → 200 OK
```

### 4. Sign requests

Every authenticated request needs three headers. Build a canonical string
and HMAC-SHA256 it with your secret. The canonical format is five
newline-separated fields with no trailing newline:

```
{unix_ms}\n{METHOD}\n{path}\n{query}\n{sha256_hex(body)}
```

**Important:** use `printf '%s'` (not `echo`) when piping to OpenSSL — `echo`
appends a trailing newline that shifts the hash and causes a 401.

```bash
TS=$(date +%s%3N)
BODY='{"market_id":"BTC-USDX-PERP","side":"Buy","order_type":"Limit","price":"84000","quantity":"0.01","time_in_force":"GTC"}'
BODY_HASH=$(printf '%s' "$BODY" | openssl dgst -sha256 | awk '{print $2}')
SIG=$(printf '%s\nPOST\n/api/exchange/orders\n\n%s' "$TS" "$BODY_HASH" \
  | openssl dgst -sha256 -hmac "$SECRET" | awk '{print $2}')

curl -X POST https://exchange.nexus.xyz/api/exchange/orders \
  -H "X-API-Key: $KEY_ID" \
  -H "X-Timestamp: $TS" \
  -H "X-Signature: $SIG" \
  -H 'Content-Type: application/json' \
  -d "$BODY"
```

## Request conventions

Every official client (SDK, CLI, MCP server) sends two advisory headers on **every** request:

| Header | Value | Purpose |
|---|---|---|
| `X-Nexus-Api-Version` | released spec tag it was compiled against, e.g. `v0.7.0` (`vMAJOR.MINOR.PATCH`, matching the client's `.api-version`) | attribute traffic to a spec version; future compatibility handling |
| `User-Agent` | `nexus-exchange-<lang>/<version>`, e.g. `nexus-exchange-rs/0.5.1` | per-client usage metering |

Both are **advisory**: the server accepts requests when they are missing, malformed, or name an unknown tag, and never uses them for authentication, authorization, or routing. They are **not** covered by the HMAC signature (they sit outside the [canonical string](#4-sign-requests)), so they are unauthenticated and are used for observability and usage metering only, never for access control. The one case where `X-Nexus-Api-Version` affects a response is the [API version support](#api-version-support) policy below, which may return `426` for a recognized tag below the published minimum — a compatibility gate, not a security boundary.

Clients derive `X-Nexus-Api-Version` from their existing `.api-version` pin — the same pin the drift checks enforce — so the header always reflects the exact contract the client was built against.

## API version support

The version identifier is the released spec tag of this repository — `vMAJOR.MINOR.PATCH`, the same tag every official SDK pins to (drift-gated against its `.api-version`) and reports in [`X-Nexus-Api-Version`](#request-conventions).

The exchange publishes the versions it supports — and the [network](#networks) targets it serves — at the **`/metadata`** endpoint, served by the edge at each network's own host rather than as an operation in this spec. A client, or an autonomous agent, can discover both programmatically instead of scraping docs:

```json
{
  "current_api_version": "v0.7.1",
  "min_api_version": "v0.6.0",
  "network": "testnet",
  "ws_url": "wss://api.testnet.nexus.xyz",
  "signing_domain": { "name": "Nexus Exchange", "version": "1", "chain_id": null },
  "networks": {
    "testnet": { "network": "testnet", "rest_base": "https://api.testnet.nexus.xyz/v1", "funds": "play" },
    "mainnet": { "network": "mainnet", "rest_base": "https://api.nexus.xyz/v1", "funds": "real" }
  }
}
```

| Field | Meaning |
|---|---|
| `current_api_version` | latest released spec tag the edge serves |
| `min_api_version` | oldest spec tag still accepted; a client pinned below this must upgrade |
| `network` | the [network](#networks) this host serves — the one your credentials must belong to |
| `ws_url` | WebSocket origin for that network (`…/stream` public, `…/ws?token=…` authenticated) |
| `signing_domain` | network-scoped EIP-712 domain; `chain_id` is `null` when the edge does not publish it |
| `networks` | every network the edge knows about, keyed by identifier — resolve a sibling target without a hardcoded host map |

`/metadata` is the machine-readable source of truth; the values above are illustrative. Only the two version fields are guaranteed — an older edge may omit the rest, in which case fall back to the spec's `x-nexus-networks` map.

Two values carry a fail-safe rule. A `network` you do not recognize must **not** be assumed to be play funds: treat it as real funds and confirm before acting. And a `signing_domain.chain_id` that is `null` or absent means *the edge did not publish it* — it does not mean `0`, and it is not a cue to fall back to a cached or default value. A client that cannot obtain a `chain_id` should refuse to sign, because the wrong domain either fails verification or produces a signature valid on a network you did not intend. The `Metadata`, `NetworkTarget`, and `SigningDomain` schemas in the spec document the full payload.

### Pre-1.0 support window

While the API is pre-1.0 (`v0.x.y`), breaking changes are frequent and `min_api_version` may advance with any breaking release. The support window is intentionally short: a released tag stays at or above `min_api_version` for **at least 14 days** after the release that supersedes it. This window widens substantially after 1.0 (GA).

Pin your client to a released tag and either poll `/metadata` or watch [releases](https://github.com/nexus-xyz/nexus-exchange-api/releases), so you can upgrade before your pin falls below the minimum.

### Below-minimum requests

A request whose `X-Nexus-Api-Version` names a recognized tag older than `min_api_version` receives a machine-readable **`426 Upgrade Required`** with the structured error code `api_version_unsupported` and a link to the current spec — enough for tooling (and self-healing agents) to detect the skew, fetch the current spec, and upgrade. A missing, malformed, or unknown version header is treated as an unknown/legacy client and is **not** blocked by this policy (see [Request conventions](#request-conventions)).

Because `X-Nexus-Api-Version` is [unauthenticated](#request-conventions), this gate is a compatibility courtesy, not a security control: a caller can spoof any version, but doing so only relaxes the version check — it never grants access that HMAC authentication wouldn't already allow. Authentication and authorization never depend on it.

## WebSocket

```bash
# 1. Mint a token (server-signed, 60s TTL)
TOKEN=$(curl -s -X POST https://exchange.nexus.xyz/api/exchange/ws/token \
  -H "X-API-Key: $KEY_ID" -H "X-Timestamp: $TS" -H "X-Signature: $SIG" \
  | jq -r .token)

# 2. Connect
wscat -c "wss://exchange.nexus.xyz/api/exchange/ws?token=$TOKEN"

# 3. Subscribe
{"op": "subscribe", "channel": "orders"}
{"op": "subscribe", "channel": "book", "market": "BTC-USDX-PERP"}
```

The example uses the current base. On a per-network host the WebSocket origin is the host itself — `wss://api.testnet.nexus.xyz/ws?token=…` (market data at `/stream`) — with no `/api/exchange` prefix. The token must be minted on the **same** network you connect to; tokens are network-scoped like every other credential, so one minted on testnet will not upgrade a mainnet connection. See [Networks](#networks).

## Rate limits

The budget is **weight per second, not requests per second**. Most calls cost one
unit; heavy aggregate reads (`/account/summary`, `/fills`, `/orders/history`,
`/account/portfolio-history`) cost 5; a batch order submit costs
`1 + floor(order_count / 40)`. A Pro key at 20/s gets 20 ticker reads per second
— or 4 `/fills` reads. Operations that cost more carry
`x-nexus-rate-limit-weight` in the spec; absence means 1.

Three independent budgets. Spending one does not spend the others:

| Tier | Requests | Trading actions | WS conns | WS subs | WS frames |
|---|---|---|---|---|---|
| `Pro` | 20/s | 20/s | 5 | 50 | 10/s |
| `MarketMaker` | 2,000/s | 2,000/s | 100 | 1,000 | 50/s |
| `Unlimited` (gateway keys) | per-IP, 50/s | per-IP, 50/s (same bucket as reads) | exempt | exempt | exempt |

Order writes (`POST`/`PATCH`/`DELETE` under `/orders` — including
`POST /orders/preview`) are charged to the trading bucket *instead of* the
request bucket, so a healthy `x-ratelimit-remaining` says nothing about your
order-placement headroom. That independence does not apply to `Unlimited`, whose
reads and order writes share one per-IP bucket; its WS exemptions are from the
per-account ceilings only, and a per-IP connection cap still binds.

Headers: `x-ratelimit-limit` and `x-ratelimit-remaining` on every authenticated
response; `x-ratelimit-reset` and `retry-after` on a `429` only. Note the units
differ — `remaining` counts weight-1 requests, `retry-after` is weighted.
Exceeded → `429 Too Many Requests`, body `{"code": "RATE_LIMIT_EXCEEDED", ...}`,
retryable. `GET /account/rate-limit` reports the request budget and is free to
poll. Ceilings are per-deployment configuration, not contract.

## Markets

32 perpetual futures markets including BTC, ETH, SOL, and traditional assets (GOLD, SPX, NDQ, EUR, GBP, JPY). Full list: `GET /markets/summary`.

## License

Dual-licensed under [MIT](./LICENSE-MIT) or [Apache-2.0](./LICENSE-APACHE), at
your option — same as the Nexus Exchange SDKs.
