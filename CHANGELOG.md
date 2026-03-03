## 1.2.35
- Security: hardened `scripts/optional/setup_clawhub_oauth.sh` to fail closed by default.
- Security: remote credential bootstrap now requires explicit opt-in (`clawhub_oauth_allow_remote_fetch=true`) and a valid pinned `clawhub_credentials_sha256` match before writing `credentials.json`.
- Security: added strict payload shape checks and response size limits for clawhub OAuth credential fetch.
- Security: `credentials.json` is written with restrictive file permissions (`0600`).
- Docs/Config: updated `config.example.json`, `scripts/setup.sh`, and `SECURITY.md` to reflect the SHA-256 pin workflow and correct optional script path.

## 1.2.34
- Security: voice command routing now uses an explicit in-process allowlist dispatcher for user intents, removing subprocess execution for routed commands.
- Security: stricter input validation rejects flag-like capture tokens (for example `--dry-run`) to reduce argument-injection risk.
- Fix: policy voice intents now call policy parsing/storage directly in-process.

## 1.2.33
- Bug: `_iso_to_ts()` returned 0 on parse failure — action events no longer fire at Unix epoch; events with unparseable times are skipped with a warning log.
- Bug: Quiet hours (configured in `config.json`) now enforced in daemon — non-critical notifications are suppressed during configured windows. Conflict alerts remain critical and are always delivered.
- Bug: State file write is now atomic (temp file + `os.replace()`) — prevents `daemon_state.json` corruption on crash or kill signal.
- Bug: `notified_events` cleanup is now TTL-based (48h expiry) instead of a 200-key size cap — old entries no longer crowd out new ones after heavy use.
- Fix: CalDAV event parse errors now log the event UID and error reason instead of silently dropping the event.
- Fix: `config_wizard.py` `DEFAULT_CONFIG` feature flags aligned to match `config.example.json` — all features default `False` (opt-in), not `True`.
- Feature: `event_relink_tolerance_sec` configurable via `config.json` (default: 300s). Controls how far a moved recurring event can shift and still be auto-relinked.
- Feature: CalDAV server path configurable via `nextcloud.caldav_path` in `config.json` (default: `/remote.php/dav`). Supports non-Nextcloud CalDAV servers.

## 1.2.31
- Security: Hardened voice command routing to avoid argument/command injection (no string splitting; suspicious metacharacters rejected; captured text passed as single args).
- Docs: Updated SKILL.md description and diagrams.

# Changelog — Proactive Claw

## 1.2.30
- Security hardening: default autonomy is **confirm** even when config is missing (fail-closed defaults).
- Updated SKILL.md: clearer proactive loop (calendar ⇄ engine ⇄ chat), stronger scenarios, and local-first positioning.

## 1.2.29
- Safe defaults: `config_wizard.py --defaults` generates `max_autonomy_level=confirm`.
- Autonomous mode requires explicit opt-in with `--i-accept-risk`.
