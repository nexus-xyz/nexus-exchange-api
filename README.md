# Nexus Exchange API

[![API Version](https://img.shields.io/github/v/release/nexus-xyz/nexus-exchange-api?label=api)](https://github.com/nexus-xyz/nexus-exchange-api/releases)
[![License](https://img.shields.io/badge/license-MIT%2FApache--2.0-blue.svg)](#license)

REST and WebSocket API for [Nexus Exchange](https://exchange.nexus.xyz) — a perpetual futures exchange for crypto, equities, FX, and commodities.

- **Base URL:** `https://exchange.nexus.xyz/api/exchange`
- **Direct-service base:** `https://exchange.nexus.xyz/api/v1` — the market-data and account/trading surfaces are also served directly by their backend service under an `/api/v1` prefix (routed by the load balancer). Both bases are live in parallel; the same HMAC signing applies, over the full request path (e.g. `/api/v1/orders`).
- **OpenAPI spec:** [`openapi.json`](./openapi.json)
- **Changelog:** [`CHANGELOG.md`](./CHANGELOG.md)
- **Interactive docs:** [exchange.nexus.xyz/api-docs](https://exchange.nexus.xyz/api-docs)

## Versioning

Pre-1.0 semver — the minor version increments on breaking changes:

| Version bump | Meaning |
|---|---|
| `0.X.0` | Breaking change — removed endpoint, renamed field, new required parameter |
| `0.X.Y` | New feature or backwards-compatible fix |

## API version support

The version identifier is the released spec tag of this repository — `vMAJOR.MINOR.PATCH`, the same tag every official SDK pins to (drift-gated against its `.api-version`) and reports in [`X-Nexus-Api-Version`](#request-conventions).

The exchange publishes the versions it supports at the **`/metadata`** endpoint, so a client — or an autonomous agent — can discover them programmatically instead of scraping docs:

```json
{ "current_api_version": "v0.7.1", "min_api_version": "v0.6.0" }
```

| Field | Meaning |
|---|---|
| `current_api_version` | latest released spec tag the edge serves |
| `min_api_version` | oldest spec tag still accepted; a client pinned below this must upgrade |

`/metadata` is the machine-readable source of truth; the values above are illustrative.

### Pre-1.0 support window

While the API is pre-1.0 (`v0.x.y`), breaking changes are frequent and `min_api_version` may advance with any breaking release. The support window is intentionally short: a released tag stays at or above `min_api_version` for **at least 14 days** after the release that supersedes it. This window widens substantially after 1.0 (GA).

Pin your client to a released tag and either poll `/metadata` or watch [releases](https://github.com/nexus-xyz/nexus-exchange-api/releases), so you can upgrade before your pin falls below the minimum.

### Below-minimum requests

A request whose `X-Nexus-Api-Version` names a recognized tag older than `min_api_version` receives a machine-readable **`426 Upgrade Required`** with the structured error code `api_version_unsupported` and a link to the current spec — enough for tooling (and self-healing agents) to detect the skew, fetch the current spec, and upgrade. A missing, malformed, or unknown version header is treated as an unknown/legacy client and is **not** blocked by this policy (see [Request conventions](#request-conventions)).

Because `X-Nexus-Api-Version` is [unauthenticated](#request-conventions), this gate is a compatibility courtesy, not a security control: a caller can spoof any version, but doing so only relaxes the version check — it never grants access that HMAC authentication wouldn't already allow. Authentication and authorization never depend on it.

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

### 3. Deposit collateral

Before placing orders you need a balance. On testnet the deposit endpoint
acts as a faucet — no real funds are required.

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

Both are **advisory**: the server accepts requests when they are missing, malformed, or name an unknown tag, and never uses them for authentication, authorization, or routing. They are **not** covered by the HMAC signature (they sit outside the [canonical string](#4-sign-requests)), so they are unauthenticated and are used for observability and usage metering only, never for access control. The one case where `X-Nexus-Api-Version` affects a response is the [API version support](#api-version-support) policy above, which may return `426` for a recognized tag below the published minimum — a compatibility gate, not a security boundary.

Clients derive `X-Nexus-Api-Version` from their existing `.api-version` pin — the same pin the drift checks enforce — so the header always reflects the exact contract the client was built against.

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

## Rate limits

| Tier | Limit |
|---|---|
| API key (standard) | 20 req/s |
| IP (unauthenticated) | 50 req/s |
| MarketMaker | 2,000 req/s |

Rate limit headers: `x-ratelimit-limit`, `x-ratelimit-remaining`. Exceeded → `429 Too Many Requests` with `Retry-After`.

## Markets

32 perpetual futures markets including BTC, ETH, SOL, and traditional assets (GOLD, SPX, NDQ, EUR, GBP, JPY). Full list: `GET /markets/summary`.

## License

Dual-licensed under [MIT](./LICENSE-MIT) or [Apache-2.0](./LICENSE-APACHE), at
your option — same as the Nexus Exchange SDKs.
