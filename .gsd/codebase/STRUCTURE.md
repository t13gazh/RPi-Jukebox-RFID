# Codebase Structure

**Updated:** 2026-03-16
**Previous:** 2026-02-06 (`.planning/codebase/` — 7 files, pre-fork)

## Repository Overview

Fork of [MiczFlor/RPi-Jukebox-RFID](https://github.com/MiczFlor/RPi-Jukebox-RFID) v2.x.
We replace only the web frontend (PHP/jQuery → Svelte/FastAPI). All hardware layers stay untouched.

**Remotes:**
- `origin` → `t13gazh/RPi-Jukebox-RFID` (our fork)
- `upstream` → `MiczFlor/RPi-Jukebox-RFID` (original)

**Branches:**
- `develop` — upstream-synced, our docs commits on top
- `feature/webui-modernization` — **active dev branch**, all new work here
- `master` — release tags reference only

## Directory Layout

```
rfid-jukebox/
├── .gsd/                       # GSD-2 project management (our addition)
│   ├── PROJECT.md              # Living project description
│   ├── REQUIREMENTS.md         # 58 v1 + 8 v2 requirements
│   ├── DECISIONS.md            # Append-only decision register
│   ├── STATE.md                # Quick-glance status
│   ├── codebase/               # THIS directory — codebase analysis
│   └── milestones/M001/        # Phoniebox Web-Interface milestone
│       ├── M001-CONTEXT.md     # Milestone scope and constraints
│       ├── M001-RESEARCH.md    # Full stack/architecture research
│       ├── M001-ROADMAP.md     # 11 slices, 4 PR groups
│       └── slices/S01-S11/     # Slice plans (goals, must-haves, empty tasks)
│
├── .planning/                  # OLD planning docs (pre-GSD, read-only reference)
│   ├── ANALYSIS.md             # v2 vs v3 evaluation, strategy decision
│   ├── codebase/               # 7-file codebase analysis (2026-02-06)
│   ├── research/               # Stack, architecture, features, pitfalls research
│   └── phases/01-*/            # Phase 1 context (implementation decisions)
│
├── components/                 # Hardware drivers (DO NOT MODIFY)
│   ├── gpio_control/           # GPIO buttons/encoder/LED — Python
│   ├── rfid-reader/            # RC522, PN532, USB reader drivers
│   ├── audio/                  # Audio HAT drivers
│   ├── displays/               # LCD displays (HD44780)
│   ├── smart-home-automation/  # MQTT integration
│   └── ...
│
├── htdocs/                     # OLD web UI (WILL BE REPLACED)
│   ├── _assets/                # jQuery 1.12.4, Bootstrap 3, CSS
│   ├── api/                    # PHP REST endpoints (player.php, playlist.php)
│   │   └── common.php          # Shell exec wrapper (command injection risk)
│   ├── js/jukebox.js           # Frontend JS (AJAX polling)
│   ├── index.php               # Main entry point
│   ├── func.php                # 600+ line utility library
│   ├── cardRegisterNew.php     # Card registration (command injection vuln)
│   └── ... (77 PHP files total)
│
├── scripts/                    # Core daemons and control (DO NOT MODIFY)
│   ├── playout_controls.sh     # Central command hub (1153 lines) ← OUR API WRAPS THIS
│   ├── daemon_rfid_reader.py   # RFID polling daemon (shell=True vuln)
│   ├── rfid_trigger_play.sh    # Card-to-folder resolver
│   ├── Reader.py               # RFID hardware abstraction
│   ├── inc.writeGlobalConfig.sh# Config merger (settings/ → global.conf)
│   ├── helperscripts/          # File org, analytics utilities
│   ├── installscripts/         # Installation automation
│   └── userscripts/            # User-definable post-trigger hooks
│
├── settings/                   # Configuration store (50+ files)
│   ├── global.conf             # Aggregated config (generated)
│   ├── rfid_trigger_play.conf  # RFID card → action mappings
│   └── *.conf.sample           # Defaults (committed)
│
├── shared/                     # User data (NOT committed)
│   ├── audiofolders/           # Music library (SMB/NFS accessible)
│   └── shortcuts/              # Card assignment metadata
│
├── tests/                      # Existing test suite
│   └── htdocs/api/             # PHP unit tests (PlayerTest, PlayListTest)
│
├── CLAUDE.md                   # Agent handoff context (our addition)
├── README.md                   # Fork-aware README (our modification)
├── .gitignore                  # Updated with .env (our modification)
├── LICENSE                     # MIT — Copyright (c) 2017 Micz Flor
├── CONTRIBUTING.md             # Original contributing guide
├── composer.json               # PHP deps (PHPUnit)
├── requirements*.txt           # Python deps (multiple variants)
└── packages*.txt               # Debian system package lists
```

## What We Touch vs. Don't Touch

### DO NOT MODIFY (existing v2 backend)

| Path | Why |
|------|-----|
| `scripts/playout_controls.sh` | Single source of truth for player commands. RFID, GPIO, gyro all route through it. Our API wraps it via subprocess. |
| `scripts/daemon_rfid_reader.py` | RFID polling daemon. Runs as systemd service. We read its output (latestID.txt), don't modify it. |
| `scripts/rfid_trigger_play.sh` | Card-to-folder resolver. Called by RFID daemon. |
| `components/gpio_control/` | GPIO buttons/encoder. Calls playout_controls.sh directly. |
| `components/rfid-reader/` | RC522/PN532/USB drivers. Hardware abstraction. |
| `settings/` (format) | Config file format must stay compatible with shell `source`. Our SQLite is additive. |
| `shared/audiofolders/` | Music library structure. MPD indexes this. |

### WILL REPLACE (old web frontend)

| Path | Replaced By |
|------|-------------|
| `htdocs/` (77 PHP files) | SvelteKit SPA (static build served by Nginx) |
| `htdocs/api/` (PHP endpoints) | FastAPI REST endpoints |
| `htdocs/_assets/` (jQuery/Bootstrap) | Svelte 5 + Tailwind CSS |
| Lighttpd config | Nginx (WebSocket proxy support) |

### NEW (our additions)

| Path | Purpose |
|------|---------|
| `.gsd/` | GSD-2 project management |
| `CLAUDE.md` | Agent context and handoff |
| `README.md` | Fork-aware documentation |
| (future) `api/` | FastAPI server |
| (future) `ui/` | SvelteKit frontend |
| (future) `deploy/` | Nginx config, systemd units, deploy scripts |

## Key Integration Points

These are the seams between our new code and the existing backend:

### 1. playout_controls.sh (Command Interface)

Our FastAPI calls this via `subprocess.run()` (no `shell=True`).

```
API endpoint → subprocess.run(['playout_controls.sh', '-c=playerplay']) → mpc play → MPD
```

Commands (from the 1153-line case statement):
- Player: `playerplay`, `playerpause`, `playerstop`, `playernext`, `playerprev`
- Volume: `setvolume`, `volumeup`, `volumedown`, `setmaxvolume`
- Playlist: `playlistaddplay`, `playlistappend`, `playlistclear`
- System: `shutdown`, `reboot`
- RFID: Various card-trigger commands

### 2. MPD Socket (Port 6600)

Direct TCP connection for real-time state. Our MPD bridge uses `python-mpd2` asyncio.

```
MPD idle → state change detected → WebSocket broadcast → browser updates
```

Key protocol commands: `status`, `currentsong`, `idle` (blocks until change), `playlistinfo`

### 3. Configuration Files (settings/)

Our SQLite wraps these. Changes write back to flat files for shell script compatibility.

Key files:
- `global.conf` — aggregated config (generated by `inc.writeGlobalConfig.sh`)
- `rfid_trigger_play.conf` — card ID → folder/command mappings
- Individual setting files: `Max_Volume`, `Idle_Time_Before_Shutdown`, etc.

### 4. RFID Events (latestID.txt)

The RFID daemon writes the last scanned card ID to a known file. Our API watches this with inotify for the card registration wizard.

```
daemon_rfid_reader.py → writes latestID.txt → API file-watch → WebSocket push → browser
```

### 5. Audio File System (shared/audiofolders/)

Our library browser reads this directory tree. Cover art is `cover.jpg` in each folder.

```
shared/audiofolders/
├── Die drei Fragezeichen/
│   ├── cover.jpg
│   ├── track01.mp3
│   └── track02.mp3
├── Bibi Blocksberg/
│   ├── cover.jpg
│   └── ...
```

### 6. phonie-gyro (External Service)

Standalone systemd service. Calls `mpc` directly (decoupled from Phoniebox).
Our web UI reads/writes its config file for the settings UI.

Repository: https://github.com/t13gazh/phonie-gyro

## Technology Stack

### Current (v2 — what exists)

| Layer | Technology | Status |
|-------|-----------|--------|
| Web server | Lighttpd | Will be replaced by Nginx |
| Web frontend | PHP 7+ / jQuery 1.12.4 / Bootstrap 3 | Will be replaced |
| Web API | PHP `exec()` → shell scripts | Will be replaced by FastAPI |
| Player control | `playout_controls.sh` (Bash, 1153 lines) | Kept — our API wraps it |
| Audio engine | MPD on port 6600 | Kept |
| RFID daemon | Python 3 (`daemon_rfid_reader.py`) | Kept |
| GPIO control | Python 3 (`gpio_control.py`) | Kept |
| Gyro sensor | Python 3 (phonie-gyro, systemd service) | Kept — new config UI |
| Config | 50+ flat files in `settings/` | Kept + SQLite overlay |
| Testing | PHPUnit (PHP), pytest (Python) | Kept + new API/UI tests |
| CI | GitHub Actions (5 workflows) | Kept |

### Planned (new web stack)

| Layer | Technology | Notes |
|-------|-----------|-------|
| Web server | Nginx | WebSocket proxy, static file serving |
| Frontend | Svelte 5 + SvelteKit (adapter-static) | Built on dev machine, deployed as static files |
| Styling | Tailwind CSS | Mobile-first, purged to tiny bundles |
| Backend API | FastAPI + Uvicorn (Python) | REST + WebSocket hub |
| Real-time | WebSocket + MPD idle bridge | Replaces 5s polling |
| Config DB | SQLite (WAL mode) | Replaces flat files for API, writes back for shell |
| Auth | PIN-based JWT (3 tiers) | Open / Parent / Expert |
| Deployment | rsync/scp from dev machine | Single-command deploy |

## Security Notes (Inherited Issues)

These exist in the current codebase and are documented for awareness. Our new API must not replicate them.

| Issue | Location | Our Mitigation |
|-------|----------|---------------|
| Command injection | `htdocs/cardRegisterNew.php` (lines 118, 140) | FastAPI uses `subprocess.run()` with argument lists, never `shell=True` |
| `shell=True` | `scripts/daemon_rfid_reader.py` (line 99) | We don't modify this file; our API doesn't use `shell=True` |
| No authentication | All `htdocs/` files | Three-tier PIN model in S09 |
| chmod 777 | Multiple PHP files | Proper permissions (644/755) in our code |
| Unsanitized file uploads | `htdocs/cardRegisterNew.php` | Chunked upload with validation in S06 |
| No CSRF protection | All forms | Not applicable (SPA with JWT) |

## Test Infrastructure

### Existing

```bash
composer run-script test          # PHP unit tests (excludes real-env)
pytest --cov                      # Python tests with coverage
flake8 --config .flake8           # Python linting (max-line 127, complexity 12)
```

- Python tests: `components/gpio_control/test/` (SimpleButton, RotaryEncoder, LED, GPIOControl)
- PHP tests: `tests/htdocs/api/` (PlayerTest, PlayListTest, TrackEditTest)
- CI: 5 GitHub Actions workflows (Python, PHP, Docker, CodeQL, Markdown)

### Planned (our additions)

- FastAPI: pytest + httpx for API endpoint tests
- SvelteKit: Vitest for component tests, Playwright for E2E
- Bundle size: CI check enforcing <150KB gzipped

## References

The old `.planning/codebase/` directory (7 files, 2026-02-06) contains deeper analysis of:
- `ARCHITECTURE.md` — Layer diagram, data flows, entry points
- `CONCERNS.md` — Tech debt, security vulns, performance bottlenecks, fragile areas
- `CONVENTIONS.md` — Naming, code style, imports, error handling, logging patterns
- `INTEGRATIONS.md` — MPD, Mopidy, yt-dlp, MQTT, GPIO, RFID, serial/PCSC
- `STACK.md` — Full dependency inventory (Python, PHP, system packages)
- `STRUCTURE.md` — Directory purposes, file locations, where to add new code
- `TESTING.md` — Test framework, patterns, mocking, coverage, CI integration

These remain valid for understanding the v2 codebase. Consult them during implementation.

---

*Updated: 2026-03-16 (fork setup, Git strategy, new stack decisions)*
*Previous: 2026-02-06 (initial v2 codebase analysis in .planning/codebase/)*
