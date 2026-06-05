# Changelog

All notable changes to the Nexus Exchange API are documented here.

Pre-1.0 versioning convention:
- `0.X.0` — breaking change (removed endpoint, renamed field, new required parameter)
- `0.X.Y` — new feature or backwards-compatible fix

---

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
