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

Both are **advisory**: the server accepts requests when they are missing, malformed, or name an unknown tag — it never rejects or routes on them. They are **not** covered by the HMAC signature (they sit outside the [canonical string](#4-sign-requests)), so they are unauthenticated and are used for observability and usage metering only, never for authentication or access control.

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
