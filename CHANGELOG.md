# Changelog

All notable changes to the Nexus Exchange API are documented here.

Pre-1.0 versioning convention:
- `0.X.0` — breaking change (removed endpoint, renamed field, new required parameter)
- `0.X.Y` — new feature or backwards-compatible fix

---

## [0.3.3](https://github.com/nexus-xyz/nexus-exchange-api/compare/v0.3.1...v0.3.3) (2026-06-15)

### Added

- `GET /account/rate-limit` — query the caller's current rate-limit tier, ceiling, remaining requests, and reset timestamp without consuming a token ([#4](https://github.com/nexus-xyz/nexus-exchange-api/pull/4))
- `POST /account/credit` — claim synthetic USDX against a per-API-key daily allowance (default 500 USDX/day, resets at midnight UTC). Omit `amount` to claim the full remaining allowance. Documented since the Gate 2 release but missing from the spec.
- `POST /orders/batch` description — sequential, non-atomic processing; per-order results preserve request order
- `POST /auth/login` — documented the 24-hour session token expiry

### Fixed

- `GET /ws` protocol documentation now matches the implementation: messages are JSON envelopes tagged with an `op` field (`subscribe` / `unsubscribe` client ops; `subscribed`, `unsubscribed`, `event`, `out_of_sync`, `error` server ops), one channel per subscribe with an optional `market` field, and `since` / `seq_at_join` reconnect cursors. The previously documented untagged `{"subscribe": [...]}` array format belongs to the legacy `GET /stream` endpoint (now documented there) and is rejected by `/ws`.
- `GET /ws` channel list corrected: `trades`, `book`, `candles` (public, per-market) and `orders`, `fills`, `positions`, `balances` (per-account). `stats` is a `/stream`-only channel.

## [0.3.1] — 2026-06-05

### Added

- `GET /keys` — list API keys associated with the authenticated session
- `DELETE /keys/{key_id}` — revoke an API key by ID
- `POST /ws/token` — mint a short-lived WebSocket authentication token
- Per-IP WebSocket connection limit (max 5 simultaneous connections)
- `GET /markets/{market_id}/funding-samples` — dense intra-hour funding rate samples for chart rendering
- Early-access gate: wallets not in the allowlist receive `403 early_access_required` on write endpoints

### Fixed

- Rate limit headers (`x-ratelimit-limit`) now correctly report the IP rate for gateway-key (Unlimited tier) requests rather than `u32::MAX`
- `POST /deposits` route corrected (was incorrectly documented as `POST /account/deposit`)
- API error responses standardised to `{"code": "...", "message": "..."}` across all endpoints

## [0.3.0] — 2026-05-01

Initial public API release.

### Endpoints

- Authentication: `POST /auth/login` (EIP-191), `POST /keys`, `GET /keys`, `DELETE /keys/{key_id}`
- Markets: `GET /markets/summary`, `GET /markets/{id}/ticker`, `GET /markets/{id}/orderbook`, `GET /markets/{id}/trades`, `GET /markets/{id}/candles`, `GET /markets/{id}/funding`, `GET /markets/{id}/mark-price`
- Account: `GET /account`, `GET /positions`, `GET /orders`, `GET /fills`, `GET /balances`
- Trading: `POST /orders`, `DELETE /orders/{id}`, `POST /orders/batch`, `POST /account/deposit`, `POST /account/withdraw`
- WebSocket: `GET /ws` (per-account stream), `POST /ws/token`
