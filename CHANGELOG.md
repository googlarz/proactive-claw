# Changelog — proactive-agent

## [1.0.0] — 2025-02-19

Initial release on clawhub.ai.

### Features

- **Conversation Radar** — Scores every exchange 0–10 and asks once, briefly, when a calendar entry is worth creating
- **Calendar Monitoring + Conflict Detection** — Scans upcoming events, detects overlaps, overloaded days, back-to-back runs
- **Background Daemon** — Runs every 15 min via launchd (macOS) or systemd (Linux); sends system and Telegram push notifications before you open OpenClaw
- **SQLite Memory + TF-IDF Search** — All event outcomes stored in `memory.db`; semantic search, pattern analysis, open action items, quarterly summaries
- **Cross-Skill Intelligence** — Pulls live GitHub PRs/issues and Notion pages at prep time; enriches agenda automatically
- **Natural Language Rules Engine** — "Never bother me about standups", "Always prep 2 days before board" — parsed and applied automatically
- **Post-Event Intelligence Loop** — Weekly Monday digest, stale action item follow-ups, auto-schedule calendar events for open items
- **Autonomous Calendar Policy Engine** — "Block 1 hour prep before board meetings" — creates events automatically based on policies you state in plain English
- **Multi-Agent Orchestration** — Full pre-event pipeline: action items → cross-skill context → patterns → prep block → email draft → Notion notes → pending nudge
- **Predictive Energy Scheduling** — Learns when you perform best from sentiment history; warns before scheduling high-stakes events at low-energy times; suggests focus blocks
- **Natural Language Calendar Editing** — Move events, find free time, clear windows, read calendar in plain English ("move sprint review to next Monday 2pm")
- **Relationship Memory** — Lightweight CRM built automatically from calendar attendees and outcome notes; surfaces interaction history at prep time
- **Voice-First Interaction** — Transcribes audio via Whisper skill (or openai-whisper package); routes voice commands to the right script
- **Adaptive Notification Intelligence** — Learns which channels and times you respond to; self-tunes per event type after 5+ samples
- **Team Awareness** — Opt-in cross-calendar coordination; find slots when everyone is free; suggest meeting times for named attendees

### Calendar backends

- Google Calendar (native API, no gcalcli dependency)
- Nextcloud CalDAV

### Setup paths

- **clawhub OAuth** — one-token setup, no Google Cloud Console required
- **Manual Google credentials** — standard OAuth desktop flow
- **Nextcloud** — URL + app-password

### Platform support

- macOS (launchd daemon)
- Linux (systemd user timer)
- Python 3.8+
