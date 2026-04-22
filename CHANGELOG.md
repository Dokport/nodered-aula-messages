# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [1.1.0] - 2026-04-22

### Changed

- **Architecture overhaul**: replaced direct OIDC token refresh with Home Assistant REST API proxy
  - Removed: Token Manager node (OIDC refresh flow)
  - Removed: `AULA_REFRESH_TOKEN` environment variable
  - Added: **HA API Fetcher** node — calls `aula.api_call` service via HA REST API
  - Added: `HA_URL` and `HA_TOKEN` environment variables
- Flow now delegates authentication entirely to Scaarup's HA integration; Node-RED only
  needs a HA Long-Lived Access Token, not any AULA session material
- Updated README setup instructions and troubleshooting guide accordingly

### Fixed

- Token approach (v1.0.0) did not work: Scaarup's integration stores the AULA session
  in memory, not in `core.config_entries` — there is no refresh_token on disk to extract

---

## [1.0.0] - 2026-04-22

### Added

- Initial release
- Node-RED flow (`flows.json`) that polls the AULA messaging API and publishes via MQTT
- **Token Manager** function node: OIDC token refresh using `AULA_REFRESH_TOKEN` env var;
  auto-refreshes before expiry, stores updated tokens in flow context
- **AULA Poller** function node: paginated fetch of all message threads
  (`messaging.getThreads`) and per-thread messages (`messaging.getMessagesForThread`)
  using the AULA REST API v22
- **MQTT Formatter** function node: publishes per-thread topics, a summary topic,
  a status/heartbeat topic, and an unread-only topic
- MQTT LWT (Last Will and Testament) messages on the bridge state topic
- Catch node for centralised error logging in the Node-RED debug panel
- `package.json` prepared for future npm publishing
- `README.md` in Danish and English with full setup instructions, MQTT output
  examples, configuration options, and troubleshooting guide
- `CHANGELOG.md` (this file)

### Notes

- Requires Node.js ≥ 18 (for native `fetch` API)
- Requires [Scaarup's HA AULA integration](https://github.com/scaarup/aula) to be
  installed and authenticated for the initial refresh token
- Credentials never stored in `flows.json` — the `AULA_REFRESH_TOKEN` is read
  exclusively from environment variables; subsequent tokens live in Node-RED flow
  context (volatile, not persisted to disk)
