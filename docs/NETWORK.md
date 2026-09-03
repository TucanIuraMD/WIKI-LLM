# Network Overview

The terminal uses two outbound network channels plus an optional inbound UI
endpoint.

| Direction | Port | Protocol | Purpose |
|-----------|------|----------|---------|
| out | 333 | TCP | monitoring (see `MONITORING.md`) |
| out | 14111 | TCP/HTTP | payment (see `PAYMENT.md`) |
| out | 14444 | HTTP proxy (optional) | optional egress proxy |
| in (optional) | 8585 | WebSocket | kiosk UI |
| in | 3306 | TCP | MySQL (`127.0.0.1`) |

## Settings that drive connectivity

Stored in MySQL `settings` (see `DATABASE.md`, `DEPLOYMENT.md`):

- `MonitorURL` + `MonitorPort` — monitoring server endpoint.
- `PaymentUri` — payment server base URI.
- `UseProxy` / `ProxyName` / `ProxyPort` — optional HTTP proxy for payment
  channel.
- `HttpTimeOut` — HTTP timeout.

## Channel separation

- Monitoring is a persistent raw-TCP client (`Protocol.dll`) — details in
  `MONITORING.md`.
- Payment is HTTP POST of signed binary frames (`SinkLib.dll`) — details in
  `PAYMENT.md`.
- Proxy applies to the HTTP payment client when enabled.

## Diagnostics

- Verify listeners: `ss -tlnp | grep -E ':333|:14111|:3306|:8585'`.
- On Linux, log directory names may literally contain backslashes (runtime
  quirk); locate logs accordingly (`TROUBLESHOOTING.md`).
