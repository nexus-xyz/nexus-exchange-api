# Changelog

All notable changes to the Nexus Exchange API are documented here.

Pre-1.0 versioning convention:
- `0.X.0` — breaking change (removed endpoint, renamed field, new required parameter)
- `0.X.Y` — new feature or backwards-compatible fix

---

## [0.5.0](https://github.com/nexus-xyz/nexus-exchange-api/compare/v0.4.0...v0.5.0) (2026-06-25)


### ⚠ BREAKING CHANGES

* adding `format: int64` to the halted_at, expires_at, and nonce integer fields is classified as a type/format change by oasdiff. This is a deliberate, value-safe tightening (the fields always carried ms-epoch values); no wire-format change. Minor version bump per the api-diff workflow.

### Features

* add PATCH /orders/{order_id} (amend) to spec ([#24](https://github.com/nexus-xyz/nexus-exchange-api/issues/24)) ([0679217](https://github.com/nexus-xyz/nexus-exchange-api/commit/067921769d42d7f68addc4172bc822b0bcc64f42))
* pin POST /orders/batch per-order result schema (ENG-4199) ([#32](https://github.com/nexus-xyz/nexus-exchange-api/issues/32)) ([8eca98f](https://github.com/nexus-xyz/nexus-exchange-api/commit/8eca98ffecc1c0a811a7d25a57976807bbf19486))
* reconcile spec with live indexer routes (ENG-3821, ENG-3822, ENG-3823) ([#30](https://github.com/nexus-xyz/nexus-exchange-api/issues/30)) ([784f23b](https://github.com/nexus-xyz/nexus-exchange-api/commit/784f23b9618a30994ec0eaa3bba8c66a338a92d1))
* type weak schemas before SDK codegen (ENG-3543) ([#19](https://github.com/nexus-xyz/nexus-exchange-api/issues/19)) ([701201b](https://github.com/nexus-xyz/nexus-exchange-api/commit/701201bddb7a64497438e192f70d735e78e09dbb))

## [0.4.0](https://github.com/nexus-xyz/nexus-exchange-api/compare/v0.3.5...v0.4.0) (2026-06-23)


### ⚠ BREAKING CHANGES

* **markets:** /markets/summary responses use `last_trade_price` instead of `mark_price`; clients reading `mark_price` will now get undefined.

### Features

* **markets:** rename MarketSummary.mark_price → last_trade_price ([#20](https://github.com/nexus-xyz/nexus-exchange-api/issues/20)) ([058e336](https://github.com/nexus-xyz/nexus-exchange-api/commit/058e33634103290453c807caf8743794013c9c7e))

## [0.3.5](https://github.com/nexus-xyz/nexus-exchange-api/compare/v0.3.4...v0.3.5) (2026-06-16)


### Features

* type scalar schemas and add operationIds for SDK codegen ([#17](https://github.com/nexus-xyz/nexus-exchange-api/issues/17)) ([c145733](https://github.com/nexus-xyz/nexus-exchange-api/commit/c14573374e4110016765f96de880fc425256c8ea))

## [0.3.4](https://github.com/nexus-xyz/nexus-exchange-api/compare/v0.3.3...v0.3.4) (2026-06-16)


### Bug Fixes

* add required response descriptions to all 200 responses ([#8](https://github.com/nexus-xyz/nexus-exchange-api/issues/8)) ([dc3fe8b](https://github.com/nexus-xyz/nexus-exchange-api/commit/dc3fe8bb9674a6c81c962b2431d06becd834c16a))
* **docs:** correct HMAC example, add deposit step, fix auth claim, fix servers URL ([#1](https://github.com/nexus-xyz/nexus-exchange-api/issues/1)) ([7af831c](https://github.com/nexus-xyz/nexus-exchange-api/commit/7af831c860cd3469a88f9ed5e4db6c39d87e2d50))
* use OpenAPI 3.1 null type instead of 3.0 nullable ([#12](https://github.com/nexus-xyz/nexus-exchange-api/issues/12)) ([3f4d6d0](https://github.com/nexus-xyz/nexus-exchange-api/commit/3f4d6d0d54707f1395709628c1fd954ab9c95a9f))

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
