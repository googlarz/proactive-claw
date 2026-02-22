---
name: proactive-claw
version: 1.2.7
description: >
  🦞 Your AI gets ahead of your calendar — not just reactive, but proactive.

  📅 Reads your calendars + 💬 chat context → plans reminders, prep blocks, buffers & debriefs
  → writes only to its own "Proactive Claw — Actions" calendar. Local-first, non-root daemon
  runs every 15 min. All decisions logged locally. Safe defaults — most features OFF.

  ⚙️ Daemon mode: PLAN→EXECUTE→CLEANUP automatically in background.
  💬 Conversation mode: reads context from chat + calendar, proposes actions, you approve.

  🔒 Governed by max_autonomy_level (advisory/confirm/autonomous). Priority tiers P0–P5,
  quiet hours, cooldowns, explainability mode. 35 features. Local SQLite. No telemetry.

  Requires: python3 + Google OAuth credentials OR Nextcloud app password (via setup.sh).
  Optional (all OFF): Telegram, Notion, GitHub, LLM rater, voice.

tags:
  - calendar
  - productivity
  - automation
  - proactive
  - daemon
  - google-calendar
  - nextcloud
  - local-first

requires:
  bins:
    - python3
    - bash
  env_vars: []
  credentials:
    - Google OAuth credentials.json (via setup.sh — Google Cloud Console or clawhub OAuth flow)
    - OR Nextcloud app-specific password (set in config.json)

install:
  - kind: script
    label: "One-time setup — installs Python deps, creates action calendar, optionally installs daemon"
    command: "bash scripts/setup.sh"

side_effects:
  - Installs a user-level background daemon (launchd on macOS, systemd user timer on Linux) via install_daemon.sh. Runs every 15 min. Does NOT run as root. Uninstall instructions in SKILL.md.
  - Writes local files under ~/.openclaw/workspace/skills/proactive-claw/ only. No files written outside this directory.
  - Creates a "Proactive Claw — Actions" calendar in Google/Nextcloud. Never modifies your existing calendars — reads them only.
  - Maintains a local SQLite link graph (proactive_links.db) connecting your events to planned actions.
  - Outbound HTTPS to Google Calendar API only by default. Notion, Telegram, GitHub, clawhub.ai, LLM rating API all require explicit opt-in via feature_* flags in config.json.
  - pip installs google-api-python-client, google-auth-oauthlib, google-auth-httplib2 (Google) or caldav, icalendar (Nextcloud) during setup.sh.
---

# 🦞 Proactive Claw v1.2.7

> Transform AI agents into governed execution partners that understand your work, monitor your context, and act ahead of you — predictively and under your control.

---

## 🏗️ Architecture

```
┌─────────────────────────┐        ┌─────────────────────────┐
│     YOUR CALENDARS      │        │          CHAT            │
│     (N calendars)       │        │   your conversations    │
│                         │        │   with Claude Code      │
└────────────┬────────────┘        └────────────┬────────────┘
             │ ▲                                │ ▲
    read     │ │ write                  nudges  │ │ proposals
    events   │ │ action events        & context │ │ & approvals
             │ │                                │ │
             ▼ │                                ▼ │
┌──────────────────────────────────────────────────────────────┐
│                   Proactive Claw — Actions                    │
│              (skill-owned calendar, always visible)          │
│         Reminders  ·  Prep blocks  ·  Buffers  ·  Debriefs  │
└──────────────────────────────┬───────────────────────────────┘
                               │
                   ┌───────────▼───────────┐
                   │     Link Graph DB      │
                   │     (SQLite)           │
                   │  user_events           │
                   │  action_events         │
                   │  links · suppression   │
                   │  sent_actions          │
                   └───────────▲───────────┘
                               │ (background, every 15 min)
┌─────────────────────────────────────────────────────────────┐
│                  BACKGROUND DAEMON                           │
│  PLAN → EXECUTE → CLEANUP (user-level, non-root)            │
└─────────────────────────────────────────────────────────────┘
```

**Two independent modes — both governed by `max_autonomy_level`:**

| Mode | Trigger | Autonomy |
|------|---------|----------|
| 💬 **Chat** | You, explicitly per conversation | Capped by `max_autonomy_level` |
| ⚙️ **Daemon** | Background timer, every 15 min | Capped by `max_autonomy_level` |

**Your calendars** are read-only in both modes — never modified. All writes go to the **"Proactive Claw — Actions"** calendar only. Events are linked via a SQLite graph so actions stay in sync when source events move or are deleted.

### ⚙️ Daemon Cycle: PLAN → EXECUTE → CLEANUP

Every 15 minutes (background, after `install_daemon.sh`):

1. **PLAN** — Ingest user events, detect deletions, auto-relink moved events, plan reminder/prep/buffer/debrief actions
2. **EXECUTE** — Fire due actions idempotently (check `sent_actions` table before sending)
3. **CLEANUP** — Once daily: rename paused/canceled events, delete old canceled entries

### 💬 Chat Mode: On-demand, With Your Approval

When chatting with Claude Code, it can call proactive-claw scripts to:

| Action | Script | Effect |
|--------|--------|--------|
| Read your schedule | `scan_calendar.py` | Shows result — no writes |
| Propose a change | `cal_editor.py --dry-run` | You approve before anything changes |
| Log an outcome | `capture_outcome.py` | Only after you confirm the summary |
| Check policies | `policy_engine.py --evaluate --dry-run` | Suggestions only |

With `max_autonomy_level: confirm` (default), Claude Code **always asks before writing**. With `advisory`, it can only suggest — never execute. With `autonomous`, it acts without asking (not recommended).

---

## 🔒 Security & Privacy

### ⚠️ CRITICAL — Read Before Installing

**Step 1 — Review setup scripts:**
```bash
cat ~/.openclaw/workspace/skills/proactive-agent/scripts/setup.sh
cat ~/.openclaw/workspace/skills/proactive-agent/scripts/install_daemon.sh
```
Both are plain shell scripts. Confirm they only write to `~/.openclaw/` and create user-level timers (not root services).

**Step 2 — Start with safe defaults:**
- Run `python3 config_wizard.py` for guided setup — do NOT copy `config.example.json` directly
- Ensure `max_autonomy_level: "confirm"` (not `autonomous`)
- All `feature_*` default to `false` for external services — only enable what you need

**Step 3 — For credentials:**
- **Google**: use standard OAuth desktop flow (`setup.sh` handles this)
- **Nextcloud**: generate an app-specific password — never your account password
- **clawhub_token**: optional; only use if you trust clawhub.ai
- **External APIs** (Telegram, Notion, GitHub, LLM): only provide tokens when enabling the feature

**Step 4 — Test in dry-run mode first:**
```bash
python3 daemon.py --simulate --days 3   # safe preview, no writes
python3 action_planner.py --dry-run     # see what would be planned
python3 action_executor.py --dry-run    # see what would be executed
```

### 🛡️ Security Guarantees

| Guarantee | Detail |
|-----------|--------|
| **Credentials stay local** | `credentials.json`, `token.json`, `config.json` stored only in skill directory. Never uploaded. |
| **User-level daemon only** | Creates user-level timers (launchd/systemd user). Runs as your user — never root. |
| **Calendar writes isolated** | Only writes to the `Proactive Claw — Actions` calendar. All other calendars are read-only. |
| **Network calls gated** | Default: Google Calendar API only. Notion, Telegram, GitHub, clawhub.ai, LLM — all opt-in. |
| **Nextcloud password** | App-specific password only. Generate at `your-nextcloud.com/settings/personal/security`. |
| **Safe-by-default config** | `max_autonomy_level` defaults to `confirm`. All external features default to `false`. |
| **Inspectable scripts** | `setup.sh` and `install_daemon.sh` are plain shell — no obfuscated downloads, no root commands. |
| **clawhub OAuth scope** | `clawhub_token` downloads only the OAuth client definition. Your Google token is generated locally and never sent to clawhub.ai. |
| **LLM rater is local-first** | Defaults to Ollama on `localhost` — no API key, no data sent anywhere. |
| **Link graph is local** | `proactive_links.db` stores only event UIDs, fingerprints, and link metadata. All local SQLite. |

**Uninstall daemon:**
```bash
# macOS
launchctl unload ~/Library/LaunchAgents/ai.openclaw.proactive-agent.plist
rm ~/Library/LaunchAgents/ai.openclaw.proactive-agent.plist

# Linux
systemctl --user disable --now openclaw-proactive-agent.timer
rm ~/.config/systemd/user/openclaw-proactive-agent.*
```

### 📡 What Data Leaves Your Machine

| Service | When | What is sent | Enabled by |
|---------|------|-------------|------------|
| Google Calendar API | Always (core) | Calendar read/write requests with OAuth token | `feature_calendar` |
| clawhub.ai | Setup only, if using clawhub OAuth | `clawhub_token` to fetch `credentials.json` | `clawhub_token` set in config |
| Notion API | Only if enabled | Event title (first 50 chars, read-only) or outcome notes | `feature_cross_skill` + `NOTION_API_KEY` |
| GitHub API | Only if enabled | Read-only: open PRs and issues via `gh` CLI | `feature_cross_skill` + `gh` CLI authenticated |
| Telegram API | Only if enabled | Notification message text | `notification_channels` includes `telegram` |
| Nextcloud CalDAV | Only if using Nextcloud backend | Calendar read/write via CalDAV | `calendar_backend: nextcloud` |
| LLM rating API | Only if enabled AND using cloud backend | Outcome notes + event title + sentiment | `llm_rater.enabled` + non-localhost `base_url` |

> 💡 **Local LLM = zero external calls.** With `base_url: http://localhost:11434/v1` (Ollama) or `http://localhost:1234/v1` (LM Studio), nothing leaves your machine.

**Nothing else.** No analytics, no telemetry, no data sent to the skill author.

---

## ✨ Features at a Glance

### Core Features (v1.0–1.1)

| # | Feature | Description |
|---|---------|-------------|
| 1 | 📡 Conversation Radar | Score 0–10 silently after every exchange |
| 2 | 📅 Calendar Monitoring | Scan + conflict detection + actionable events |
| 3 | ⚙️ Background Daemon | PLAN→EXECUTE→CLEANUP cycles every 15 min |
| 4 | 🧠 SQLite Memory | Outcome history with TF-IDF semantic search |
| 5 | 🔗 Cross-Skill Intelligence | GitHub + Notion context (opt-in) |
| 6 | 📝 Natural Language Rules | User-defined rules engine |
| 7 | 🔄 Post-Event Intelligence | Follow-ups, weekly digest, quarterly insights |
| 8 | 🤖 Calendar Policy Engine | Autonomous prep/focus/buffer/debrief blocking |
| 9 | 🎭 Multi-Agent Orchestration | Full pre-event preparation pipeline |
| 10 | ⚡ Energy Prediction | Predictive energy scheduling with decay weighting |
| 11 | ✏️ Calendar Editing | Move, find free time, clear, read in plain English |
| 12 | 👥 Relationship Memory | Lightweight CRM from attendees + outcomes |
| 13 | 🎙️ Voice-First | Whisper integration + intent routing |
| 14 | 🔔 Adaptive Notifications | Self-tuning channel + time learning with decay |
| 15 | 👫 Team Awareness | Opt-in cross-calendar coordination |
| 16 | 🤖 LLM Interaction Rater | Local model rates check-in quality |

### New in v1.2.0 🆕

| # | Feature | Description |
|---|---------|-------------|
| 17 | 🎯 Proactivity Engine | Unified scoring: energy + notification + policy + relationship |
| 18 | 🚦 Interruption Governance | Priority tiers P0–P5, max nudges/day, cooldowns |
| 19 | 🔍 Explainability Mode | Trace every nudge/policy/energy decision |
| 20 | 🎚️ Proactivity Intensity Dial | `low` / `balanced` / `executive` mode |
| 21 | 🔐 Max Autonomy Cap | `advisory` / `confirm` / `autonomous` global override |
| 22 | ⏳ Memory Decay | Exponential recency weighting across all modules |
| 23 | 🏥 System Health Audit | 7 diagnostic checks: DB, daemon, config, calendar, flags, stale, disk |
| 24 | ⚡ Policy Conflict Detection | Pairwise detection of contradictory policies |
| 25 | 🧙 Config Wizard | Interactive CLI setup with validation |
| 26 | 🔬 Simulation Mode | Dry-run daemon over N future days |
| 27 | 🌙 Quiet Hours | Suppress non-safety nudges during quiet windows |
| 28 | 💾 Data Export/Import | JSON/CSV backup + restore |
| 29 | 📊 Drift Monitoring | Monthly behaviour reports with delta alerts |
| 30 | 📆 2-Calendar Architecture | Read user calendars, write to Action Calendar only |
| 31 | 🕸️ Link Graph | SQLite graph connecting user events ↔ planned actions |
| 32 | 📋 Action Planner | PLAN phase: ingest, detect missing, auto-relink, create actions |
| 33 | ▶️ Action Executor | EXECUTE phase: fire due actions idempotently |
| 34 | 🗑️ Deletion Detection | Fingerprint-based move detection + confirm/suppress workflow |
| 35 | 🦞 Soft-Cancel Policy | Rename canceled events, cleanup after N days |

---

## 🛠️ Setup

### Quick Start

```bash
bash ~/.openclaw/workspace/skills/proactive-agent/scripts/setup.sh
```

### Option A — clawhub OAuth ✨ Recommended

1. Go to **https://clawhub.ai/settings/integrations** → Connect Google Calendar → copy your token
2. In `config.json` set `"clawhub_token": "your-token-here"`
3. Run `setup.sh` — credentials download automatically, no Google Cloud Console needed

### Option B — Manual Google Credentials

1. Go to **https://console.cloud.google.com** → New project → Enable Google Calendar API
2. Create OAuth 2.0 credentials (Desktop app) → download JSON
3. `mv ~/Downloads/credentials.json ~/.openclaw/workspace/skills/proactive-agent/credentials.json`
4. Run `setup.sh`

### Option C — Nextcloud CalDAV

```json
"calendar_backend": "nextcloud",
"nextcloud": {
  "url": "https://your-nextcloud.com",
  "username": "...",
  "password": "app-password"
}
```

> ⚠️ Use a Nextcloud **app-specific password**, not your account password. Generate one at `your-nextcloud.com/settings/personal/security`.

Run `setup.sh` — connects, creates Proactive Claw — Actions calendar, saves URL.

### Install Background Daemon

```bash
bash ~/.openclaw/workspace/skills/proactive-agent/scripts/install_daemon.sh
```

| Platform | Method | Log |
|----------|--------|-----|
| macOS | launchd plist, runs every 15 min | `~/.openclaw/.../daemon.log` |
| Linux | systemd user timer | `~/.openclaw/.../daemon.log` |

### Additional Setup Steps

```bash
# Migrate existing outcomes to SQLite
python3 ~/.openclaw/workspace/skills/proactive-agent/scripts/memory.py --import-outcomes

# Interactive config wizard (optional but recommended)
python3 ~/.openclaw/workspace/skills/proactive-agent/scripts/config_wizard.py
```

---

## ⚙️ Configuration

**File:** `~/.openclaw/workspace/skills/proactive-agent/config.json`

### Core Settings

| Key | Default | Description |
|-----|---------|-------------|
| `calendar_backend` | `"google"` | `google` or `nextcloud` |
| `timezone` | `"UTC"` | IANA tz — e.g. `"Europe/Berlin"` |
| `daemon_interval_minutes` | `15` | How often daemon scans |
| `scan_days_ahead` | `7` | Days to look ahead |
| `user_email` | `""` | Your email (for filtering) |

### Governance Settings

| Key | Default | Description |
|-----|---------|-------------|
| `proactivity_mode` | `"balanced"` | `low` · `balanced` · `executive` |
| `max_autonomy_level` | `"confirm"` | `advisory` · `confirm` · `autonomous` |
| `quiet_hours.weekdays` | `"22:00-07:00"` | Suppress non-safety nudges at night |
| `quiet_hours.weekends` | `"21:00-09:00"` | Suppress non-safety nudges on weekends |
| `max_nudges_per_day` | `12` | Hard cap on daily nudges |
| `nudge_cooldown_minutes` | `30` | Minimum gap after a dismissed nudge |
| `memory_decay_half_life_days` | `90` | Half-life for exponential decay weighting |

### Calendar Settings

| Key | Default | Description |
|-----|---------|-------------|
| `watched_calendars` | `[]` | Calendar IDs to watch (empty = all except action calendar) |
| `ignored_calendars` | `[]` | Calendar IDs to skip entirely |
| `action_cleanup_days` | `30` | Days before canceled action events are deleted |
| `openclaw_cal_id` | `""` | ID of the action calendar (set by setup.sh) |
| `default_user_calendar` | `""` | Primary calendar for event creation |

### Notification Settings

| Key | Default | Description |
|-----|---------|-------------|
| `notification_channels` | `["openclaw","system"]` | `openclaw` · `system` · `telegram` |
| `telegram.bot_token` | `""` | Telegram bot token |
| `telegram.chat_id` | `""` | Your Telegram chat ID |
| `notes_destination` | `"local"` | `local` · `apple-notes` · `notion` |
| `clawhub_token` | `""` | Token from clawhub.ai/settings/integrations |

### Feature Flags

All local features are **ON** by default. External-facing features are **OFF** by default and must be explicitly enabled.

**Local features (default: `true`):**

| Flag | Description |
|------|-------------|
| `feature_conversation` | Conversation radar scoring |
| `feature_calendar` | Calendar scanning |
| `feature_daemon` | Background daemon |
| `feature_memory` | SQLite memory |
| `feature_conflicts` | Conflict detection |
| `feature_rules` | Rules engine |
| `feature_intelligence_loop` | Follow-ups + digest |
| `feature_policy_engine` | Calendar policies |
| `feature_orchestrator` | Multi-agent orchestration |
| `feature_energy` | Energy prediction |
| `feature_cal_editor` | Calendar editing |
| `feature_relationship` | Relationship CRM |
| `feature_adaptive_notifications` | Self-tuning notifications |
| `feature_proactivity_engine` | Unified proactivity scoring |
| `feature_interrupt_controller` | Interruption governance |
| `feature_explainability` | Decision trace |
| `feature_health_check` | System diagnostics |
| `feature_simulation` | Simulation mode |
| `feature_export` | Data export/import |
| `feature_behaviour_report` | Drift monitoring |
| `feature_config_wizard` | Config wizard |
| `feature_policy_conflict_detection` | Policy conflict alerts |

**External features (default: `false` — must opt in):**

| Flag | Description |
|------|-------------|
| `feature_cross_skill` | GitHub/Notion context (makes external network calls) |
| `feature_voice` | Voice transcription (requires whisper skill) |
| `feature_team_awareness` | Team cross-calendar (accesses other calendars) |
| `feature_llm_rater` | LLM rater (external if using cloud backend) |
| `feature_telegram_notifications` | Telegram push notifications (requires bot token) |

### LLM Rater Config

| Key | Default | Description |
|-----|---------|-------------|
| `llm_rater.enabled` | `false` | Enable the rater |
| `llm_rater.base_url` | `"http://localhost:11434/v1"` | LLM endpoint (Ollama = local, no key needed) |
| `llm_rater.model` | `"qwen2.5:3b"` | Model name |
| `llm_rater.api_key_env` | `""` | Env var holding API key (empty = no key) |
| `llm_rater.timeout` | `30` | Request timeout in seconds |

---

## 📖 Feature Reference

---

### Feature 1 — 📡 Conversation Radar

Score 0–10 silently after every exchange. Ask once, briefly, at threshold.

| Score Contribution | Signal |
|-------------------|--------|
| +3 | Explicit future event |
| +3 | Active preparation language |
| +2 | Importance / stress markers |
| +2 | Hard deadline |
| +1 | Recurring obligation |
| +1 | Post-event reflection |
| −2 | Hypothetical or historical |

**Before asking**, check pending nudges from daemon:
```bash
python3 ~/.openclaw/workspace/skills/proactive-agent/scripts/cross_skill.py --pending-nudges
```
If nudges exist, surface the most urgent one first instead of a new ask.

---

### Feature 2 — 📅 Calendar Monitoring + Conflict Detection

```bash
# Scan calendar (cache-aware)
python3 scan_calendar.py

# Detect conflicts
python3 scan_calendar.py | python3 conflict_detector.py
```

**Conflict types detected:** Overlaps · Overloaded days (4+ events) · Back-to-back runs (3+ with <10 min gaps)

**Library function** (used by action_planner):
```python
from scan_calendar import scan_user_events
events = scan_user_events(config, backend, now, time_max)
# Respects watched_calendars and ignored_calendars
```

---

### Feature 3 — ⚙️ Background Daemon + Conversation Mode

#### Daemon (Automatic, every 15 min)

**Phase 1 — PLAN** (`action_planner.py`):
- Ingest user events into link graph
- Detect missing events (deletion/move detection)
- Auto-relink moved events via fingerprint matching
- Create confirm_delete actions after 2 consecutive misses
- Plan reminder, prep, buffer, and debrief actions

**Phase 2 — EXECUTE** (`action_executor.py`):
- Fire only due actions from the action calendar
- Idempotency: check `sent_actions` table before sending
- Skip paused/canceled/suppressed actions

**Phase 3 — CLEANUP** (once daily, `action_cleanup.py`):
- Rename paused events → `🦞 [Paused] Original Title`
- Rename canceled events → `🦞 [Canceled] Original Title`
- Delete canceled entries older than `action_cleanup_days`

**Legacy fallback**: If `action_planner` fails, falls back to v1.1.x scan→notify flow automatically.

```bash
python3 daemon.py --status
python3 daemon.py --simulate --days 7
```

#### Conversation Mode (Manual, Per-Request)

Claude Code can call scripts during conversations. **This is NOT automatic** — each call requires:
1. You enable it explicitly (e.g., "Check my calendar")
2. Claude Code shows you the proposed action
3. You approve before execution
4. `max_autonomy_level: confirm` enforces step 3 (default)

---

### Feature 4 — 🧠 SQLite Memory + Semantic Search

```bash
python3 memory.py --save '{"event_title":"Demo","sentiment":"positive","follow_up_needed":true}'
python3 memory.py --search "times I felt underprepared"
python3 memory.py --open-actions
python3 memory.py --summary --days 90
```

Memory uses **decay-weighted averages** — recent outcomes count more. Configure via `memory_decay_half_life_days`.

---

### Feature 5 — 🔗 Cross-Skill Intelligence

> **Scope:** `cross_skill.py` does NOT read other skills' tokens, config files, or stored data.
> It only checks if specific skills are installed and uses their CLI tools if present.

```bash
python3 cross_skill.py --event-title "Sprint Review" --event-type "one_off_high_stakes"
python3 cross_skill.py --list-available
```

| Skill | Requires | What it reads |
|-------|---------|---------------|
| GitHub | `gh` CLI authenticated | Open PRs + issues (read-only) |
| Notion | `NOTION_API_KEY` env var | Pages matching event title |

`feature_cross_skill` defaults to `false`.

---

### Feature 6 — 📝 Natural Language Rules

```bash
python3 rules_engine.py --parse "Never bother me about standups unless I haven't spoken in 2 weeks"
python3 rules_engine.py --parse "Always prep me 2 days before anything with the word board"
python3 rules_engine.py --list
```

---

### Feature 7 — 🔄 Post-Event Intelligence Loop

```bash
python3 intelligence_loop.py --weekly-digest
python3 intelligence_loop.py --check-followups
python3 intelligence_loop.py --create-followups
python3 intelligence_loop.py --summary --days 90
```

---

### Feature 8 — 🤖 Calendar Policy Engine

Parse and execute natural language calendar policies autonomously:

```bash
python3 policy_engine.py --parse "Always block 1 hour of prep time before board meetings"
python3 policy_engine.py --parse "Add 15 min buffer after back-to-back meetings"
python3 policy_engine.py --parse "Block focus time every Tuesday morning"
python3 policy_engine.py --parse "Always schedule a debrief 30 min after investor calls"
python3 policy_engine.py --evaluate   # run all policies against current calendar
python3 policy_engine.py --list
python3 policy_engine.py --delete <id>
```

Respects `max_autonomy_level`:

| Level | Behavior |
|-------|----------|
| `advisory` | Suggests actions — never executes |
| `confirm` | Presents action for approval |
| `autonomous` | Executes immediately |

---

### Feature 9 — 🎭 Multi-Agent Orchestration

```bash
python3 orchestrator.py \
  --event-id <id> --event-title "Sprint Review" \
  --event-type one_off_high_stakes \
  --event-datetime 2025-03-15T10:00:00

python3 orchestrator.py --dry-run ...
```

**Pipeline:** Open action items → cross-skill context → outcome patterns → prep block → draft email → Notion notes → enriched nudge.

---

### Feature 10 — ⚡ Predictive Energy Scheduling

Uses **decay-weighted scoring** — recent energy data counts more than historical data.

```bash
python3 energy_predictor.py --analyse
python3 energy_predictor.py --suggest-focus-time
python3 energy_predictor.py --check "2025-03-15T09:00:00" one_off_high_stakes
python3 energy_predictor.py --block-focus-week
```

---

### Feature 11 — ✏️ Natural Language Calendar Editing

```bash
python3 cal_editor.py --move "Sprint Review" "next Monday 2pm"
python3 cal_editor.py --find-free "tomorrow" --duration 60
python3 cal_editor.py --clear "this Friday afternoon"   # OpenClaw events only (safe)
python3 cal_editor.py --read "this week"
python3 cal_editor.py --reschedule-conflict
```

---

### Feature 12 — 👥 Relationship Memory

```bash
python3 relationship_memory.py --ingest
python3 relationship_memory.py --lookup "Alice"
python3 relationship_memory.py --brief "Sprint Review"
python3 relationship_memory.py --stale --days 30
python3 relationship_memory.py --top
python3 relationship_memory.py --add-note alice@example.com "Prefers async updates"
```

---

### Feature 13 — 🎙️ Voice-First Interaction

```bash
python3 voice_bridge.py --check-whisper
python3 voice_bridge.py --record --seconds 10
python3 voice_bridge.py --transcribe /path/audio.wav
python3 voice_bridge.py --route "move sprint review to next Monday"
```

Backends: OpenClaw `whisper` skill → `openai-whisper` package → `whisper` CLI.

---

### Feature 14 — 🔔 Adaptive Notification Intelligence

Uses **decay-weighted response scoring** — recent response patterns count more.

```bash
python3 adaptive_notifications.py \
  --record-response <nudge_id> opened \
  --event-type one_off_high_stakes \
  --channel system --sent-at 2025-03-15T09:00:00

python3 adaptive_notifications.py --get-channel "one_off_high_stakes"
python3 adaptive_notifications.py --get-timing "Monday"
python3 adaptive_notifications.py --analyse
```

---

### Feature 15 — 👫 Team Awareness

Opt-in cross-calendar coordination. All sharing is explicit — nothing automatic.

```bash
python3 team_awareness.py --add-member alice@example.com "Alice"
python3 team_awareness.py --availability "this week"
python3 team_awareness.py --meeting-time "Sprint Review" \
  --attendees "alice@example.com,bob@example.com"
```

---

### Feature 16 — 🤖 LLM Interaction Rater

Local-first quality rating. Defaults to Ollama — no cloud account needed.

```bash
python3 llm_rater.py --outcome-file <path>
python3 llm_rater.py --check-backend
python3 llm_rater.py --list-backends
```

---

### Feature 17 — 🎯 Proactivity Engine 🆕

Unified scoring core that merges 5 signal sources into a single proactivity score per event:

```bash
python3 proactivity_engine.py --score <scan_json>
python3 proactivity_engine.py --score-event <event_json>
python3 proactivity_engine.py --history <event_id>
```

**Signal merging:**

| Signal | Source |
|--------|--------|
| `energy_delta` | `energy_predictor.check_event_timing()` |
| `notification_delta` | `adaptive_notifications` frequency preference |
| `policy_delta` | `policy_engine` boost/suppress |
| `relationship_delta` | `relationship_memory` high-impact contacts |
| `base_score` | Conversation radar + event type |

Applies `proactivity_mode` multiplier (low=0.5, balanced=1.0, executive=1.3). Stores results in `proactivity_scores` SQLite table. **Local only.**

---

### Feature 18 — 🚦 Interruption Governance 🆕

Priority-based nudge filtering with enforcement of daily limits, cooldowns, and quiet hours:

```bash
python3 interrupt_controller.py --filter <scan_json>
python3 interrupt_controller.py --status
python3 interrupt_controller.py --record-dismissal <event_id>
python3 interrupt_controller.py --quiet-hours-check
```

**Priority tiers:**

| Tier | Category | Example |
|------|----------|---------|
| P0 🔴 | Safety | Conflicts, double-books |
| P1 🟠 | High-stakes prep | < 24h to important event |
| P2 🟡 | Policy-triggered | Auto-blocked prep/focus time |
| P3 🔵 | Follow-up | Stale action items |
| P4 ⚪ | Routine | Periodic check-ins |
| P5 ⚫ | Informational | Digests, stats |

**Mode limits:** `low` = 3 · `balanced` = 6 · `executive` = 12 nudges per session. **Local only.**

---

### Feature 19 — 🔍 Explainability Mode 🆕

Trace every decision the system makes:

```bash
python3 explain.py --explain-nudge <event_id>         # show signal breakdown
python3 explain.py --explain-policy <policy_id>        # show policy match history
python3 explain.py --explain-energy-decision <event_id> # show energy slot data used
python3 explain.py --trace <event_id>                  # full decision trace
```

Shows: all signal contributions, priority classification, suppression reasons, scoring breakdowns. **Local only.**

---

### Feature 20 — 🎚️ Proactivity Intensity Dial 🆕

Global mode that controls scoring multiplier and nudge limits:

| Mode | Score Multiplier | Max Nudges/Session |
|------|:---------------:|:-----------------:|
| `low` | 0.5 | 3 |
| `balanced` | 1.0 | 6 |
| `executive` | 1.3 | 12 |

```json
"proactivity_mode": "balanced"
```

---

### Feature 21 — 🔐 Max Autonomy Cap 🆕

Global override that limits what the system can do without asking:

| Level | Behavior |
|-------|----------|
| `advisory` | Suggest only — never create or modify events |
| `confirm` | Present action for approval before executing ✅ **Default** |
| `autonomous` | Act immediately (not recommended) |

```json
"max_autonomy_level": "confirm"
```

---

### Feature 22 — ⏳ Memory Decay 🆕

Exponential recency weighting across all modules. Recent data gets higher weight, old data fades naturally.

**Used by:** `energy_predictor.py` · `adaptive_notifications.py` · `memory.py` · `proactivity_engine.py`

```json
"memory_decay_half_life_days": 90
```

Shared library: `scripts/decay.py` — pure math, no I/O, no network, no subprocess.

---

### Feature 23 — 🏥 System Health Audit 🆕

7 diagnostic checks in one command:

```bash
python3 health_check.py                    # full report
python3 health_check.py --check db         # database integrity (PRAGMA integrity_check)
python3 health_check.py --check daemon     # is daemon running?
python3 health_check.py --check config     # config validity
python3 health_check.py --check calendar   # calendar connectivity
python3 health_check.py --check flags      # feature flag consistency
python3 health_check.py --check stale      # stale data detection (30+ days no outcomes)
python3 health_check.py --check disk       # disk usage (memory.db, log size)
```

> **Note:** The `daemon` check uses `subprocess` to run `launchctl list` (macOS) or `systemctl --user status` (Linux) — local process inspection only. The `calendar` check uses the same network scope as core calendar features.

---

### Feature 24 — ⚡ Policy Conflict Detection 🆕

Detects contradictory policies before they cause issues:

```bash
python3 policy_conflict_detector.py --check-all
python3 policy_conflict_detector.py --check-new '<policy_json>'
```

**Detects:** Same event type with conflicting actions · Overlapping time conditions · Autonomy mismatches · Duplicate policies. **Local only.**

---

### Feature 25 — 🧙 Config Wizard 🆕

Interactive CLI setup — the safest way to create `config.json`:

```bash
python3 config_wizard.py              # interactive mode (recommended)
python3 config_wizard.py --defaults   # non-interactive, safe defaults
python3 config_wizard.py --validate   # check existing config
```

Detects system timezone, walks through: backend · mode · autonomy · channels · quiet hours. Writes `config.json` only.

---

### Feature 26 — 🔬 Simulation Mode 🆕

Dry-run the daemon over N future days to preview what would fire:

```bash
python3 daemon.py --simulate --days 7
```

All state in-memory only — **no writes** to DB, calendar, or files. Safe to run anytime.

---

### Feature 27 — 🌙 Quiet Hours 🆕

Suppress non-safety (non-P0) nudges during configured quiet windows:

```json
"quiet_hours": {
  "weekdays": "22:00-07:00",
  "weekends": "21:00-09:00"
}
```

P0 safety nudges (double-bookings, conflicts) still come through. Implemented in `interrupt_controller.py`.

---

### Feature 28 — 💾 Data Export / Import 🆕

Backup and restore all local data:

```bash
python3 export_data.py --export --output ~/backup --format json
python3 export_data.py --export --output ~/backup --format csv
python3 export_data.py --import ~/backup
python3 export_data.py --list-tables
```

Exports all SQLite tables + redacted config (secrets stripped). Import uses `INSERT OR REPLACE`. **Local file I/O only.**

---

### Feature 29 — 📊 Drift Monitoring 🆕

Monthly behaviour reports that detect concerning trends:

```bash
python3 behaviour_report.py --monthly
python3 behaviour_report.py --snapshot
python3 behaviour_report.py --compare "2025-01" "2025-02"
python3 behaviour_report.py --drift-alert
```

**Alerts when:** Dismiss rate increases >20% · Prep rate drops >15% · Negative sentiment rises >15%. **Local only.**

---

### Feature 30 — 📆 2-Calendar Architecture 🆕

Your calendars are **read-only**. All actions are written to the **"Proactive Claw — Actions"** calendar.

```json
"watched_calendars": [],       // which to monitor (empty = all except action calendar)
"ignored_calendars": [],       // which to skip entirely
"openclaw_cal_id": "..."       // set automatically by setup.sh
```

**Migration:** `setup.sh` recognises both old name ("OpenClaw") and new name ("Proactive Claw — Actions") — existing setups migrate automatically.

---

### Feature 31 — 🕸️ Link Graph 🆕

SQLite database (`proactive_links.db`) connecting user events to planned actions:

| Table | Purpose |
|-------|---------|
| `user_events` | Tracked user events with fingerprint + missing_count |
| `action_events` | Planned actions (reminder, prep, buffer, debrief, confirm_delete) |
| `links` | Connects user_events ↔ action_events |
| `suppression` | Events the user said "don't ask me about this" |
| `sent_actions` | Idempotency log — action_uid + due_ts = unique key |

```bash
python3 link_store.py --status       # graph stats
python3 link_store.py --missing      # events with missing_count > 0
python3 link_store.py --links <uid>  # show all linked actions for a user event
```

**Fingerprint:** SHA256 of normalized `title|start|end|attendees|location` — detects moved/recreated events. **Local only.**

---

### Feature 32 — 📋 Action Planner (PLAN Phase) 🆕

```bash
python3 action_planner.py --plan      # full plan cycle
python3 action_planner.py --dry-run   # show what would be planned
python3 action_planner.py --status    # show plan stats
```

**Steps:**
1. **Ingest** — Upsert all seen user events, reset missing_count
2. **Detect missing** — Mark unseen events, increment missing_count, pause linked actions
3. **Auto-relink** — Check fingerprint/title match for moved events
4. **Confirm delete** — After ≥2 consecutive misses, create confirm_delete action
5. **Plan actions** — Create reminder, prep, buffer, debrief actions based on policies

---

### Feature 33 — ▶️ Action Executor (EXECUTE Phase) 🆕

```bash
python3 action_executor.py --execute  # fire due actions
python3 action_executor.py --dry-run  # show what would fire
python3 action_executor.py --due      # list due actions
```

- Idempotent: checks `sent_actions` table before sending
- Skips paused/canceled/suppressed actions
- Default lookahead: 20 minutes (> daemon interval of 15 min)

---

### Feature 34 — 🗑️ Deletion Detection 🆕

When a user event disappears from the calendar:

| Miss # | Action |
|--------|--------|
| 1st miss | `missing_count` incremented, linked actions paused, fingerprint check attempted |
| 2nd miss | `confirm_delete` action created with 3 options |

**Confirm delete options:**
```bash
python3 confirm_delete.py --yes <user_event_uid>       # mark deleted, cancel linked actions
python3 confirm_delete.py --no <user_event_uid>        # 24h cooldown, expanded recovery scan
python3 confirm_delete.py --dont-ask <user_event_uid>  # suppress event permanently
```

**Local only.**

---

### Feature 35 — 🦞 Soft-Cancel Policy 🆕

Canceled action events aren't deleted immediately — they're renamed so users can see what happened:

| Status | Calendar Title |
|--------|---------------|
| Paused | `🦞 [Paused] Original Title` |
| Canceled | `🦞 [Canceled] Original Title` |

```bash
python3 action_cleanup.py --cleanup             # run cleanup cycle
python3 action_cleanup.py --cleanup --dry-run   # show what would be cleaned
python3 action_cleanup.py --status              # show cleanup stats
```

Entries older than `action_cleanup_days` (default 30) are permanently deleted.

---

## 🏷️ PC_ACTION Metadata Encoding

Action calendar events include a machine-readable marker in the description for cross-backend compatibility:

```
PC_ACTION: {"action_event_uid":"abc123","action_type":"reminder","source_event_uid":"def456","status":"active"}
```

This allows the system to track which action events belong to which user events, even when using CalDAV backends that don't support custom properties. **Pure string encoding — no I/O.**

```bash
python3 -c "from action_codec import decode_action_description; print(decode_action_description('...'))"
```

---

## 🔁 Recurring Event Intelligence

| Type | Detection | Behaviour |
|------|-----------|-----------|
| `routine_low_stakes` | Recurring + internal + avg 0 action items | Suppress — every 4th occurrence only |
| `routine_high_stakes` | Recurring + external OR avg ≥ 2 action items | Always check in, personalise with history |
| `one_off_standard` | Not recurring, < 60 min, internal | Standard scoring |
| `one_off_high_stakes` | Not recurring + external OR importance signals | Max prep — full orchestration pipeline |

---

## 📝 Auto Agenda & Talking Points

| Event Type | Auto-generated Content |
|-----------|----------------------|
| Presentation / Demo | Hook → Problem → Solution → Demo → CTA |
| Interview | STAR prompts for role/company if mentioned |
| 1:1 | Open action items + relationship brief |
| Standup | GitHub activity from cross_skill.py |
| Board / Investor | Metrics, narrative arc, likely hard questions |
| Workshop | Desired outcomes, pre-reads |
| External (no history) | Company/attendee context + relationship brief |

---

## ⚠️ Error Handling

| Error | User Message |
|-------|-------------|
| `calendar_backend_unavailable` | "Can't reach your calendar. Try again, or continue without calendar features?" |
| `failed_to_list_calendars` | "Trouble reading calendars. Check connection and that setup.sh ran." |
| `failed_to_create_events` | "Couldn't create check-in events — [detail]. Try again?" |
| Setup not run | "Calendar not set up yet. Run: `bash ~/.openclaw/workspace/skills/proactive-agent/scripts/setup.sh`" |
| `python_version_too_old` | "Python 3.8+ required. Install at https://www.python.org/downloads/" |
| Daemon not installed | "Background notifications are off. Run install_daemon.sh to enable." |
| Voice backend missing | "No transcription backend found. Run: `pip install openai-whisper`" |
| Team calendar not accessible | "Alice's calendar isn't accessible. Ask her to share it with your Google account." |
| `action_planner_failed` | Falls back to v1.1.x scan→notify flow automatically |
| `link_graph_unavailable` | "Link graph database unavailable. Actions may not be tracked." |

---

## 📋 Script Audit — Full Inventory

All v1.2.0 scripts are **local-only** unless noted.

| Script | Network | Subprocess | Notes |
|--------|---------|------------|-------|
| `scan_calendar.py` | Google/Nextcloud API | — | Core calendar read |
| `conflict_detector.py` | — | — | Local analysis |
| `daemon.py` | Google/Nextcloud API, Telegram (opt-in) | `osascript` (macOS), `notify-send` (Linux) | Core daemon loop |
| `memory.py` | — | — | Local SQLite |
| `capture_outcome.py` | Notion API (opt-in) | `osascript` (Apple Notes, opt-in) | Outcome storage |
| `create_checkin.py` | Google/Nextcloud API | — | Creates calendar check-in events |
| `cross_skill.py` | Notion API (opt-in), GitHub via `gh` CLI (opt-in) | `gh pr list`, `gh issue list` (opt-in) | External context |
| `rules_engine.py` | — | — | Local SQLite |
| `intelligence_loop.py` | — | `python3` (calls scan/conflict scripts) | Local orchestration |
| `policy_engine.py` | Google/Nextcloud API (creates events) | — | Calendar writes to action calendar only |
| `orchestrator.py` | — | `python3` (calls sub-scripts) | Local orchestration |
| `energy_predictor.py` | Google/Nextcloud API (creates focus blocks) | — | Calendar writes to action calendar only |
| `cal_editor.py` | Google/Nextcloud API | `python3` (calls scan/conflict) | Calendar read/write |
| `relationship_memory.py` | — | — | Local SQLite |
| `voice_bridge.py` | — | `whisper`, `sox`, `arecord`/`afrecord` | Local audio processing |
| `adaptive_notifications.py` | — | — | Local SQLite |
| `team_awareness.py` | Google/Nextcloud API (reads shared calendars) | — | Opt-in team features |
| `llm_rater.py` | LLM API (local Ollama default, cloud opt-in) | — | Rating endpoint |
| `cal_backend.py` | Google/Nextcloud API | — | Calendar abstraction layer |
| `setup.sh` | clawhub.ai (opt-in), Google OAuth, Nextcloud | `pip3 install` | One-time setup |
| `install_daemon.sh` | — | `launchctl`/`systemctl` | One-time daemon install |
| **`decay.py`** 🆕 | — | — | Pure math library |
| **`proactivity_engine.py`** 🆕 | — | — | Local SQLite scoring |
| **`interrupt_controller.py`** 🆕 | — | — | Local SQLite nudge filter |
| **`explain.py`** 🆕 | — | — | Local SQLite trace |
| **`health_check.py`** 🆕 | Calendar API (connectivity check) | `launchctl`/`systemctl` (daemon check) | Diagnostic tool |
| **`policy_conflict_detector.py`** 🆕 | — | — | Local SQLite |
| **`config_wizard.py`** 🆕 | — | — | Interactive CLI, writes config.json |
| **`export_data.py`** 🆕 | — | — | Local file I/O |
| **`behaviour_report.py`** 🆕 | — | — | Local SQLite |
| **`link_store.py`** 🆕 | — | — | Local SQLite (proactive_links.db) |
| **`action_codec.py`** 🆕 | — | — | Pure string encoding |
| **`action_planner.py`** 🆕 | Via scan_calendar (Google/Nextcloud API) | — | PLAN phase |
| **`action_executor.py`** 🆕 | Via daemon send_notification (system/Telegram) | — | EXECUTE phase |
| **`confirm_delete.py`** 🆕 | — | — | Local SQLite |
| **`action_cleanup.py`** 🆕 | Via cal_backend (Google/Nextcloud API) | — | Calendar event rename/delete |

---

## 🗃️ SQLite Tables

### memory.db

| Table | Purpose |
|-------|---------|
| `outcomes` | Event outcomes with sentiment, action items, notes |
| `rules` | User-defined natural language rules |
| `policies` | Calendar automation policies |
| `contacts` | Relationship memory CRM |
| `notification_log` | Adaptive notification tracking |
| `energy_scores` | Energy prediction data |
| `proactivity_scores` | Unified proactivity scoring (v1.2.0) |
| `nudge_log` | Interruption governance tracking (v1.2.0) |
| `policy_conflicts` | Detected policy contradictions (v1.2.0) |
| `behaviour_snapshots` | Monthly drift monitoring data (v1.2.0) |

### proactive_links.db *(new in v1.2.0)*

| Table | Purpose |
|-------|---------|
| `user_events` | Tracked user events with fingerprint + missing_count |
| `action_events` | Planned actions with type, status, due_ts |
| `links` | User event ↔ action event connections |
| `suppression` | Events user said "don't ask about" |
| `sent_actions` | Idempotency log for action execution |

---

## 📜 install_daemon.sh — Full Source

This script does exactly **four things**:
1. Detects macOS or Linux
2. Writes a plist to `~/Library/LaunchAgents/` (macOS) or `.service` + `.timer` to `~/.config/systemd/user/` (Linux) — **user directory, not system**
3. Registers the timer with `launchctl load` or `systemctl --user enable`
4. Prints status and uninstall instructions

**No `sudo`. No root. No downloads. No curl/wget. No network calls. Writes only to your home directory.**

```bash
#!/bin/bash
# install_daemon.sh — Install proactive-agent as a background daemon
# Supports: macOS (launchd) | Linux (systemd user service)
# Run once after setup.sh

set -e

SKILL_DIR="$HOME/.openclaw/workspace/skills/proactive-agent"
PYTHON=$(command -v python3)
PLATFORM=$(uname -s)

if [ "$PLATFORM" = "Darwin" ]; then
  PLIST_DIR="$HOME/Library/LaunchAgents"
  PLIST="$PLIST_DIR/ai.openclaw.proactive-agent.plist"
  mkdir -p "$PLIST_DIR"
  cat > "$PLIST" << EOF
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN"
  "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>Label</key>         <string>ai.openclaw.proactive-agent</string>
  <key>ProgramArguments</key>
  <array>
    <string>$PYTHON</string>
    <string>$SKILL_DIR/scripts/daemon.py</string>
  </array>
  <key>StartInterval</key> <integer>900</integer>
  <key>RunAtLoad</key>     <true/>
  <key>StandardOutPath</key>  <string>$SKILL_DIR/daemon.log</string>
  <key>StandardErrorPath</key><string>$SKILL_DIR/daemon.log</string>
  <key>EnvironmentVariables</key>
  <dict>
    <key>HOME</key> <string>$HOME</string>
    <key>PATH</key> <string>/usr/local/bin:/usr/bin:/bin:/opt/homebrew/bin</string>
  </dict>
  <key>WorkingDirectory</key> <string>$SKILL_DIR</string>
</dict>
</plist>
EOF
  launchctl unload "$PLIST" 2>/dev/null || true
  launchctl load "$PLIST"

elif [ "$PLATFORM" = "Linux" ]; then
  SERVICE_DIR="$HOME/.config/systemd/user"
  mkdir -p "$SERVICE_DIR"
  cat > "$SERVICE_DIR/openclaw-proactive-agent.service" << EOF
[Unit]
Description=OpenClaw Proactive Agent
After=network.target
[Service]
Type=oneshot
ExecStart=$PYTHON $SKILL_DIR/scripts/daemon.py
StandardOutput=append:$SKILL_DIR/daemon.log
StandardError=append:$SKILL_DIR/daemon.log
Environment=HOME=$HOME
EOF
  cat > "$SERVICE_DIR/openclaw-proactive-agent.timer" << EOF
[Unit]
Description=Run OpenClaw Proactive Agent every 15 minutes
[Timer]
OnBootSec=2min
OnUnitActiveSec=15min
Unit=openclaw-proactive-agent.service
[Install]
WantedBy=timers.target
EOF
  systemctl --user daemon-reload
  systemctl --user enable --now openclaw-proactive-agent.timer
else
  echo "Platform not supported. Run manually: python3 $SKILL_DIR/scripts/daemon.py --loop"
fi
```

---

## 📦 setup.sh — What Each Step Does

| Step | What it does | Network? |
|------|-------------|----------|
| 1 | Checks Python 3.8+ is installed | No |
| 2 | Reads `calendar_backend` from config.json (defaults to `google`) | No |
| 3 | **If** `clawhub_token` set AND `credentials.json` missing: fetches OAuth config from `clawhub.ai/api/oauth/google-calendar-credentials` | One HTTPS GET to clawhub.ai only, **optional** |
| 4 | Creates default `config.json` if it doesn't exist | No |
| 5 | Creates `outcomes/` directory | No |
| Nextcloud path | `pip3 install caldav icalendar`; connects to Nextcloud; creates action calendar if missing | HTTPS to **your own Nextcloud only** |
| Google path | `pip3 install google-api-python-client google-auth-oauthlib google-auth-httplib2`; OAuth flow; creates action calendar | HTTPS to **Google OAuth + Calendar API only** |

**No curl/wget. No arbitrary downloads. No root. No system file modifications. No data sent to skill author.**

---

## 🎯 Tone & Interaction Rules

- **One question at a time** — never stack asks
- **Daemon nudges first** — check `pending_nudges.json` before starting new asks at conversation open
- **Never repeat** the same event ask twice in one conversation
- **Always confirm** before writing calendar events (title, date, friendly time + tz)
- **Always confirm** before clearing or moving events — show what will change first
- **Always confirm** before writing outcome notes (bullet summary)
- **Respect "no"** — dismissed forever; "not now" snoozed
- **Be brief** — check-in prompts ≤ 2 sentences; agenda is a starting point, not a report
- **Surface, don't overwhelm** — multiple actionable items → highest-scored first
- **Timezone-aware** — always display in user's `timezone` config, never UTC
- **Privacy first** — team calendar features are opt-in, never auto-enroll anyone
- **Never reveal the Action calendar** — it's internal; never tell users to look at it directly
- **Respect autonomy cap** — if `max_autonomy_level` is `advisory`, never create events
- **Respect quiet hours** — no non-safety nudges during configured quiet windows
