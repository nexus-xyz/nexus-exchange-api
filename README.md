# Nexus Exchange API

[![API Version](https://img.shields.io/github/v/release/nexus-xyz/nexus-exchange-api?label=api)](https://github.com/nexus-xyz/nexus-exchange-api/releases)

REST and WebSocket API for [Nexus Exchange](https://exchange.nexus.xyz) — a perpetual futures exchange for crypto, equities, FX, and commodities.

- **Base URL:** `https://exchange.nexus.xyz/api/exchange`
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

All write endpoints require HMAC-SHA256 request signing. Read endpoints on public market data are unauthenticated.

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
```

### 3. Sign requests

Every authenticated request needs three headers. Build a canonical string, HMAC-SHA256 it with your secret:

```
{unix_ms}\n{METHOD}\n{path}\n{query}\n{sha256(body)}
```

```bash
TS=$(date +%s%3N)
BODY='{"market_id":"BTC-USDX-PERP","side":"Buy","order_type":"Limit","price":"84000","quantity":"0.01","time_in_force":"GTC"}'
BODY_HASH=$(echo -n "$BODY" | openssl dgst -sha256 | awk '{print $2}')
CANONICAL="${TS}\nPOST\n/orders\n\n${BODY_HASH}"
SIG=$(echo -e "$CANONICAL" | openssl dgst -sha256 -hmac "$SECRET" | awk '{print $2}')

curl -X POST https://exchange.nexus.xyz/api/exchange/orders \
  -H "X-API-Key: $KEY_ID" \
  -H "X-Timestamp: $TS" \
  -H "X-Signature: $SIG" \
  -H 'Content-Type: application/json' \
  -d "$BODY"
```

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
