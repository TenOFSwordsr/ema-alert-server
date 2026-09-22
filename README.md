# EMA Alert Server & Monitor

Go rewrite of the EMA alert relay that sits behind the MetaTrader 5 Expert Advisor.
The MT5 EA POSTs moving-average cross events to it; it stores them in SQLite and fans
them out to Telegram and Iranian SMS panels. It also mirrors live chart data pushed by
the EA so other bots can read the market over HTTP. A companion Windows desktop app
exposes that state read-only. Two binaries, one module (`ema-alert`).

**Suggested repo name:** `ema-alert-server`
**Stack:** Go 1.26, gin, lxn/walk (Windows UI), modernc.org/sqlite, godotenv, lumberjack
**Status:** active
**Last modified:** 2026-08-24

## What it does

- `cmd/server` - the HTTP service (binds `HOST:PORT`, 5000 by default):
  - `POST /alert` ingests an EA cross event, `GET /alerts` reads history with delivery statuses.
  - `POST /market/ingest`, `GET /market`, `GET /market/candles` serve live chart snapshots and OHLC.
  - `GET /health` unauthenticated liveness; everything else needs `Authorization: Bearer <key>`.
  - Multiple named API keys in `api_keys.json`, hot-reloaded; per-IP rate limit (240/min).
  - Async dispatcher (`internal/notify`): worker queue decouples the 201 response from delivery.
    Telegram with multiple bot tokens as failover transports, SMS via Kavenegar
    (`send` or `lookup` template) or Twilio.
  - `internal/market` runs a file watcher that picks up `ema_ticker_*.json` written by the EA
    into the terminal's common files folder - the EA cannot use `WebRequest` from tick context.
- `cmd/monitor` - read-only dark-themed Windows desktop UI (walk): live alerts, streaming
  server log with level filters, API key generate/enable/disable/delete, health. It mutates
  nothing on the trading side.
- `Docs/API.md` - full HTTP contract aimed at third-party trader bots (Python, MQL, C#, Node).
- `deploy/` - `.env.production`, an `EMAAlertEA_M5.set` chart preset, `test_payload.json`, and
  `wg_setup.ps1` / `wg_probe.ps1` / `wg_search.ps1` / `wg_restore.ps1` to install and tear down a
  WireGuard tunnel so bearer keys do not cross the open internet in cleartext.
- `mt5-keepalive.cmd` - scheduled task script: relaunches `terminal64.exe` when it is gone, and
  backs up / restores MT5 chart profiles when the EA's ticker file goes stale for >10 minutes.

## Layout

```
cmd/server/main.go      HTTP service entrypoint
cmd/monitor/main.go     Windows tray/desktop UI (+ manifest.xml, .syso, Terminal.ico)
internal/config         env loading (.env.example documents every key)
internal/keys           multi-key store, hot reload
internal/server         gin router, auth, rate limit, handlers
internal/notify         dispatcher, telegram, sms, templates
internal/market         in-memory snapshots + ema_ticker file watcher
internal/store          SQLite alert history
internal/model          alert types (tested)
Docs/API.md             HTTP documentation
deploy/                 production env, EA preset, WireGuard scripts
```

## Running it

```bash
cp .env.example .env      # fill in API_KEY, Telegram/SMS credentials
go run ./cmd/server       # or: go build -o ema-alert.exe ./cmd/server
go build -o ema-monitor.exe ./cmd/monitor    # needs cmd/monitor/manifest.xml (walk)
go test ./... -race -coverprofile=coverage.txt
```

## Notes

- `deploy/.env.production` contains live values (server API key, Kavenegar key, Telegram bot
  tokens, recipient phone). Scrub or move to a secret store before publishing.
- `cmd/monitor/main.go` defaults `DATABASE_PATH`, `LOG_FILE`, `KEYS_FILE` to absolute
  `C:/Users/Administrator/Desktop/...` paths; override via env for any other machine.
- `ema-alert.exe` and `ema-monitor.exe` are committed build output; `internal/market` and the
  DB path are shared with the legacy Python service so history carries over during rollback.
- `patch_parser.py` and `patch_ui.py` are one-off source rewriters that patch
  `cmd/monitor/main.go` by string replacement. They are dev scaffolding, not part of the build.
- `.github/workflows/ci.yml` runs `go mod tidy` diff check, vet, race tests and govulncheck.
- Related: `ema-alert-ea` (the MQL5 EA that feeds this) and `ema-alert-watchdog` (backup detector).
