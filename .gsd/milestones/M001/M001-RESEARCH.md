# Project Research Summary

**Project:** Phoniebox Web Interface Modernization (Phase 1)
**Domain:** Embedded web UI for Raspberry Pi music player
**Researched:** 2026-02-06
**Confidence:** MEDIUM (WebSearch unavailable — based on codebase analysis + training data)

## Executive Summary

This project replaces the outdated PHP/jQuery/Bootstrap 3 web interface of a Raspberry Pi-based children's music box (Phoniebox v2.8.0) with a modern, mobile-first SPA. The core value proposition is simple: **parents can effortlessly manage their children's music box from any phone.** Research shows this requires a lightweight frontend (Svelte 5 compiled to <150KB), a Python API layer (FastAPI) that bridges to existing shell scripts and MPD, and WebSocket for real-time player state updates. The architecture is an API gateway pattern that leaves all existing daemons (RFID, GPIO, phonie-gyro) completely untouched.

The recommended approach is incremental replacement, not a rewrite. The new FastAPI server wraps existing shell scripts (`playout_controls.sh`) rather than bypassing them — critical because RFID, GPIO buttons, and gyro all route through these scripts. The frontend is built on a development machine and deployed as static files served by Nginx (which replaces Lighttpd for better WebSocket proxy support). Three-tier access control (open player / PIN-protected parent area / expert settings) provides the right balance of convenience and security for a family device on a home network.

Key risks center on bundle size (Pi 3B can't handle React/MUI bloat), preserving the shell script bridge (breaking it creates split-brain between web UI and physical controls), and command injection vulnerabilities (the current PHP code has them — the new API must not replicate them). Mitigation: enforce a 150KB gzipped bundle budget, call `playout_controls.sh` via subprocess without `shell=True`, and whitelist all commands with strict input validation.

## Key Findings

### Recommended Stack

The stack prioritizes lightweight runtime on the Pi's constrained hardware (1GB RAM shared between OS, MPD, RFID daemon, GPIO daemon, and phonie-gyro). Every choice optimizes for bundle size, minimal server memory, and leveraging Python (already the dominant backend language on this Pi).

**Core technologies:**
- **Svelte 5 + SvelteKit (adapter-static):** Compiles to vanilla JS at build time (~2KB runtime vs React's 45KB). Build happens on dev machine, Pi serves pure static files via Nginx. Mobile-first reactivity model perfect for real-time player state.
- **FastAPI + Uvicorn:** Python async API server with native WebSocket support. Wraps existing shell scripts and MPD socket communication. Python already runs RFID daemon, GPIO control, and will integrate with phonie-gyro config. ~30-50MB memory vs keeping PHP (~15-30MB) — acceptable tradeoff for WebSocket capability.
- **Nginx:** Replaces Lighttpd for native WebSocket proxy support (critical for real-time player state). Serves static SvelteKit build, proxies `/api/` and `/ws` to FastAPI. ~2-5MB memory footprint.
- **SQLite:** Replace 50+ fragile config files in `settings/` with ACID-compliant database. WAL mode survives power cuts (critical for a child's toy). Zero memory overhead (Python stdlib). No external service.
- **WebSocket:** Bidirectional real-time communication for player state (push MPD updates via idle protocol) and RFID events (push card scans during registration). Replaces wasteful 5-second polling with instant sub-100ms updates.
- **Tailwind CSS:** Utility-first CSS purged to tiny production bundles. Mobile-first by design. No JS runtime weight like MUI/Bootstrap.

**Version confidence:** MEDIUM — all versions based on training data through Jan 2025. Svelte 5, FastAPI 0.115+, Tailwind 4, SvelteKit 2 should be verified against current releases before implementation.

**Alternatives considered and rejected:**
- React 19: 45KB runtime, vDOM overhead wasteful on Pi — Svelte compiles away
- Keep PHP: No WebSocket, command injection, no async I/O — fundamentally wrong for real-time
- Express.js backend: Adds Node.js to Pi, Python already dominant — unnecessary complexity
- Bootstrap/MUI: Heavy component libraries — custom Tailwind components yield 80% smaller bundles

### Expected Features

Research into competitive landscape (Toniebox/mytonies, Volumio, moOde, Sonos) reveals clear feature tiers. The Phoniebox sweet spot: **Sonos-level simplicity applied to children's content management.**

**Must have (table stakes):**
- Player controls with now-playing display: Play/pause/prev/next, volume slider (not just +/-), progress bar with seek, cover art, track metadata. This is the home screen parents see.
- Library browsing with cover art grid: Visual folder/album browsing replaces v2's text-only folder tree. Search included.
- RFID card management: Simplified wizard for assign-card-to-folder, card overview ("card wall" showing all cards), edit/delete cards.
- Parental controls: Max volume limit, sleep timer, idle shutdown — child safety and bedtime essentials.
- Mobile-first responsive design: Touch-friendly 48x48px minimum tap targets, works perfectly on 375px phone screens.

**Should have (differentiators):**
- Three-tier access model (open player / PIN parent / expert): Unique to Phoniebox. Kids see only player, parents manage content, experts configure system. Not in any competitor.
- Card registration wizard with visual feedback: Step-by-step guided flow instead of v2's overwhelming "all options at once" page.
- Real-time player state via WebSocket: Instant UI updates when child presses physical button or scans card. Parents see exactly what the box is doing.
- Pre-configured stream catalog: Browse ARD/NDR Audiothek, Deutschlandfunk, etc. with one tap. Huge value for German market vs manual URL entry.
- Gyro configuration in Web UI: Configure phonie-gyro tilt gestures without SSH, sensitivity profiles, calibration trigger.

**Defer (Phase 2+):**
- Podcast RSS subscription (complex, can manually upload for now)
- YouTube/URL download via yt-dlp (was broken in v2, not expected)
- Resume position per card (requires backend tracking)
- Scheduled playback / volume schedule (requires backend timer service)
- Drag-and-drop playlist editing (polish, not MVP)

**Anti-features (explicitly NOT build):**
- Full metadata library (artist/album/genre parsing): Phoniebox content is folders of audiobooks, not a music collection. Folders ARE the organization.
- Equalizer/DSP settings: Parents don't care about FLAC vs MP3. Fixed "good enough" audio.
- Multi-room sync: One box for one child. Sonos complexity unneeded.
- Streaming service browse UI: Spotify Connect only (let parents use official apps to cast).

### Architecture Approach

**API Gateway Pattern with Event Bridge** — the FastAPI server acts as both REST gateway and real-time event bus, bridging the modern SPA to existing shell scripts, MPD daemon, and config files. All existing daemons (RFID, GPIO, phonie-gyro) remain completely untouched. This is not a rewrite, it's a facade.

**Major components:**
1. **Nginx reverse proxy:** Serves static SvelteKit build, proxies `/api/` to FastAPI, proxies `/ws` WebSocket connections. Replaces Lighttpd.
2. **FastAPI API Gateway:** REST endpoints (`/api/player`, `/api/library`, `/api/cards`, `/api/settings`) and WebSocket hub (`/ws`). Wraps `playout_controls.sh` via subprocess (no `shell=True`), connects to MPD via `python-mpd2` asyncio, reads/writes `settings/` files.
3. **MPD Bridge (background task):** Subscribes to MPD idle protocol, broadcasts state changes to all connected WebSocket clients in real-time. Replaces 5-second polling with instant updates.
4. **Shell Bridge:** Typed Python wrapper around every `playout_controls.sh` command. Prevents command injection (subprocess argument lists, no string interpolation), provides type safety, enables mocking in tests.
5. **SvelteKit SPA (static build):** Player view, library browser, card management, settings. Service Worker for PWA "add to home screen." Built on dev machine, deployed as static files to Pi.

**Key architectural principle:** The existing shell scripts (`playout_controls.sh`, `rfid_trigger_play.sh`) remain the single source of truth for player commands. The API layer calls them, does not bypass them. This preserves integration with RFID daemon, GPIO daemon, and phonie-gyro which all route through these scripts. Only in Phase 3+ should we consider replacing shell scripts — and only if ALL consumers migrate simultaneously.

**Data flow example:** User presses "play" in browser → REST PUT to `/api/player` → FastAPI calls `subprocess.run(['playout_controls.sh', '-c=playerplay'])` → script calls `mpc play` → MPD state changes → MPD Bridge detects via idle → WebSocket push to all browsers → UI updates. Child places RFID card → daemon calls `rfid_trigger_play.sh` (unchanged) → MPD state changes → same MPD Bridge → same WebSocket push → UI updates. One event bus, multiple triggers.

### Critical Pitfalls

1. **SPA bundle too heavy for Pi's browser:** A typical React/MUI SPA produces 500KB-2MB bundles. On Pi 3B with 1GB RAM shared across MPD + daemons + web server, JS parsing takes 2-5 seconds and can trigger OOM killer. **Prevention:** Enforce 150KB gzipped budget via bundler config, choose Svelte over React (compiles away vs 45KB runtime), build custom Tailwind components (no MUI/Bootstrap), lazy-load admin pages, test on actual Pi hardware weekly.

2. **Breaking the shell script bridge:** New API talks directly to MPD via `python-mpd2`, bypassing `playout_controls.sh`. Now web UI and RFID/GPIO create split-brain state — volume systems fight, resume position lost, different control paths with different side effects. **Prevention:** Keep `playout_controls.sh` as single source of truth in Phase 1. API wraps it via subprocess. Document all 30+ commands it handles. Only replace in Phase 3+ after all consumers migrate together.

3. **Carrying over command injection vulnerabilities:** Current PHP uses `exec("sudo ...")` with unsanitized user input (card IDs, folder names, URLs). Easy to replicate: `subprocess.call(f"playout_controls.sh --cardid={cardid}", shell=True)` is vulnerable. **Prevention:** Never use `shell=True`. Use argument lists `subprocess.call(['/path/to/script', '-c=cmd'])`. Whitelist all command names. Validate card IDs as numeric-only (`^[0-9]+$`), folder names as alphanumeric+hyphens. Security boundary: one function sanitizes all shell calls.

4. **Polling MPD too aggressively (or not enough):** Either poll every 500ms (120 requests/min, overwhelms Pi CPU) or add WebSocket without understanding it requires a persistent server process (Lighttpd+PHP can't do this natively). **Prevention:** Phase 1 keeps 3-5 second polling with client-side interpolation (acceptable for music player). If adding real-time later: WebSocket via FastAPI background task subscribing to MPD's idle protocol (blocks until state changes, zero CPU when idle). Never poll from multiple components independently.

5. **Development-to-Pi deployment gap:** Build on Windows, test in Chrome, deploys to Pi and discovers Node.js not installed, Lighttpd URL rewriting breaks SPA routing, fetch() uses localhost:3000 instead of relative paths, fonts/images 5MB. **Prevention:** Establish build-and-deploy pipeline in Sprint 0 before any UI code. Use hash-based routing (`/#/settings`) instead of history API (works without server config). All API calls use relative paths. Test production build locally (`npx serve dist/`) before deploying. Pre-built static files committed to git (Pi never needs Node.js).

## Implications for Roadmap

Based on research, suggested phase structure follows technical dependencies and risk mitigation:

### Phase 1a: API Foundation (Build-Deploy Pipeline + Core Player API)
**Rationale:** Everything depends on the API layer working and being deployable. Establish this first before any UI work. The build-deploy gap (Pitfall 5) must be solved in Sprint 0.
**Delivers:** FastAPI project structure, shell service wrapper, MPD service wrapper, player REST endpoints (play/pause/next/prev/volume), build script that deploys to Pi via rsync/scp, Nginx configuration serving static files and proxying `/api/`.
**Addresses:** Technical foundation. No visible features yet.
**Avoids:** Development-to-Pi deployment gap (Pitfall 5). Tests subprocess wrapper pattern to avoid command injection (Pitfall 3).
**Research flag:** Standard patterns, skip `/gsd:research-phase`.

### Phase 1b: Real-Time Layer (WebSocket + MPD Bridge)
**Rationale:** WebSocket replaces wasteful polling (Pitfall 4) and is required for good parent UX (instant feedback when child uses physical controls). Depends on Phase 1a API foundation.
**Delivers:** WebSocket connection manager, MPD idle bridge (subscribes to MPD, broadcasts state changes), `/ws` endpoint, reconnection logic with exponential backoff.
**Addresses:** Real-time player state updates (differentiator feature).
**Avoids:** Polling MPD too aggressively (Pitfall 4).
**Research flag:** MPD idle protocol is well-documented standard, skip `/gsd:research-phase`.

### Phase 1c: Frontend Foundation (Player View)
**Rationale:** With API + WebSocket working, build the most-used screen (player view) first. Tests bundle size budget and mobile-first design immediately.
**Delivers:** SvelteKit project with adapter-static, WebSocket store (connects to `/ws`, manages reconnect), player component (NowPlaying, controls, volume slider, progress bar with seek, cover art), bottom navigation.
**Addresses:** Player controls with now-playing display (table stakes). Mobile-first responsive design (table stakes).
**Avoids:** SPA bundle too heavy (Pitfall 1 — enforce 150KB budget from day one). Ignoring mobile-first (Pitfall 11 — design at 375px width). MPD state and web UI state diverge (Pitfall 12 — MPD is truth, frontend is display).
**Research flag:** Needs `/gsd:research-phase` for Svelte 5 + Vite setup, PWA manifest, WebSocket reconnection patterns.

### Phase 1d: Deployment & Polish (Full Stack on Pi)
**Rationale:** Get Phase 1a-c working end-to-end on actual Pi hardware. Smoke out any Pi-specific issues (performance, memory, I/O) before building more features.
**Delivers:** Systemd service for FastAPI, Nginx config deployed, static SvelteKit build deployed, sleep timer UI (bedtime essential), repeat/shuffle toggles, cover art caching.
**Addresses:** Sleep timer (table stakes for parents). Full production deployment.
**Avoids:** Forgetting sleep timer (Pitfall 15). Cover art path assumptions (Pitfall 16).
**Research flag:** Standard deployment, skip `/gsd:research-phase`.

### Phase 2: Library & Card Management
**Rationale:** Depends on Phase 1 player foundation. Parents need to browse library visually and assign cards — next most critical workflows after "play music."
**Delivers:** Library REST endpoints (browse folders, list files), folder grid UI with covers, card REST endpoints (list, register, edit, delete), RFID event bridge (file watch → WS), card registration wizard, card overview ("card wall").
**Addresses:** Library browsing with cover art grid (table stakes). RFID card management (table stakes). Card registration wizard (differentiator).
**Avoids:** RFID card registration workflow breaks (Pitfall 9 — preserve `latestID.txt` polling, document dual-trigger problem, add registration mode in Phase 3).
**Research flag:** RFID file-watch integration might need `/gsd:research-phase` for latency tuning.

### Phase 3: File Upload & Settings
**Rationale:** Depends on Phase 1 foundation. Upload is deferred from MVP because it's complex (Pitfall 6) and parents can use SMB/SCP initially. Settings enable customization.
**Delivers:** Chunked file upload endpoint (tus protocol or resumable.js), upload UI with drag-drop and progress, disk space checks, settings REST endpoints (read/write config files), settings UI (max volume, sleep timer config, idle shutdown, WiFi, language), system info page.
**Addresses:** File upload (deferred from Phase 1). Settings & configuration (table stakes).
**Avoids:** File upload crashes on limited storage/memory (Pitfall 6 — chunked upload, disk space checks, ionice for low I/O priority).
**Research flag:** Chunked upload protocol needs `/gsd:research-phase` (tus.io or resumable.js pattern, Pi I/O performance testing).

### Phase 4: Access Tiers & Security
**Rationale:** Can start in parallel with Phase 2-3 but only integrate after those are done. Auth wraps existing features.
**Delivers:** PIN storage (bcrypt hash), JWT middleware with tier claims (open/parent/expert), PIN dialog UI, tier enforcement on routes, input validation layer (prevent command injection).
**Addresses:** Three-tier access model (differentiator). Security hardening.
**Avoids:** Over-engineering authentication (Pitfall 7 — simple PIN, not user accounts). Carrying over command injection (Pitfall 3 — validation layer, whitelist commands).
**Research flag:** Standard JWT + bcrypt, skip `/gsd:research-phase`.

### Phase 5: Gyro Integration & Streams
**Rationale:** Depends on Phase 3 settings UI. Gyro config exposes phonie-gyro settings through web interface. Stream catalog adds content acquisition convenience.
**Delivers:** Gyro config UI (sensitivity profiles, calibration trigger, gesture visualization), stream catalog (pre-configured ARD/NDR/DLF stations), stream assignment to cards.
**Addresses:** Gyro configuration in Web UI (differentiator). Pre-configured stream catalog (differentiator).
**Avoids:** Breaking phonie-gyro integration (Phase 2 warning — it calls `mpc` directly, verify still works).
**Research flag:** phonie-gyro config format needs review but likely standard YAML/JSON, skip `/gsd:research-phase`.

### Phase 6: PWA & Advanced Features
**Rationale:** Polish phase after core functionality complete. PWA enables "add to home screen" for app-like feel. Resume position and scheduled playback are nice-to-haves.
**Delivers:** Service Worker registration (network-first for API, cache-first for app shell), PWA manifest + icons, resume position per card, scheduled playback (alarm clock mode), volume schedule (quiet hours).
**Addresses:** PWA / Add to Home Screen (differentiator). Resume position (differentiator). Scheduled playback (differentiator).
**Avoids:** PWA/Service Worker confusion (Pitfall 8 — network-first for API, no caching API responses, clear update strategy).
**Research flag:** Scheduled playback needs cron/systemd timer integration, might need `/gsd:research-phase`.

### Phase Ordering Rationale

- **Foundation first (1a-d):** Build-deploy pipeline, API layer, WebSocket, player view MUST work before anything else. Tests all critical pitfalls (bundle size, shell bridge, deployment gap) early.
- **MVP completes with Phase 1:** Player + library browsing + basic card management (Phase 2) is minimum viable. Parents can control playback, browse visually, assign cards. Matches v2 parity for core use cases.
- **Upload deferred to Phase 3:** Complex (chunked upload, disk space, I/O priority), and SMB/SCP workaround exists. Not blocking for MVP.
- **Auth wraps features (Phase 4):** Can develop in parallel with Phase 2-3, integrate after. PIN protection is important but not blocking for single-user testing.
- **Gyro/Streams/PWA are polish (Phase 5-6):** Differentiators that make product great, but core value (parents control music box from phone) works without them.

**Dependency chain:** 1a → 1b → 1c → 1d (sequential, critical path). Then 2, 3, 4 can partially overlap. Then 5, 6 are independent polish.

### Research Flags

**Phases needing `/gsd:research-phase`:**
- **Phase 1c (Frontend Foundation):** Svelte 5 + SvelteKit + Vite setup, PWA manifest patterns, WebSocket reconnection strategies. Framework setup is well-documented but Svelte 5 runes reactivity model might have changed since training cutoff.
- **Phase 3 (File Upload):** Chunked upload protocols (tus.io vs resumable.js), Pi I/O performance testing with ionice, disk space monitoring patterns. Upload on resource-constrained devices has many edge cases.
- **Phase 6 (Scheduled Playback):** Cron vs systemd timer integration for alarm clock mode, Python APScheduler patterns, time zone handling. Scheduling on embedded Linux has many approaches.

**Phases with standard patterns (skip research-phase):**
- **Phase 1a (API Foundation):** FastAPI project structure, subprocess wrappers, Nginx reverse proxy config — all well-documented standard patterns.
- **Phase 1b (Real-Time Layer):** MPD idle protocol (standard since MPD 0.14), FastAPI WebSocket (well-documented), python-mpd2 library (already used in v2).
- **Phase 1d (Deployment):** Systemd service files, rsync/scp deploy scripts — standard Linux deployment.
- **Phase 2 (Library & Cards):** File system browsing, REST CRUD endpoints, file-watch patterns — standard backend work.
- **Phase 4 (Access Tiers):** JWT + bcrypt, FastAPI dependency injection for auth — very standard patterns.
- **Phase 5 (Gyro & Streams):** Config file reading/writing, static data catalogs — standard.

## Confidence Assessment

| Area | Confidence | Notes |
|------|------------|-------|
| Stack | MEDIUM | Svelte 5, FastAPI 0.115+, Tailwind 4 versions based on training data (Jan 2025 cutoff). Core choices (Svelte over React, FastAPI over Flask, Nginx over Lighttpd) are sound and well-reasoned, but version numbers need verification against current releases. python-mpd2 is HIGH confidence (already in v2 codebase). |
| Features | MEDIUM | Competitive analysis (Toniebox, Volumio, moOde, Sonos) based on training data. Core features unlikely changed, but specific UI patterns may have evolved. Phoniebox v2 feature inventory is HIGH confidence (directly analyzed 77 PHP files). |
| Architecture | HIGH | Based on deep codebase analysis of v2 integration points (shell scripts, MPD socket, config files, RFID daemon). API Gateway pattern is standard and well-suited. MPD idle protocol is HIGH confidence (standard feature). Only MEDIUM on RFID event forwarding (file-watch untested, may need latency tuning). |
| Pitfalls | MEDIUM to HIGH | Command injection vulnerabilities HIGH confidence (directly observed in code). Bundle size concerns HIGH confidence (established Pi 3B limitations). Polling strategy HIGH confidence (current code does 5s polling). Deployment gap HIGH confidence (common Windows-to-Pi issue). PWA caching and auth over-engineering are MEDIUM confidence (inferred from common mistakes, not project-specific). |

**Overall confidence:** MEDIUM

Research is solid for making architectural decisions and avoiding known pitfalls. The recommended stack is well-reasoned based on Pi constraints and existing v2 integration points. However, WebSearch/WebFetch were unavailable — version numbers for Svelte 5, FastAPI, Tailwind 4, SvelteKit 2 are from training data (Jan 2025 cutoff) and MUST be verified before implementation. Competitive feature analysis relies on training knowledge of Toniebox/Volumio/moOde/Sonos which may have evolved.

### Gaps to Address

**Version verification (before Phase 1a):**
- Verify current stable versions: `svelte --version`, `pip index versions fastapi`, check https://svelte.dev, https://fastapi.tiangolo.com
- Confirm Tailwind v4 + DaisyUI v5 compatibility (v4 was early 2025 release, may have breaking changes)
- Verify SvelteKit adapter-static current capabilities and PWA plugin compatibility
- Check python-mpd2 asyncio support (listed in v2's `requirements-GPIO.txt` but verify async API)

**Pi hardware testing (before Phase 1c complete):**
- Actual bundle size test on Pi: load SvelteKit build in Pi's Chromium, measure JS parse time with slow-3G throttling
- Memory pressure test: FastAPI + Uvicorn + Nginx + MPD + RFID daemon + GPIO daemon + phonie-gyro all running, measure total RAM usage
- I/O performance test: Large file upload over WiFi while MPD is playing, measure stuttering/dropped frames
- WebSocket connection limit: How many concurrent browser sessions before Pi struggles? (Expect 1-3, but verify)

**RFID event forwarding (during Phase 2):**
- File-watch latency: How fast does the API detect `latestID.txt` change? Is inotify/watchdog fast enough or does it need polling?
- Registration mode implementation: Modify `daemon_rfid_reader.py` to check for `settings/registration_mode` flag file without breaking existing card-trigger flow. Needs careful testing.

**Chunked upload (during Phase 3):**
- Choose protocol: tus.io (resumable upload standard) vs resumable.js (library-specific). Research during Phase 3 planning.
- Disk space monitoring: Best pattern for checking available space before accepting upload chunk? `shutil.disk_usage()` every chunk or once at start?
- ionice effectiveness: Does `ionice -c3` actually prevent MPD stuttering during upload on Pi 3B? Needs real-world test.

**Spotify integration (deferred beyond Phase 6):**
- Spotify is an industry-wide unsolved problem for OSS (libspotify discontinued, no official API for playback). Current v2 Spotify is broken (pinned on Mopidy-Spotify alpha, password auth removed by Spotify). Best approach: Spotify Connect (box appears as speaker in official Spotify app). Research if/when prioritized — likely a separate project.

## Sources

### Primary (HIGH confidence)
- **Phoniebox v2.8.0 codebase:** Direct analysis of `htdocs/` (77 PHP files), `scripts/playout_controls.sh` (1,153 lines), `scripts/daemon_rfid_reader.py`, `components/gpio_control/`, `requirements.txt`, `packages.txt`, `misc/sampleconfigs/lighttpd.conf-default.sample`. All integration points, command flows, and vulnerabilities verified from source code.
- **Existing project analysis:** `.planning/ANALYSIS.md` (452 lines, v2 vs v3 evaluation, strategy decision), `.planning/codebase/` (7 documents, 1,687 lines total: ARCHITECTURE.md, STACK.md, CONCERNS.md, INTEGRATIONS.md, FEATURES.md, SECURITY.md, TESTING.md).
- **phonie-gyro project:** User's own project at https://github.com/t13gazh/phonie-gyro.git, verified working MPU6050 integration calling `mpc` directly.
- **python-mpd2:** Listed in v2's `requirements-GPIO.txt`, confirmed usage in `components/gpio_control/GPIOControl.py`.

### Secondary (MEDIUM confidence)
- **Technology stack knowledge (training data through Jan 2025):** Svelte 5 (runes reactivity), FastAPI 0.115+ (WebSocket support), Tailwind CSS 4 (released early 2025), SvelteKit 2 (adapter-static), Vite 6, Nginx 1.26+, PyJWT 2.x, pnpm 9.x. Core capabilities accurate, version numbers need current verification.
- **Competitive landscape:** Toniebox/mytonies app (iOS/Android), Volumio 3 web interface, moOde Audio Player 8.x, Sonos S2 app. Based on training knowledge of established products — core features stable, specific UIs may have evolved.
- **Raspberry Pi 3B hardware:** 1GB RAM, quad-core ARM Cortex 1.2-1.4GHz, typical memory distribution for Raspberry Pi OS + services. Well-established specifications.
- **MPD idle protocol:** Standard MPD feature since 0.14, well-documented at https://www.musicpd.org/doc/protocol/. Zero-CPU-cost state change subscription.

### Tertiary (LOW confidence, needs validation)
- **DaisyUI v5 compatibility with Tailwind 4:** DaisyUI v5 may or may not exist yet. Tailwind 4 was early 2025 release. Verify before using DaisyUI or consider plain Tailwind only.
- **Svelte i18n libraries (paraglide-js vs svelte-i18n):** Need to verify current state of Svelte 5 i18n ecosystem. paraglide-js compile-time approach preferred but compatibility unknown.
- **Lighttpd WebSocket proxy limitations:** Based on general knowledge that Lighttpd has poor/hacky WebSocket support vs Nginx's native proxy. Need to verify if recent Lighttpd versions improved (unlikely but check).

---
*Research completed: 2026-02-06*
*Ready for roadmap: yes*

# Architecture Patterns

**Domain:** Embedded web UI for Raspberry Pi music player (Phoniebox modernization)
**Researched:** 2026-02-06
**Overall confidence:** HIGH (based on deep codebase analysis + established architecture patterns)

---

## Recommended Architecture

### Overview: API Gateway Pattern with Event Bridge

Replace the PHP/Lighttpd layer with a Python API server that acts as both REST gateway and real-time event bus. The frontend becomes a standalone SPA/PWA served as static files. All existing daemons (RFID, GPIO, gyro, MPD) remain untouched.

```
                    BROWSER (Phone/Tablet/Desktop)
                    +---------------------------------+
                    |  SvelteKit SPA / PWA            |
                    |  (Static files served by Nginx) |
                    |  Service Worker for offline      |
                    +-----------+---------------------+
                                |
                    REST API    |    WebSocket
                    (commands)  |    (real-time state)
                                |
                    +-----------v---------------------+
                    |  API Gateway (FastAPI/Python)    |
                    |                                  |
                    |  +----------------------------+  |
                    |  | REST Routes                |  |
                    |  | /api/player/*              |  |
                    |  | /api/library/*             |  |
                    |  | /api/cards/*               |  |
                    |  | /api/system/*              |  |
                    |  | /api/upload/*              |  |
                    |  +----------------------------+  |
                    |                                  |
                    |  +----------------------------+  |
                    |  | WebSocket Hub              |  |
                    |  | Player state (from MPD)    |  |
                    |  | RFID events (from daemon)  |  |
                    |  | Gyro events (from phonie-  |  |
                    |  |   gyro via FIFO/socket)    |  |
                    |  +----------------------------+  |
                    |                                  |
                    |  +----------------------------+  |
                    |  | Auth Middleware            |  |
                    |  | PIN-based access tiers     |  |
                    |  +----------------------------+  |
                    +--+--------+--------+--------+---+
                       |        |        |        |
            +----------+  +-----+--+ +---+----+ +-+--------+
            |             |        | |        | |          |
     +------v------+ +---v----+ +-v-+----+ +-v-+------+ +-v--------+
     |playout_     | | MPD    | |Config  | |File      | |Settings  |
     |controls.sh  | | :6600  | |Files   | |System    | |Files     |
     |(subprocess) | |(socket)| |global  | |audio     | |50+ files |
     +-------------+ +--------+ |.conf   | |folders   | +----------+
                                +--------+ +----------+

     EXISTING DAEMONS (unchanged, running as systemd services):
     +------------------+  +------------------+  +------------------+
     | daemon_rfid_     |  | gpio_control.py  |  | phonie-gyro      |
     | reader.py        |  | (buttons/rotary) |  | (MPU6050 gyro)   |
     | (polls RFID)     |  | (listens GPIO)   |  | (gesture detect) |
     +--------+---------+  +--------+---------+  +--------+---------+
              |                      |                     |
              v                      v                     v
     calls rfid_trigger_    calls playout_         calls mpc
     play.sh directly       controls.sh            (MPD client)
```

### Why This Architecture

1. **Python API server bridges all worlds.** The existing backend is shell scripts, MPD socket, and file-based config. Python can call subprocess, open TCP sockets, and read files natively. No language impedance mismatch.

2. **WebSocket solves the polling problem.** The current UI polls `player.php` every 5 seconds. A WebSocket connection to the API server can push MPD state changes in real-time (MPD supports idle/subscribe protocol).

3. **Static SPA decouples frontend completely.** The new frontend has zero runtime dependency on the server technology. It consumes a REST API and a WebSocket. This means the frontend can be developed, built, and tested entirely on Windows with mock data.

4. **Existing daemons remain untouched.** The RFID daemon, GPIO control, and phonie-gyro all interact with `playout_controls.sh` or `mpc` directly. They never touch the web layer. This architecture preserves that boundary perfectly.

5. **Nginx replaces Lighttpd.** Nginx is better at reverse-proxying (needed for WebSocket upgrade) and serving static files. It also enables future HTTPS with Let's Encrypt.

---

## Component Boundaries

| Component | Responsibility | Input | Output | Communicates With |
|-----------|---------------|-------|--------|-------------------|
| **Nginx** | Reverse proxy + static file server | HTTP/WS requests | Routed to API or static files | Browser, API Gateway |
| **SvelteKit SPA** | UI rendering, user interaction | User touch/click, WebSocket events | REST API calls, WS subscriptions | API Gateway (via Nginx) |
| **Service Worker** | Offline caching, PWA install | Network requests | Cached responses | Browser cache, Nginx |
| **API Gateway (FastAPI)** | REST endpoints, WS hub, auth | HTTP/WS from browser, MPD events, RFID events | Commands to shell/MPD, state to browser | Nginx, MPD, Shell scripts, Config files |
| **MPD Bridge** | Subscribe to MPD idle, translate state | MPD idle events on :6600 | Player state objects to WS hub | MPD daemon, API Gateway |
| **Shell Bridge** | Execute playout_controls.sh | Command + args from REST | Exit code + stdout | playout_controls.sh subprocess |
| **Config Manager** | Read/write settings files | REST requests | Config objects, file writes | settings/ directory |
| **File Manager** | Upload, list, delete audio files | Multipart uploads, REST | File operations on shared/audiofolders/ | Filesystem |
| **RFID Event Listener** | Detect card scans, forward to WS | RFID daemon events | Card ID events to WS hub | daemon_rfid_reader.py (via FIFO or file watch) |
| **Auth Module** | PIN verification, tier enforcement | PIN from browser | Session token (JWT or cookie) | API Gateway middleware |

### Component Ownership Rules

**New code we write:**
- Nginx configuration
- SvelteKit frontend (SPA + PWA)
- FastAPI server (REST + WebSocket)
- Auth middleware
- Config manager (replaces PHP config reading)

**Existing code we do NOT modify:**
- `daemon_rfid_reader.py` -- RFID polling loop
- `gpio_control.py` -- GPIO button handling
- `phonie-gyro` -- Gyro gesture detection
- `playout_controls.sh` -- Central command dispatcher (1153 lines)
- `rfid_trigger_play.sh` -- Card-to-folder resolver
- `inc.writeGlobalConfig.sh` -- Config merger
- MPD configuration and daemon

---

## Data Flow

### Flow 1: User Presses Play in Browser

```
Browser                  Nginx              API Gateway          Shell               MPD
  |                        |                     |                 |                  |
  |-- PUT /api/player ---->|                     |                 |                  |
  |   {"command":"play"}   |-- proxy ----------->|                 |                  |
  |                        |                     |                 |                  |
  |                        |                     |-- subprocess -->|                  |
  |                        |                     |   playout_      |                  |
  |                        |                     |   controls.sh   |                  |
  |                        |                     |   -c=playerplay |-- mpc play ----->|
  |                        |                     |                 |                  |
  |                        |                     |<-- exit 0 ------|                  |
  |                        |<-- 200 OK ----------|                 |                  |
  |<-- 200 OK -------------|                     |                 |                  |
  |                        |                     |                 |                  |
  |                        |   MPD Bridge        |                 |                  |
  |                        |   (background)      |                 |                  |
  |                        |                     |<---- idle:player ------------------|
  |                        |                     |                 |                  |
  |                        |                     |-- query status ->|                  |
  |                        |                     |   (TCP :6600)   |                  |
  |                        |                     |<- state:play ----|                  |
  |                        |                     |                 |                  |
  |<====== WebSocket push: {"state":"play","song":"..."} =========|                  |
  |        (player state)  |                     |                 |                  |
```

### Flow 2: Child Places RFID Card

```
RFID Reader        daemon_rfid_reader.py      rfid_trigger_play.sh      MPD
  |                       |                          |                    |
  |-- card detected ----->|                          |                    |
  |   (polling loop)      |                          |                    |
  |                       |-- subprocess.call ------->|                    |
  |                       |   --cardid=1234567890    |                    |
  |                       |                          |-- lookup card      |
  |                       |                          |   in global.conf   |
  |                       |                          |                    |
  |                       |                          |-- playout_controls |
  |                       |                          |   .sh -c=play...   |
  |                       |                          |                    |
  |                       |                          |-- mpc load/play -->|
  |                       |                          |                    |
  |                       |                          |                    |

API Gateway (running independently, subscribed to MPD idle):
  |                                                                       |
  |<================ MPD idle:player event ==============================|
  |                                                                       |
  |-- query status (TCP :6600) ----------------------------------------->|
  |<-- state:play, song:... --------------------------------------------|
  |                                                                       |
  |====== WebSocket push to all connected browsers =====================>|
  |  {"event":"player","state":"play","song":"Die drei ???"}              |
```

**Key insight:** The API server does NOT need to know about RFID card scans directly. It subscribes to MPD idle events. When any source (RFID, GPIO, gyro, web UI) causes MPD state to change, the API server detects it and pushes to all browsers. This is the fundamental decoupling principle.

### Flow 3: RFID Card Scan Notification to Browser (Optional Enhancement)

For the card management UI, we DO want to know when a card is scanned (e.g., "hold card to reader, then assign folder"). This requires a separate event channel from the RFID daemon:

**Option A: File Watch (simplest, recommended for Phase 1)**
```
daemon_rfid_reader.py writes cardid to a known file (it already does this)
API Gateway watches file with inotify/watchdog
On change: push {"event":"rfid","cardid":"1234567890"} to WS
```

**Option B: Named FIFO (better for Phase 3+)**
```
Create /tmp/phoniebox-rfid-events FIFO
Modify daemon_rfid_reader.py to also write to FIFO
API Gateway reads from FIFO in background task
On read: push {"event":"rfid","cardid":"..."} to WS
```

**Option C: Shared SQLite + polling (for card management)**
```
RFID daemon writes last-seen card to settings file (already happens)
API Gateway polls this file every 500ms only when card-registration page is open
Minimizes changes to existing daemon
```

Recommendation: Start with Option A (file watch), migrate to Option B if latency matters.

### Flow 4: File Upload

```
Browser                  Nginx              API Gateway          Filesystem
  |                        |                     |                    |
  |-- POST /api/upload --->|                     |                    |
  |   multipart/form-data  |-- proxy ----------->|                    |
  |   (audio file chunks)  |                     |                    |
  |                        |                     |-- validate auth    |
  |                        |                     |   (admin tier)     |
  |                        |                     |                    |
  |                        |                     |-- stream to ------>|
  |                        |                     |   tmp file         |
  |                        |                     |                    |
  |                        |                     |-- validate format  |
  |                        |                     |   (ffprobe)        |
  |                        |                     |                    |
  |                        |                     |-- move to -------->|
  |                        |                     |   audiofolders/    |
  |                        |                     |                    |
  |                        |                     |-- mpc update ----->|
  |                        |                     |   (refresh MPD DB) |
  |                        |                     |                    |
  |<-- 201 Created --------|<-- 201 -------------|                    |
```

### Flow 5: Real-Time Player State (MPD Bridge Detail)

```python
# Pseudocode for MPD Bridge (runs as FastAPI background task)
async def mpd_bridge():
    mpd = MPDClient()
    mpd.connect("localhost", 6600)

    while True:
        # MPD idle blocks until state changes
        changes = await mpd.idle()  # returns: ['player', 'mixer', 'playlist', etc.]

        if 'player' in changes or 'mixer' in changes:
            status = mpd.status()
            current = mpd.currentsong()

            state = {
                "event": "player",
                "state": status["state"],      # play/pause/stop
                "volume": status["volume"],
                "elapsed": status.get("elapsed"),
                "duration": status.get("duration"),
                "song": current.get("title"),
                "artist": current.get("artist"),
                "album": current.get("album"),
                "file": current.get("file"),
                "repeat": status.get("repeat"),
                "single": status.get("single"),
            }

            # Push to all connected WebSocket clients
            await websocket_hub.broadcast(state)
```

This replaces the current 5-second polling with instant updates. MPD's `idle` command is designed exactly for this purpose -- it blocks until something changes, then returns what changed.

---

## Detailed Component Architecture

### 1. Nginx Configuration

```
server {
    listen 80;
    server_name phoniebox.local;

    # Serve SvelteKit build output as static files
    location / {
        root /home/pi/phoniebox-ui/build;
        try_files $uri $uri/ /index.html;  # SPA fallback
    }

    # Proxy API requests to FastAPI
    location /api/ {
        proxy_pass http://127.0.0.1:8000;
        proxy_set_header Host $host;
    }

    # WebSocket upgrade for real-time
    location /ws {
        proxy_pass http://127.0.0.1:8000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }

    # Serve audio folder covers directly (bypass API for performance)
    location /covers/ {
        alias /home/pi/RPi-Jukebox-RFID/shared/audiofolders/;
        try_files $uri/cover.jpg /static/no-cover.jpg;
    }

    # Legacy PHP compatibility (Phase 1 transition)
    location /legacy/ {
        alias /home/pi/RPi-Jukebox-RFID/htdocs/;
        # Keep old PHP working during migration
    }
}
```

**Why Nginx, not Lighttpd:** Lighttpd cannot reverse-proxy WebSocket connections properly. Nginx handles HTTP, WS, and static files in one config. Lighttpd is still fine for serving PHP, but we are eliminating PHP.

### 2. FastAPI Server Structure

```
phoniebox-api/
  main.py                    # FastAPI app, startup/shutdown events
  config.py                  # App configuration (paths, ports)

  routers/
    player.py                # GET/PUT /api/player (play, pause, next, etc.)
    volume.py                # GET/PUT /api/volume
    library.py               # GET /api/library (browse folders)
    playlist.py              # GET/PUT /api/playlist
    cards.py                 # GET/PUT/DELETE /api/cards (RFID management)
    upload.py                # POST /api/upload (file upload)
    system.py                # GET/POST /api/system (shutdown, wifi, etc.)
    settings.py              # GET/PUT /api/settings (configuration)
    gyro.py                  # GET/PUT /api/gyro (phonie-gyro config)

  services/
    mpd_service.py           # MPD socket communication (python-mpd2)
    shell_service.py         # subprocess wrapper for playout_controls.sh
    config_service.py        # Read/write settings/ files
    file_service.py          # Audio file management
    rfid_service.py          # RFID event monitoring (file watch)
    auth_service.py          # PIN verification, session management

  websocket/
    hub.py                   # WebSocket connection manager
    mpd_bridge.py            # MPD idle -> WS broadcast
    rfid_bridge.py           # RFID events -> WS broadcast

  models/
    player.py                # Pydantic models for player state
    card.py                  # Pydantic models for RFID cards
    settings.py              # Pydantic models for configuration

  middleware/
    auth.py                  # Access tier enforcement
```

**Key design decisions:**

- **`shell_service.py` wraps playout_controls.sh.** Every command the current PHP API sends to the shell goes through this service. This preserves the existing command interface exactly.
- **`mpd_service.py` uses python-mpd2.** This is the same library used in v2's GPIO control. It provides a proper Python client for MPD's protocol, replacing raw socket code in `common.php`.
- **`config_service.py` reads the same files.** It reads `settings/global.conf`, `settings/rfid_trigger_play.conf`, and individual setting files. No config format changes needed.

### 3. SvelteKit Frontend Structure

```
phoniebox-ui/
  src/
    routes/
      +layout.svelte          # Root layout (nav, WS connection)
      +page.svelte             # Player view (default/home)
      library/
        +page.svelte           # Browse audio folders
      cards/
        +page.svelte           # RFID card management
        register/
          +page.svelte         # Card registration flow
      settings/
        +page.svelte           # Admin settings (PIN-protected)
      gyro/
        +page.svelte           # Gyro configuration

    lib/
      stores/
        player.ts              # Player state store (from WebSocket)
        library.ts             # Audio library store
        cards.ts               # Card registry store
        auth.ts                # Auth state store
        websocket.ts           # WebSocket connection + reconnect

      components/
        Player/
          PlayerControls.svelte     # Play/Pause/Next/Prev buttons
          VolumeSlider.svelte       # Volume control
          ProgressBar.svelte        # Track progress with seek
          NowPlaying.svelte         # Current track info + cover
          ChapterNavigation.svelte  # Chapter skip (audiobooks)
        Library/
          FolderGrid.svelte         # Grid of audio folders
          FolderCard.svelte         # Single folder with cover
          FileList.svelte           # Files within folder
          AudioUpload.svelte        # Drag-and-drop upload
        Cards/
          CardList.svelte           # All registered cards
          CardRegisterWizard.svelte # Step-by-step card registration
          CardAssignmentPicker.svelte # Assign folder/file/command
        Settings/
          GeneralSettings.svelte    # Volume limits, idle timer, etc.
          GyroSettings.svelte       # phonie-gyro configuration
          SystemInfo.svelte         # CPU temp, disk usage, uptime
          WifiSettings.svelte       # Network configuration
        Layout/
          BottomNav.svelte          # Mobile bottom navigation
          PinDialog.svelte          # PIN entry overlay

      api/
        client.ts              # Fetch wrapper with auth headers
        types.ts               # TypeScript interfaces matching API

      sw/
        service-worker.ts      # Service Worker for offline + PWA

  static/
    manifest.json              # PWA manifest
    icons/                     # App icons for home screen

  svelte.config.js             # SvelteKit config (adapter-static)
  vite.config.js               # Vite dev server with API proxy
```

**Why SvelteKit with adapter-static:**
- Build output is pure static HTML/JS/CSS -- no Node.js runtime on Pi
- Vite dev server can proxy `/api/` to FastAPI during development
- Excellent PWA support through `@vite-pwa/sveltekit`
- Svelte's reactivity model naturally fits real-time player state updates
- Smaller bundle size than React (critical on Pi -- faster load on low-power device)

### 4. WebSocket Protocol

All real-time communication uses a single WebSocket at `/ws`.

**Server -> Client messages:**

```json
// Player state update (from MPD bridge)
{
  "type": "player",
  "data": {
    "state": "play",
    "volume": 75,
    "elapsed": 42.3,
    "duration": 180.0,
    "song": "Die drei Fragezeichen - Folge 1",
    "artist": "Die drei ???",
    "file": "die-drei-fragezeichen/folge-01/track01.mp3",
    "repeat": "0",
    "single": "0",
    "playlist_length": 12,
    "song_position": 3
  }
}

// RFID card event (from RFID bridge)
{
  "type": "rfid",
  "data": {
    "cardid": "1234567890",
    "timestamp": "2026-02-06T15:30:00Z"
  }
}

// System event
{
  "type": "system",
  "data": {
    "event": "shutdown_scheduled",
    "in_seconds": 300
  }
}
```

**Client -> Server messages:**

```json
// Subscribe to specific event types
{
  "type": "subscribe",
  "channels": ["player", "rfid"]
}

// Heartbeat/keepalive
{
  "type": "ping"
}
```

**Reconnection strategy:**
- Initial connect on page load
- On disconnect: retry at 1s, 2s, 4s, 8s, 16s (exponential backoff, max 30s)
- On reconnect: request full state snapshot
- Service Worker maintains last-known state for offline display

### 5. Authentication / Access Tiers

Three access tiers, enforced at API middleware level:

| Tier | Access | Auth Required | Endpoints |
|------|--------|---------------|-----------|
| **Open** | Player control, now playing | None | `GET/PUT /api/player`, `GET /api/playlist`, `/ws` |
| **Parent** | Card management, library, upload | 4-digit PIN | `GET/PUT/DELETE /api/cards`, `POST /api/upload`, `GET/PUT /api/library` |
| **Expert** | System settings, shutdown, wifi | Different PIN | `GET/PUT /api/settings`, `POST /api/system/*`, `GET/PUT /api/gyro` |

**Implementation:**
- PIN stored as bcrypt hash in `settings/pin_parent.hash` and `settings/pin_expert.hash`
- On PIN entry: server returns a JWT (short-lived, 24h) stored in localStorage
- JWT contains tier claim: `{"tier": "parent", "exp": ...}`
- FastAPI dependency injection checks tier on protected routes
- No external auth provider needed -- this is a local network device

**Why JWT over session cookie:** The Pi may restart. JWTs are self-contained (no server-side session store needed). If the Pi reboots, existing tokens still validate.

**Why NOT full user accounts:** This is a children's music box. Parents want a PIN, not username/password/email. One PIN for parent mode, one for expert mode. Simple.

---

## Patterns to Follow

### Pattern 1: Command Adapter (Shell Bridge)

**What:** Wrap every `playout_controls.sh` command in a typed Python function. The new API calls these functions instead of constructing shell strings.

**When:** For every player control action.

**Why:** Prevents command injection, provides type safety, enables mocking in tests.

```python
# services/shell_service.py
import subprocess
from pathlib import Path

SCRIPTS_DIR = Path("/home/pi/RPi-Jukebox-RFID/scripts")

class ShellService:
    """Typed wrapper around playout_controls.sh commands."""

    ALLOWED_COMMANDS = {
        "playerplay", "playerpause", "playerstop", "playernext",
        "playerprev", "volumeup", "volumedown", "setvolume",
        "getvolume", "mute", "shutdown", "reboot",
        # ... all commands from playout_controls.sh header
    }

    def execute(self, command: str, value: str | None = None) -> str:
        if command not in self.ALLOWED_COMMANDS:
            raise ValueError(f"Unknown command: {command}")

        args = [str(SCRIPTS_DIR / "playout_controls.sh"), f"-c={command}"]
        if value is not None:
            args.append(f"-v={value}")

        result = subprocess.run(
            args,
            capture_output=True,
            text=True,
            timeout=10,
        )
        return result.stdout.strip()

    def player_play(self) -> None:
        self.execute("playerplay")

    def set_volume(self, level: int) -> None:
        if not 0 <= level <= 100:
            raise ValueError(f"Volume must be 0-100, got {level}")
        self.execute("setvolume", str(level))
```

**Critical:** Note `shell=False` (the default with list args). This eliminates the command injection vulnerability present in both the PHP layer (`exec("sudo ...")`) and the RFID daemon (`shell=True`).

### Pattern 2: MPD Reactive Bridge

**What:** A long-running async task that subscribes to MPD's idle protocol and broadcasts state changes via WebSocket.

**When:** Always running as a FastAPI background task from application startup.

**Why:** Replaces 5-second polling with instant updates. MPD's idle command is zero-CPU-cost when nothing changes.

```python
# websocket/mpd_bridge.py
import asyncio
from mpd.asyncio import MPDClient

class MPDBridge:
    def __init__(self, hub: WebSocketHub):
        self.hub = hub
        self.client = MPDClient()

    async def run(self):
        await self.client.connect("localhost", 6600)

        while True:
            try:
                # This blocks until MPD state changes
                changes = await self.client.idle(
                    "player", "mixer", "playlist", "options"
                )

                status = await self.client.status()
                current = await self.client.currentsong()

                await self.hub.broadcast({
                    "type": "player",
                    "data": self._format_state(status, current)
                })

            except Exception as e:
                logger.error(f"MPD bridge error: {e}")
                await asyncio.sleep(2)
                await self.client.connect("localhost", 6600)
```

### Pattern 3: Layered Configuration Reader

**What:** Read v2's file-based config through a Python service that presents a clean interface, without changing the file format.

**When:** For any API endpoint that needs configuration data.

**Why:** The 50+ settings files in `settings/` are the source of truth. Changing them would break the existing daemons. Instead, read them as-is.

```python
# services/config_service.py
from pathlib import Path
import configparser

SETTINGS_DIR = Path("/home/pi/RPi-Jukebox-RFID/settings")

class ConfigService:
    """Read v2 settings files without modifying their format."""

    def get_global_conf(self) -> dict:
        """Parse global.conf (shell-style KEY=VALUE)."""
        config = {}
        conf_file = SETTINGS_DIR / "global.conf"
        if conf_file.exists():
            for line in conf_file.read_text().splitlines():
                line = line.strip()
                if "=" in line and not line.startswith("#"):
                    key, _, value = line.partition("=")
                    config[key.strip()] = value.strip().strip('"')
        return config

    def get_setting(self, name: str) -> str:
        """Read a single settings file (e.g., 'Audio_Folders_Path')."""
        return (SETTINGS_DIR / name).read_text().strip()

    def set_setting(self, name: str, value: str) -> None:
        """Write a single settings file."""
        (SETTINGS_DIR / name).write_text(value + "\n")
        # Regenerate global.conf after any change
        self._rebuild_global_conf()

    def _rebuild_global_conf(self) -> None:
        """Call inc.writeGlobalConfig.sh to regenerate global.conf."""
        subprocess.run(
            [str(SETTINGS_DIR.parent / "scripts" / "inc.writeGlobalConfig.sh")],
            check=True,
        )
```

### Pattern 4: Static Build Deployment

**What:** Build the SvelteKit frontend on the development machine (Windows), deploy the static output to the Pi via rsync/scp.

**When:** Every frontend change.

**Why:** The Pi should not run Node.js, npm, or any build tooling. It only serves pre-built static files.

```
Development (Windows PC):
  cd phoniebox-ui
  npm run build           # Produces build/ directory
  scp -r build/ pi@phoniebox.local:/home/pi/phoniebox-ui/

Production (Raspberry Pi):
  Nginx serves /home/pi/phoniebox-ui/build/ as static files
  No Node.js installed on Pi
  No npm, no build step on Pi
```

### Pattern 5: Progressive Enhancement for Offline

**What:** Service Worker caches the app shell and last-known player state. When offline, the UI still loads and shows last-known state.

**When:** After initial load + authentication.

**Why:** The Pi might restart, WiFi might be flaky. The UI should always be usable.

```
Cache strategy:
  App shell (HTML/JS/CSS):  Cache-first (update in background)
  API responses:            Network-first, fall back to cache
  Cover images:             Cache-first with stale-while-revalidate
  WebSocket:                Reconnect with exponential backoff
  File uploads:             Queue in IndexedDB, retry when online
```

---

## Anti-Patterns to Avoid

### Anti-Pattern 1: Replacing playout_controls.sh with Python

**What:** Rewriting the 1153-line shell script in Python.

**Why bad:** The shell script is the integration point for ALL control paths (RFID daemon, GPIO daemon, phonie-gyro via `mpc`, web UI). Replacing it means modifying 4+ other components that call it. The script works. It is ugly but stable.

**Instead:** Wrap it. Call it via subprocess from Python. Let it continue being the central command dispatcher. Only replace individual commands if they are buggy or need new functionality.

### Anti-Pattern 2: WebSocket for Commands

**What:** Sending player commands (play, pause, volume) over WebSocket instead of REST.

**Why bad:** REST gives you HTTP status codes, request/response correlation, and middleware (auth, validation, rate limiting) for free. WebSocket messages are fire-and-forget -- you lose error handling semantics.

**Instead:** REST for commands (PUT /api/player with {"command":"play"}), WebSocket only for server-push events (player state, RFID events).

### Anti-Pattern 3: Server-Side Rendering on Pi

**What:** Using SSR frameworks (Next.js, Nuxt, SvelteKit in SSR mode) that require Node.js on the Pi.

**Why bad:** The Pi has limited RAM (1-4GB). Running Node.js + Python + MPD + Nginx is too much. Node.js SSR adds cold start latency on every navigation.

**Instead:** Use SvelteKit with `adapter-static`. Build on dev machine, deploy static files. Zero runtime on Pi.

### Anti-Pattern 4: Database for Configuration

**What:** Migrating the 50+ settings files to SQLite or another database.

**Why bad:** The existing daemons (RFID, GPIO, gyro) all read these files directly. Migrating to a database means rewriting ALL of them. The file format is the API contract between components.

**Instead:** Keep files as source of truth. The Python config service reads/writes them. If needed later, add a caching layer (in-memory dict refreshed on file change via inotify).

### Anti-Pattern 5: Polling MPD from Frontend

**What:** Having the frontend poll `/api/player` every N seconds (like the current jukebox.js does every 5s).

**Why bad:** Creates unnecessary CPU load on Pi, stale data between polls (child places card, parents see old state for up to 5 seconds), wasted bandwidth.

**Instead:** One persistent WebSocket connection. The API server subscribes to MPD idle (zero CPU when idle), pushes changes instantly. The frontend only calls REST on user action.

---

## Integration Points with Existing v2 Backend

### What the API Server Replaces

| Current (v2) | New | How |
|---------------|-----|-----|
| Lighttpd + PHP-CGI | Nginx + FastAPI | Nginx serves static + proxies to FastAPI |
| `htdocs/api/player.php` | `routers/player.py` | Same commands, typed Python |
| `htdocs/api/playlist.php` | `routers/playlist.py` | Same MPD protocol queries |
| `htdocs/api/volume.php` | `routers/volume.py` | Same playout_controls.sh calls |
| `htdocs/api/cover.php` | Nginx location block | Direct file serving, no code needed |
| `htdocs/api/latest.php` | `routers/player.py` | Read settings files directly |
| `htdocs/api/common.php execScript()` | `services/shell_service.py` | subprocess without shell=True |
| `htdocs/api/common.php execMPDCommand()` | `services/mpd_service.py` | python-mpd2 library |
| `jukebox.js` 5s polling | WebSocket + MPD idle | Real-time push |
| jQuery 1.12 + Bootstrap 3 | SvelteKit + Tailwind CSS | Modern SPA |

### What Stays Unchanged

| Component | Why Unchanged | Integration Method |
|-----------|---------------|-------------------|
| `daemon_rfid_reader.py` | Works, calls shell scripts directly | API watches for card events via file |
| `gpio_control.py` | Works, calls playout_controls.sh | No integration needed (goes through MPD) |
| `phonie-gyro` | Works, calls `mpc` directly | API sees MPD state changes via idle |
| `playout_controls.sh` | Central hub, 4 callers depend on it | API calls it via subprocess |
| `rfid_trigger_play.sh` | Card-to-folder resolution | Called by RFID daemon, not by API |
| `inc.writeGlobalConfig.sh` | Config merger | Called by config service after changes |
| `settings/` file structure | Contract between components | Read/written by config service |
| MPD on port 6600 | Audio engine | API connects via python-mpd2 |

### Coexistence Strategy (Phase 1 Transition)

During migration, both old and new UI can run simultaneously:

```
Nginx config:
  /          -> New SvelteKit UI (static files)
  /api/      -> New FastAPI server
  /ws        -> New WebSocket endpoint
  /legacy/   -> Old PHP UI (Lighttpd on port 8080, proxied)
```

This allows testing the new UI while keeping the old one accessible. Once the new UI covers all features, remove the legacy path and Lighttpd.

---

## Scalability Considerations

| Concern | Current (100 cards) | At 500 cards | At 2000 cards |
|---------|--------------------|--------------| --------------|
| Card lookup | Shell config file (fast) | Still fast (grep) | Consider SQLite index |
| Audio library browsing | Filesystem scan | Add caching (1min TTL) | MPD database query |
| WebSocket connections | 1-3 browsers | 1-3 browsers | 1-3 browsers (household device) |
| File upload | Direct to disk | Direct to disk | Add progress tracking via WS |
| MPD state queries | Per-change (idle) | Per-change | Per-change (no scaling issue) |
| Config file I/O | 50+ files on change | 50+ files | Consider consolidated YAML |

**Reality check:** This is a household device. It will never have more than 3 concurrent browser sessions. Scalability concerns are about data volume (cards, audio files), not concurrent users.

---

## Build Order / Dependency Graph

The components have clear build-order dependencies:

```
Phase 1a: API Foundation
  [1] FastAPI project structure + config
  [2] Shell service (wraps playout_controls.sh)
  [3] MPD service (wraps python-mpd2)
  [4] Player REST endpoints (play, pause, next, prev, volume)
  [5] Player GET endpoint (current status from MPD)
  --> At this point: curl can control the player

Phase 1b: Real-Time Layer
  [6] WebSocket hub (connection manager)
  [7] MPD bridge (idle -> broadcast)
  --> At this point: wscat shows real-time player state

Phase 1c: Frontend Foundation
  [8] SvelteKit project with adapter-static
  [9] WebSocket store (connects to /ws, manages reconnect)
  [10] Player component (NowPlaying, controls, volume)
  --> At this point: browser shows player and controls work

Phase 1d: Deployment
  [11] Nginx configuration
  [12] systemd service for FastAPI
  [13] Build + deploy script (scp from Windows)
  --> At this point: full player works on Pi

Phase 2: Library + Cards (depends on Phase 1)
  [14] Library REST endpoints (browse folders, list files)
  [15] Library UI (folder grid, file list)
  [16] Card REST endpoints (list, register, delete)
  [17] RFID event bridge (file watch -> WS)
  [18] Card registration wizard UI

Phase 3: Upload + Settings (depends on Phase 1)
  [19] File upload endpoint (multipart, validation)
  [20] Upload UI (drag-drop, progress)
  [21] Settings REST endpoints (read/write config)
  [22] Settings UI

Phase 4: Auth (can start after Phase 1c)
  [23] PIN storage + verification
  [24] JWT middleware
  [25] PIN dialog UI
  [26] Tier enforcement on routes

Phase 5: PWA + Offline (after Phase 1c is stable)
  [27] Service Worker registration
  [28] App shell caching
  [29] Offline state display
  [30] PWA manifest + icons
```

**Key dependency chain:** Items 1-5 must exist before 6-7, which must exist before 8-10, which must exist before 11-13. This is the critical path.

**Parallelization opportunity:** Auth (Phase 4) can be developed in parallel with Library + Cards (Phase 2) -- just apply auth middleware after both are done.

---

## Development Workflow

### Local Development (Windows PC)

```
Terminal 1: Frontend dev server
  cd phoniebox-ui
  npm run dev              # Vite dev server on :5173
                           # Proxy /api/ -> Pi or mock server

Terminal 2: Mock API (optional, for offline dev)
  cd phoniebox-api
  python main.py --mock    # Returns fake data, no Pi needed

Terminal 3: Real API on Pi (via SSH tunnel)
  ssh -L 8000:localhost:8000 pi@phoniebox.local
  # Now localhost:8000 reaches FastAPI on Pi
```

**Frontend-only development (no Pi):**
- Vite proxy points to a mock FastAPI server running locally
- Mock server returns realistic player state, fake card list, etc.
- WebSocket mock sends periodic state updates
- Cover images served from local test fixtures

**Full integration testing (with Pi):**
- SSH tunnel brings Pi's FastAPI to localhost:8000
- Vite proxy points to localhost:8000
- Real MPD state, real RFID events, real file system
- Browser on Windows talks to real Pi backend

### Deployment

```bash
# Build frontend on Windows
cd phoniebox-ui && npm run build

# Deploy to Pi (from Windows)
scp -r build/ pi@phoniebox.local:/home/pi/phoniebox-ui/

# Deploy API to Pi
scp -r phoniebox-api/ pi@phoniebox.local:/home/pi/phoniebox-api/

# On Pi: restart services
ssh pi@phoniebox.local "sudo systemctl restart phoniebox-api"
```

Future improvement: A simple Makefile or PowerShell script that runs build + deploy in one command.

---

## Technology Rationale Summary

| Choice | Why | Alternative Considered | Why Not |
|--------|-----|----------------------|---------|
| **FastAPI** | Async Python, WebSocket native, auto-OpenAPI docs, Pydantic validation | Flask, Django, Express.js | Flask lacks async/WS natively; Django too heavy; Express.js adds Node.js runtime on Pi |
| **python-mpd2** | Already used in v2 (gpio_control), async support, well-maintained | Raw TCP socket (like current PHP) | Library handles protocol edge cases, reconnection, async |
| **SvelteKit** | Small bundles, excellent reactivity, adapter-static for no-runtime deploy | React, Vue, plain Svelte | React larger bundles, slower on Pi; Vue similar tradeoffs; plain Svelte lacks routing/build |
| **Tailwind CSS** | Utility-first, small production CSS, mobile-first, no JS runtime | Bootstrap 5, Material UI | Bootstrap adds unused CSS weight; MUI requires React |
| **Nginx** | WebSocket proxy, static serving, ubiquitous, well-documented | Caddy, Lighttpd (current) | Lighttpd poor WS proxy; Caddy auto-HTTPS overkill for local network |
| **JWT** | Stateless auth, survives Pi reboot, no session store needed | Cookie sessions | Sessions need server-side store; JWT self-contained |
| **adapter-static** | No Node.js on Pi, pure static output | adapter-node | Would require Node.js runtime on Pi |

---

## Confidence Assessment

| Area | Confidence | Reasoning |
|------|------------|-----------|
| Component boundaries | HIGH | Based on direct code analysis of all v2 integration points |
| FastAPI + WebSocket architecture | HIGH | Well-established pattern for Python real-time APIs; python-mpd2 is already a v2 dependency |
| MPD idle bridge pattern | HIGH | MPD protocol explicitly supports idle/subscribe; this is the standard approach |
| SvelteKit adapter-static | MEDIUM | Based on training data; verify current SvelteKit adapter-static capabilities and PWA support with official docs |
| RFID event forwarding | MEDIUM | File-watch approach is sound but untested; may need latency tuning |
| Auth tier approach | HIGH | JWT + PIN is standard for embedded devices; bcrypt well-supported in Python |
| Nginx WebSocket proxy | HIGH | Standard, well-documented configuration |
| Build order dependencies | HIGH | Derived directly from component communication analysis |

---

## Sources

- Codebase analysis: `C:\Users\herrkraft\Repos\rfid-jukebox\.planning\codebase\ARCHITECTURE.md`
- Codebase analysis: `C:\Users\herrkraft\Repos\rfid-jukebox\.planning\codebase\INTEGRATIONS.md`
- Codebase analysis: `C:\Users\herrkraft\Repos\rfid-jukebox\.planning\codebase\CONCERNS.md`
- Source code: `htdocs/api/common.php` (current PHP bridge to shell/MPD)
- Source code: `htdocs/api/player.php` (current player API, command mapping)
- Source code: `htdocs/api/playlist.php` (current playlist API)
- Source code: `htdocs/api/volume.php` (current volume API)
- Source code: `htdocs/api/cover.php` (current cover serving)
- Source code: `htdocs/api/latest.php` (current last-played API)
- Source code: `htdocs/js/jukebox.js` (current frontend polling logic)
- Source code: `scripts/daemon_rfid_reader.py` (RFID daemon integration points)
- Source code: `scripts/playout_controls.sh` (command interface, lines 1-100)
- Source code: `scripts/rfid_trigger_play.sh` (card trigger flow)
- Project analysis: `C:\Users\herrkraft\Repos\rfid-jukebox\.planning\ANALYSIS.md`
- python-mpd2: Already a dependency in `requirements-GPIO.txt` (HIGH confidence)
- FastAPI WebSocket: Based on training data for FastAPI 0.100+ (MEDIUM confidence -- verify current version)
- SvelteKit adapter-static: Based on training data (MEDIUM confidence -- verify with official docs)
- MPD idle protocol: Standard MPD protocol feature since MPD 0.14 (HIGH confidence)

---

*Architecture research: 2026-02-06*

# Technology Stack

**Project:** Phoniebox Web-Interface Modernization (Phase 1)
**Researched:** 2026-02-06
**Overall Confidence:** MEDIUM (WebSearch/WebFetch unavailable -- recommendations based on training data knowledge through May 2025, cross-referenced with existing codebase analysis. Versions should be verified against official docs before implementation.)

---

## Recommended Stack

### Frontend Framework

| Technology | Version | Purpose | Why | Confidence |
|------------|---------|---------|-----|------------|
| **Svelte 5** | ^5.x | UI framework | Compiles to vanilla JS at build time -- smallest runtime (~2KB), fastest on constrained hardware. No virtual DOM overhead. Runes reactivity model is simpler than React hooks. Perfect for resource-limited Pi. | MEDIUM |
| **SvelteKit** | ^2.x | App framework | File-based routing, SSR/SSG support, built-in adapter system. Can pre-render static pages and serve from any web server. | MEDIUM |
| **TypeScript** | ^5.x | Type safety | Catches errors at build time. Svelte 5 has first-class TS support. Essential for maintaining a multi-component app. | HIGH |

**Why Svelte over alternatives:**

| Framework | Bundle Size (min+gz) | Runtime Overhead | Pi Suitability | DX |
|-----------|---------------------|------------------|----------------|-----|
| **Svelte 5** | ~2KB runtime + components | Compiles away -- no framework in browser | Excellent | Simple, less boilerplate |
| Preact | ~4KB | Lightweight vDOM | Good | React-compatible but smaller ecosystem |
| React 19 | ~45KB | Full vDOM reconciliation | Poor -- heavy for Pi | Largest ecosystem |
| Vue 3 | ~33KB | vDOM + reactivity system | Mediocre | Good, but heavier than Svelte |
| SolidJS | ~7KB | Fine-grained reactivity, no vDOM | Good | JSX-based, smaller community |
| Vanilla JS | 0KB | None | Best raw performance | Unmaintainable at scale |

**Decision rationale:** On a Pi 3B with limited RAM and CPU, every KB matters. Svelte compiles components to imperative DOM manipulation at build time -- the browser receives optimized vanilla JS with minimal framework overhead. React's 45KB+ runtime and vDOM diffing is wasteful on constrained hardware. Preact is a reasonable alternative at ~4KB, but Svelte's compiler approach means even less work for the browser. SolidJS is strong technically but has a much smaller ecosystem and community.

### UI Component Library / CSS

| Technology | Version | Purpose | Why | Confidence |
|------------|---------|---------|-----|------------|
| **Tailwind CSS** | ^4.x | Utility-first CSS | Purged CSS = tiny production bundle. No unused CSS shipped. Mobile-first by design. Works with any framework. | MEDIUM |
| **DaisyUI** | ^5.x | Tailwind component library | Pre-built semantic components (buttons, cards, modals) on top of Tailwind. Supports theming. No JS runtime -- pure CSS. Reduces custom CSS needed for a polished UI. | LOW |

**Why NOT a full component library like Shadcn/MUI/Ant Design:**
- MUI 5/6: Requires React. Heavy. Lots of JS runtime for styling.
- Ant Design: React-only. Enterprise-focused. Massively oversized for this use case.
- Skeleton UI (Svelte): Decent option but less mature than Tailwind+DaisyUI.
- Shadcn-svelte: Possible alternative, uses Tailwind under the hood. Worth evaluating.

**Alternative considered:** Plain CSS with CSS custom properties. Zero dependency. But significantly more work to build a responsive, polished mobile UI from scratch. Tailwind trades a build dependency for massive development speed.

### Backend / API Layer

| Technology | Version | Purpose | Why | Confidence |
|------------|---------|---------|-----|------------|
| **FastAPI** | ^0.115+ | REST + WebSocket API | Async Python. Native WebSocket support. Automatic OpenAPI docs. Type-safe with Pydantic. Python already on the Pi (used by RFID daemon, GPIO control, phonie-gyro). One language for all backend. | MEDIUM |
| **Uvicorn** | ^0.34+ | ASGI server | Lightweight async server for FastAPI. Low memory footprint. Production-ready. | MEDIUM |
| **python-mpd2** | ^3.x | MPD client | Already used by GPIO control (`requirements-GPIO.txt`). Supports asyncio. Direct socket communication with MPD on port 6600. | HIGH |
| **Pydantic** | ^2.x | Data validation | Included with FastAPI. Validates all API inputs. Replaces ad-hoc validation. | MEDIUM |

**Why FastAPI over alternatives:**

| Backend | Language | WebSocket | Async | Pi Memory | Ecosystem Fit |
|---------|----------|-----------|-------|-----------|---------------|
| **FastAPI** | Python | Native | Yes (ASGI) | ~30-50MB | Excellent -- Python already on Pi for RFID/GPIO/Gyro |
| Flask | Python | Via flask-socketio | Partial | ~20-40MB | Good but no native async/WS |
| Express.js | Node.js | Via ws/socket.io | Yes | ~50-80MB | Adds Node.js dependency to Pi |
| Go (stdlib) | Go | Native | Yes (goroutines) | ~10-20MB | Fastest, but adds build complexity for ARM cross-compilation |
| **Keep PHP** | PHP | No native support | No | ~15MB | Already there but fundamentally wrong for real-time |

**Decision rationale:** Python is already the dominant backend language on this Pi (RFID daemon, GPIO control, phonie-gyro). FastAPI adds WebSocket support (critical for real-time player state), automatic API documentation, and type safety. Flask lacks native WebSocket support. Node.js adds a new runtime to the Pi. Go would be excellent performance-wise but introduces cross-compilation complexity for ARM and a new language to the project. PHP fundamentally cannot do WebSocket without bolted-on hacks.

**Why NOT keep PHP:**
1. No WebSocket support -- real-time player state requires polling (wasteful on Pi)
2. Every request spawns `exec("sudo ...")` -- process creation overhead
3. No async I/O -- blocks on MPD socket, shell exec, file reads
4. Security nightmare -- command injection via unsanitized shell exec
5. 77 files of mixed HTML/PHP cannot be incrementally improved
6. PHP-FPM or Lighttpd+FastCGI adds memory overhead for what Python does natively

### Web Server

| Technology | Version | Purpose | Why | Confidence |
|------------|---------|---------|-----|------------|
| **Nginx** | ^1.26+ | Reverse proxy + static files | Battle-tested. Tiny memory footprint (~2-5MB). Serves static SvelteKit build. Proxies `/api/` to Uvicorn. Native WebSocket proxy support. Available in Raspberry Pi OS repos. | HIGH |

**Why switch from Lighttpd to Nginx:**

| Server | Memory | WebSocket Proxy | Static Files | Config Complexity | Pi Ecosystem |
|--------|--------|----------------|--------------|-------------------|--------------|
| **Nginx** | ~2-5MB | Yes (native) | Excellent | Moderate | Standard in Pi projects |
| Lighttpd | ~1-3MB | Limited/hacky | Good | Simple | Current (works) |
| Caddy | ~15-30MB | Yes | Excellent | Simplest | Higher memory, Go binary |
| Uvicorn direct | N/A | Yes | Possible but slow | None | Not recommended for static |

**Decision rationale:** Lighttpd *could* be kept for static files, but it has poor WebSocket proxy support, which is a hard requirement for real-time player state. Nginx is the standard reverse proxy in the Raspberry Pi ecosystem, has native WebSocket proxy (`proxy_pass` with `Upgrade` headers), excellent static file serving with gzip, and uses only ~2-5MB more than Lighttpd. Caddy is simpler to configure but its Go binary uses 15-30MB of RAM -- unnecessary on a Pi 3B with 1GB. Running Uvicorn directly without a reverse proxy works for development but is not recommended for production (no static file caching, no connection limiting, no SSL termination).

### Real-Time Communication

| Technology | Version | Purpose | Why | Confidence |
|------------|---------|---------|-----|------------|
| **WebSocket** | Native | Bidirectional real-time | Player state updates (position, track, volume) need sub-second latency. WebSocket provides persistent bidirectional connection. FastAPI has native WS support. Nginx proxies WS natively. | HIGH |

**Why WebSocket over alternatives:**

| Method | Latency | Direction | Connection | Pi Impact | Use Case Fit |
|--------|---------|-----------|------------|-----------|-------------|
| **WebSocket** | <100ms | Bidirectional | Persistent | 1 connection per client | Perfect -- player controls + state |
| SSE (Server-Sent Events) | <100ms | Server-to-client only | Persistent | 1 connection per client | Good for state push but no upstream |
| Polling | 1-5 seconds | Request/response | New each time | High -- repeated connections | Wasteful, laggy UX |
| Long Polling | ~100ms | Pseudo-push | Repeated | Medium | Complex, fragile |

**Decision rationale:** The music player UI needs both directions: push player state (server->client: what's playing, position, volume) AND send commands (client->server: play, pause, next). SSE only handles server-to-client, so commands would still need REST endpoints -- two protocols for one connection. WebSocket handles both in a single persistent connection. On a Pi, fewer connections = less overhead.

**Implementation pattern:**
```
Browser <--WebSocket--> Nginx <--proxy_pass--> Uvicorn/FastAPI
                                                    |
                                            python-mpd2 (asyncio)
                                                    |
                                              MPD :6600
```

FastAPI backend subscribes to MPD idle events via `python-mpd2` asyncio client, pushes state changes to all connected WebSocket clients. Commands from clients are forwarded to MPD or dispatched to shell scripts.

### Offline / PWA Strategy

| Technology | Version | Purpose | Why | Confidence |
|------------|---------|---------|-----|------------|
| **Service Worker** | Web API | Offline app shell | Cache the SvelteKit static build so the UI loads instantly even without the Pi being fully booted. The music files are local anyway -- offline-capable by design. | HIGH |
| **PWA Manifest** | Web API | Installable app | Parents install to home screen on phone. Feels native. Full-screen mode. No app store needed. | HIGH |

**Offline strategy:**
- **App shell caching:** Service Worker caches all static assets (HTML, JS, CSS, images). UI always loads.
- **Network-first for API:** Player state and controls require the Pi backend -- no point caching API responses.
- **This is NOT a traditional offline-first app.** The Pi IS the server and the music source. "Offline" means "Pi is on but phone has no internet" -- which is the normal case. The PWA just needs to cache the UI shell so it loads from the Pi's local network instantly.

### Build Tooling

| Technology | Version | Purpose | Why | Confidence |
|------------|---------|---------|-----|------------|
| **Vite** | ^6.x | Build tool + dev server | SvelteKit uses Vite under the hood. Fast HMR for development. Optimized production builds with tree-shaking and code-splitting. | MEDIUM |
| **pnpm** | ^9.x | Package manager | Faster than npm, uses less disk space (important on SD cards). Strict by default (no phantom dependencies). | MEDIUM |

**Build strategy:** Build on development machine (PC/Mac), deploy static output to Pi. The Pi does NOT run Node.js or build tools. This is critical: `npm install` + `vite build` on a Pi 3B would take 10+ minutes and risk running out of RAM. Build artifacts are static files served by Nginx.

### Database

| Technology | Version | Purpose | Why | Confidence |
|------------|---------|---------|-----|------------|
| **SQLite** | ^3.x | Card mappings, settings | Replace 50+ config files in `settings/`. Single file database. Zero-config. Built into Python stdlib. ACID-compliant. Survives power cuts (WAL mode). | HIGH |
| **No ORM** | -- | Direct SQL | For <10 tables, an ORM adds complexity without value. Use `aiosqlite` for async access from FastAPI. | HIGH |

**Why SQLite over alternatives:**

| Option | Overhead | Reliability | Query Capability | Pi Fit |
|--------|----------|-------------|------------------|--------|
| **SQLite** | 0 (built-in) | Excellent (WAL mode) | Full SQL | Perfect |
| 50+ flat files | 0 | Poor (no atomicity) | grep | Current (fragile) |
| PostgreSQL | ~50MB RAM | Excellent | Full SQL | Overkill |
| Redis | ~10MB RAM | Good | Key-value only | Overkill |
| JSON files | 0 | Moderate | No query | Simple but limited |

**Decision rationale:** The current system uses 50+ individual text files in `settings/` that break on power cuts, have no atomicity, and are parsed by shell sourcing. SQLite provides ACID transactions in a single file, with zero additional memory overhead (it is part of Python's stdlib). WAL mode ensures the database survives unexpected power loss -- critical for a child's toy that gets turned off by pulling the plug. No external service to manage.

### Authentication

| Technology | Version | Purpose | Why | Confidence |
|------------|---------|---------|-----|------------|
| **PIN-based auth** | Custom | Three-tier access | Children see player only. Parents enter 4-digit PIN for management. Experts get full settings access. No passwords to remember -- parents just need a simple PIN. | HIGH |
| **JWT (PyJWT)** | ^2.x | Session tokens | Stateless auth tokens stored in browser. No server-side session storage needed. Include role (player/parent/expert) in token payload. | MEDIUM |

**Access tier model:**
```
Open (no auth)     -> Player view: what's playing, volume, play/pause
PIN (4-digit)      -> Parent area: card management, upload music, basic settings
PIN (separate)     -> Expert area: system config, gyro calibration, network, logs
```

**Why NOT OAuth/OIDC or complex auth:** This runs on a home network for a family. A 4-digit PIN that parents can remember is the right level of security. Full OAuth would be absurd overkill.

### Internationalization

| Technology | Version | Purpose | Why | Confidence |
|------------|---------|---------|-----|------------|
| **svelte-i18n** or **paraglide-js** | Latest | Multilingual UI | German primary, English secondary. "Leichte Sprache" (simple German) for parent UI. Compile-time i18n preferred (paraglide-js) for smallest bundle. | LOW |

**Note:** Need to verify current state of Svelte 5 i18n libraries. `svelte-i18n` is established but runtime-based. `paraglide-js` (from Inlang) compiles away at build time -- better for bundle size but need to verify Svelte 5 compatibility.

---

## Alternatives Considered

| Category | Recommended | Alternative | Why Not |
|----------|-------------|-------------|---------|
| Frontend | Svelte 5 | Preact | Viable. ~4KB. React-compatible. But Svelte's compiler approach yields even less browser work. Preact is the backup choice. |
| Frontend | Svelte 5 | React 19 | 45KB+ runtime. vDOM diffing is wasteful CPU work on Pi. Heavy. |
| Frontend | Svelte 5 | Vue 3 | 33KB+ runtime. Good framework but heavier than needed. |
| Frontend | Svelte 5 | HTMX + server templates | Interesting for simpler apps. But real-time player UI with WebSocket needs more client-side logic than HTMX provides cleanly. |
| Backend | FastAPI | Flask + flask-socketio | Flask lacks native async. flask-socketio works but adds eventlet/gevent complexity. FastAPI is simpler for async WebSocket. |
| Backend | FastAPI | Express.js | Adds Node.js to the Pi. Python is already the backend language. |
| Backend | FastAPI | Go net/http | Best performance. But ARM cross-compilation and new language add friction. Overkill for <20 API endpoints. |
| Backend | FastAPI | Keep PHP | No WebSocket. No async. Security holes. Dead end. |
| Web server | Nginx | Keep Lighttpd | No WebSocket proxy. Could theoretically hack it with mod_wstunnel but poorly supported. |
| Web server | Nginx | Caddy | 15-30MB Go binary. Unnecessary on Pi 3B. Nice auto-HTTPS but not needed on local network. |
| Database | SQLite | Keep flat files | No atomicity. Power cut = corruption. No query capability. |
| Database | SQLite | TinyDB (Python) | JSON-based. No SQL. Slower for queries. Less battle-tested. |
| Real-time | WebSocket | SSE + REST | Two protocols instead of one. SSE is server-to-client only. |
| CSS | Tailwind | Pico CSS | Classless CSS is beautiful and simple. But insufficient for a complex responsive UI with custom theming. |

---

## Complete Stack Summary

```
Production (Raspberry Pi):
+----------------------------------------------------------+
|  Browser (Phone/Tablet)                                   |
|  Svelte 5 SPA + Tailwind CSS + Service Worker (PWA)      |
+---------------------------+------------------------------+
                            | HTTP / WebSocket
+---------------------------+------------------------------+
|  Nginx (reverse proxy + static files)         :80/:443   |
|  - /           -> static SvelteKit build                 |
|  - /api/       -> proxy_pass uvicorn :8000               |
|  - /ws/        -> WebSocket proxy to uvicorn :8000       |
+---------------------------+------------------------------+
                            | ASGI
+---------------------------+------------------------------+
|  FastAPI + Uvicorn                            :8000      |
|  - REST endpoints (/api/player, /api/cards, ...)         |
|  - WebSocket endpoint (/ws/player)                       |
|  - python-mpd2 asyncio client -> MPD :6600               |
|  - subprocess calls -> playout_controls.sh (legacy)      |
|  - SQLite for config/cards                               |
+---------------------------+------------------------------+
            |                           |
    +-------+--------+     +-----------+-----------+
    | MPD :6600      |     | Existing v2 Backend   |
    | (unchanged)    |     | (shell scripts,       |
    |                |     |  RFID daemon,         |
    |                |     |  GPIO, phonie-gyro)   |
    +----------------+     +-----------------------+
```

```
Development (PC/Mac):
+------------------------------------------+
|  Vite dev server (HMR)         :5173     |
|  SvelteKit in dev mode                   |
+------------------+-----------------------+
                   | proxy to Pi
+------------------+-----------------------+
|  Pi running FastAPI            :8000     |
|  (SSH tunnel or direct network)          |
+------------------------------------------+
```

---

## Migration Strategy: PHP to FastAPI

**Do NOT replace all PHP at once.** Incremental migration:

### Phase 1a: New API alongside PHP
1. Install FastAPI + Uvicorn on Pi
2. Configure Nginx to proxy `/api/v2/` to FastAPI `:8000`
3. Keep Lighttpd running old PHP UI on `:8080` (fallback)
4. New SvelteKit frontend talks to FastAPI only

### Phase 1b: Migrate existing API endpoints
Map current PHP endpoints to FastAPI:

| Current PHP | New FastAPI | Method | Notes |
|-------------|------------|--------|-------|
| `api/player.php` GET | `/api/player/status` | GET | Direct MPD socket via python-mpd2 |
| `api/player.php` PUT | `/api/player/command` | POST | JSON body with command + value |
| `api/volume.php` GET | `/api/player/volume` | GET | Via python-mpd2 or amixer |
| `api/volume.php` PUT | `/api/player/volume` | PUT | Numeric body |
| `api/playlist.php` GET | `/api/player/playlist` | GET | Via python-mpd2 |
| `api/playlist.php` PUT | `/api/player/playlist` | POST | Trigger playlist play |
| `api/cover.php` | `/api/player/cover` | GET | Serve cover image |
| `api/latest.php` | `/api/player/latest` | GET | Read from SQLite instead of flat files |
| -- (new) | `/ws/player` | WebSocket | Real-time state push |

### Phase 1c: Remove PHP
Once all endpoints migrated and tested, remove Lighttpd and PHP.

---

## Installation

### Pi Production Dependencies

```bash
# System packages (add to packages.txt)
sudo apt-get install nginx python3-pip python3-venv

# Remove PHP (after migration complete)
# sudo apt-get remove lighttpd php-common php-cgi php

# Python virtual environment (recommended)
python3 -m venv /home/pi/phoniebox-api/.venv
source /home/pi/phoniebox-api/.venv/bin/activate

# Python packages
pip install fastapi uvicorn[standard] python-mpd2 aiosqlite pyjwt
```

### Development Machine Dependencies

```bash
# Node.js (LTS) for frontend build
# Install via nvm or system package manager

# Frontend
pnpm create svelte@latest frontend
cd frontend
pnpm install
pnpm add -D tailwindcss @tailwindcss/vite
pnpm add -D daisyui      # optional, evaluate first

# TypeScript (included with SvelteKit)
```

### Systemd Service (FastAPI)

```ini
# /etc/systemd/system/phoniebox-api.service
[Unit]
Description=Phoniebox API
After=network.target mpd.service

[Service]
Type=simple
User=pi
WorkingDirectory=/home/pi/phoniebox-api
ExecStart=/home/pi/phoniebox-api/.venv/bin/uvicorn main:app --host 0.0.0.0 --port 8000
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

### Nginx Configuration

```nginx
# /etc/nginx/sites-available/phoniebox
server {
    listen 80;
    server_name _;

    # Static SvelteKit build
    root /home/pi/phoniebox-ui/build;
    index index.html;

    # SPA fallback
    location / {
        try_files $uri $uri/ /index.html;
    }

    # API proxy
    location /api/ {
        proxy_pass http://127.0.0.1:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }

    # WebSocket proxy
    location /ws/ {
        proxy_pass http://127.0.0.1:8000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_read_timeout 86400;
    }

    # Cover images (serve directly, bypass API for performance)
    location /media/ {
        alias /home/pi/RPi-Jukebox-RFID/shared/audiofolders/;
        try_files $uri =404;
    }
}
```

---

## Pi-Specific Performance Considerations

### Memory Budget (Pi 3B = 1GB RAM)

| Component | Estimated RAM | Notes |
|-----------|---------------|-------|
| Raspberry Pi OS | ~150MB | Base system |
| MPD | ~20-40MB | Depends on library size |
| RFID daemon | ~15MB | Python process |
| GPIO daemon | ~15MB | Python process |
| phonie-gyro | ~15MB | Python process |
| **Uvicorn/FastAPI** | ~30-50MB | Single worker, async |
| **Nginx** | ~2-5MB | Static + proxy |
| **SQLite** | ~0MB | In-process, no separate daemon |
| **Total new** | ~35-55MB | |
| **Available after** | ~700-750MB | Plenty of headroom |

**Current PHP (Lighttpd + PHP-FPM):** ~15-30MB. New stack uses ~20-25MB more. Acceptable.

### CPU Considerations

- **Svelte compiles at build time on dev machine** -- Pi serves static files only. No SSR needed.
- **FastAPI async** -- single process handles multiple connections without threads. Better than PHP's fork-per-request model.
- **python-mpd2 asyncio** -- non-blocking MPD communication. Single event loop, no thread overhead.
- **Nginx** -- event-driven, handles thousands of connections with minimal CPU.

### SD Card Considerations

- **pnpm** uses hard links -- saves SD card space vs npm's flat node_modules.
- **But:** Node.js and pnpm are NOT installed on Pi. Build happens on dev machine.
- **SQLite WAL mode** -- reduces write amplification on SD card vs individual file writes.
- **Log rotation** -- critical to prevent SD card fill. Configure logrotate for Uvicorn and Nginx logs.

### Network Considerations

- **All communication is local network** -- no internet required.
- **WebSocket** -- single persistent connection instead of repeated HTTP requests.
- **Gzip via Nginx** -- compress static assets. Svelte builds are already small.
- **Service Worker caching** -- after first load, UI assets served from browser cache.

---

## Version Verification Status

| Technology | Recommended Version | Verified Via | Confidence | Action Needed |
|------------|-------------------|-------------|------------|---------------|
| Svelte | 5.x | Training data (May 2025) | MEDIUM | Verify current stable version |
| SvelteKit | 2.x | Training data | MEDIUM | Verify current stable version |
| FastAPI | 0.115+ | Training data | MEDIUM | Verify latest on PyPI |
| Uvicorn | 0.34+ | Training data | MEDIUM | Verify latest on PyPI |
| python-mpd2 | 3.x | Existing in codebase (requirements-GPIO.txt) | HIGH | Already used |
| Nginx | 1.26+ | Standard in Debian repos | HIGH | `apt-cache show nginx` on Pi |
| Tailwind CSS | 4.x | Training data (v4 released early 2025) | MEDIUM | Verify v4 is stable |
| Vite | 6.x | Training data | MEDIUM | Verify current stable |
| SQLite | 3.x | Built into Python | HIGH | Always available |
| PyJWT | 2.x | Training data | MEDIUM | Verify on PyPI |
| pnpm | 9.x | Training data | MEDIUM | Verify latest stable |
| DaisyUI | 5.x | Training data (v5 may have released) | LOW | Verify compatibility with Tailwind 4 |

**WARNING:** WebSearch and WebFetch were unavailable during this research. All version numbers are based on training data through May 2025. Before implementation, verify:
1. `svelte --version` after creating project
2. `pip index versions fastapi` on Pi
3. Check https://svelte.dev, https://fastapi.tiangolo.com for current releases
4. Check Tailwind v4 + DaisyUI v5 compatibility

---

## Sources

- **Existing codebase analysis:** `.planning/codebase/STACK.md`, `.planning/codebase/ARCHITECTURE.md`, `.planning/codebase/CONCERNS.md`
- **Existing project analysis:** `.planning/ANALYSIS.md` (strategy decision: v2 base + incremental improvements)
- **Current API surface:** `htdocs/api/*.php` (6 files analyzed)
- **Current dependencies:** `requirements.txt`, `requirements-GPIO.txt`, `packages.txt`
- **Current web server config:** `misc/sampleconfigs/lighttpd.conf-default.sample`
- **Training data knowledge** (May 2025): Svelte 5, FastAPI, Tailwind CSS 4, SvelteKit 2, Vite 6 -- all MEDIUM confidence, need verification

---

## Summary Decision Matrix

| Decision | Choice | Certainty | Fallback |
|----------|--------|-----------|----------|
| Frontend framework | Svelte 5 / SvelteKit 2 | MEDIUM | Preact + Vite |
| CSS framework | Tailwind CSS 4 | MEDIUM | Pico CSS or plain CSS |
| Backend API | FastAPI + Uvicorn | MEDIUM | Flask + flask-socketio |
| Web server | Nginx | HIGH | Keep Lighttpd (but lose WebSocket) |
| Real-time | WebSocket | HIGH | SSE + REST fallback |
| Database | SQLite | HIGH | JSON files (worse) |
| Auth | PIN + JWT | HIGH | Basic HTTP auth |
| Build tool | Vite (via SvelteKit) | MEDIUM | Rollup |
| Package manager | pnpm | MEDIUM | npm |
| Language (frontend) | TypeScript | HIGH | JavaScript |
| Language (backend) | Python 3.11+ | HIGH | N/A |
| Offline | Service Worker / PWA | HIGH | N/A |

# Feature Landscape

**Domain:** Embedded media player web interface for children's music box (Phoniebox)
**Researched:** 2026-02-06
**Confidence:** MEDIUM (based on training knowledge of Toniebox/mytonies, Volumio, moOde, Sonos; no live web verification possible)

## Competitor Reference

Before categorizing features, here is what the competitive landscape offers. This informs what parents will expect.

| Feature Area | Toniebox (mytonies) | Volumio | moOde Audio | Sonos App | Phoniebox v2 (current) |
|---|---|---|---|---|---|
| Player controls | Play/pause via physical, basic app controls | Full transport, seek bar, queue | Full transport, seek, EQ | Full transport, seek, crossfade | Play/pause/prev/next/seek/repeat/mute |
| Now playing display | Cover art, title, progress | Cover art, title, artist, progress bar | Cover art, metadata, bitrate, format | Cover art, full metadata, lyrics | Playlist name, track list, no progress bar |
| Library browsing | Simple content tiles by Tonie figure | Grid + list, search, genre/artist/album | List-based, album art, search, filters | Grid, search, browse by artist/album/genre | Folder tree with type filter tabs |
| Content management | Buy from Tonies store, record via Creative-Tonies | Local files, NAS mount | Local files, NAS, USB | Streaming only (no local management) | File upload, folder creation |
| RFID/figure assignment | Automatic (NFC in each Tonie) | N/A | N/A | N/A | Manual: swipe card + select folder/stream |
| Parental controls | Max volume, sleep timer, content curated | Volume limit | Volume limit, output config | Volume limit, room grouping | Max volume, sleep timer, idle shutdown |
| Streaming | Tonies content cloud only | Spotify, Tidal, Qobuz, web radio, Spotify Connect | Spotify Connect, web radio, AirPlay | All major services | Spotify (broken), web radio, podcast URL |
| Settings UI | Minimal (WiFi, volume limit, sleep timer) | Extensive (audio, network, plugins, sources) | Very technical (ALSA, MPD, kernel tweaks) | Moderate (rooms, services, EQ) | Extensive but technical (20+ settings panels) |
| Mobile optimization | Native iOS/Android app | Responsive web, native app | Responsive web | Native app | Responsive Bootstrap 3, dated look |
| Offline capability | Full offline playback | Yes (local files) | Yes (local files) | No (streaming only) | Yes (local files) |
| Auth/roles | Account-based (Tonies cloud) | None by default | None | Account-based | None |

**Key insight:** The Toniebox/mytonies app is the closest competitor in terms of use case (parents managing children's music box). Volumio and moOde are audiophile-oriented -- too technical. Sonos is the gold standard for UX simplicity. The sweet spot is: **Sonos-level simplicity applied to Phoniebox's use case.**

---

## Table Stakes

Features users expect. Missing = product feels incomplete or frustrating. Parents coming from Toniebox or any music app will expect these.

### Player Controls & Now Playing

| Feature | Why Expected | Complexity | Notes |
|---------|-------------|------------|-------|
| Play / Pause / Stop | Universal music control | Low | Already exists in v2, needs visual refresh |
| Previous / Next track | Universal music control | Low | Already exists in v2 |
| Volume slider (not just +/-) | Every music app has this | Low | v2 has +/- buttons only, needs proper slider |
| Now Playing: cover art | Visual identification of content | Medium | v2 has optional cover display; needs reliable, prominent display |
| Now Playing: title + artist | Basic "what's playing" info | Low | v2 shows playlist, not clean track info |
| Progress bar with seek | Parents need to skip ahead in long audiobooks | Medium | v2 has seek buttons but no visual progress bar |
| Track position in playlist | "Track 3 of 12" context | Low | v2 shows track list but not prominent position indicator |
| Repeat modes (off / playlist / single) | Standard for audiobooks: repeat off to stop after story | Low | Already in v2 |

### Library Browsing

| Feature | Why Expected | Complexity | Notes |
|---------|-------------|------------|-------|
| Browse by folder/album with cover art grid | Visual browsing is standard in every music app | Medium | v2 has text-only folder tree; grid with covers is expected |
| Content type filter (music/audiobook/radio/podcast) | v2 already has this concept (filter tabs) | Low | v2 has filter buttons, keep but improve UX |
| Search (text search across library) | Universal expectation from any content app | Medium | v2 has separate search page; should be inline/always available |
| Sort options (alphabetical, recently added, recently played) | Basic organization | Low | v2 sorts alphabetically only |
| Play directly from library | Tap an album/folder to start playing | Low | v2 has this via folder play buttons |

### Content Management

| Feature | Why Expected | Complexity | Notes |
|---------|-------------|------------|-------|
| Upload audio files via browser | Core feature -- parents need to add content | Medium | v2 has this but limited (PHP upload size issues) |
| Create/rename/delete folders | Basic file organization | Low | v2 has folder creation; rename and delete missing |
| Multi-file upload with progress indicator | Uploading 20 audiobook chapters needs progress | Medium | v2 has basic multi-upload, no progress indication |
| Folder/album cover art assignment | Visual identity for each album/audiobook | Medium | v2 has cover support but manual |

### RFID Card Management

| Feature | Why Expected | Complexity | Notes |
|---------|-------------|------------|-------|
| Assign card to folder (audio content) | Core Phoniebox feature | Low | v2 has this; needs UX improvement |
| Assign card to stream URL (radio/podcast) | Core Phoniebox feature | Low | v2 has this |
| Assign card to system command (volume, shutdown, etc.) | Power user feature but expected in Phoniebox ecosystem | Low | v2 has this |
| View all card assignments (card overview) | Parents need to see what each card does | Low | v2 has this implicitly via folder view shortcuts |
| Edit existing card assignment | Change what a card plays | Low | v2 has cardEdit.php |
| Delete card assignment | Remove unused cards | Low | v2 has this |
| "Last scanned card" display during registration | Real-time feedback: "Put card on reader, then assign" | Low | v2 has this via AJAX polling |

### Parental Controls

| Feature | Why Expected | Complexity | Notes |
|---------|-------------|------------|-------|
| Maximum volume limit | Child ear protection -- Toniebox has this | Low | v2 has this setting |
| Sleep timer (stop after X minutes) | Bedtime listening -- Toniebox has this | Low | v2 has sleep timer and stop timer |
| Idle shutdown (shutdown after X min idle) | Battery/energy saving | Low | v2 has this |
| Startup volume (fixed volume on boot) | Prevent blasting after restart | Low | v2 has this |

### Settings & Configuration

| Feature | Why Expected | Complexity | Notes |
|---------|-------------|------------|-------|
| WiFi configuration | Network setup without SSH | Medium | v2 has this |
| Volume step size configuration | Customize how much each button press changes volume | Low | v2 has this |
| Language selection | Multi-language support | Low | v2 supports DE, EN, FR, NL |
| System info (IP address, disk space, uptime) | Basic diagnostics | Low | v2 has systemInfo.php |
| Shutdown / Reboot buttons | Remote power control | Low | v2 has these in navigation |
| RFID reader behavior (second swipe action) | Phoniebox-specific: what happens when card is swiped again | Low | v2 has this |

### Mobile Optimization

| Feature | Why Expected | Complexity | Notes |
|---------|-------------|------------|-------|
| Responsive design (works on phone) | Parents primarily access from smartphone | Medium | v2 has Bootstrap 3 responsive, but dated |
| Touch-friendly controls (large tap targets) | Phone usage requires 44x44px minimum touch targets | Low | Design concern, not feature; built into modern frameworks |
| Fast initial load (< 3 seconds) | Slow load = frustration, especially on WiFi to RPi | Medium | v2 loads each page server-side; SPA would be faster after initial load |

---

## Differentiators

Features that set this product apart from v2 and from competitors. Not expected, but highly valued -- the "wow" moments that make parents love the interface.

### UX & Accessibility

| Feature | Value Proposition | Complexity | Notes |
|---------|-------------------|------------|-------|
| Three-tier access model (open player / PIN-parent / expert) | Unique: kids see only player, parents manage, experts configure | Medium | Not in v2 or any competitor. Toniebox has app-only (no open web). Core differentiator for Phoniebox. |
| "Guided" card registration wizard | Step-by-step: 1. Scan card 2. Choose content 3. Done | Medium | v2 shows all options at once (folder, stream, command, YouTube) which overwhelms. A wizard simplifies. |
| Visual card-to-content mapping ("card wall") | Grid of cards with cover art showing what each card plays | Medium | No competitor has this. Parents can see all cards at a glance. |
| Resume position memory (per card) | Pick up audiobook where child stopped | Medium | v2 has basic MPD resume. Making this visible and reliable is a differentiator (Toniebox does this well). |
| PWA / Add to Home Screen | Access like a native app, no app store needed | Low | Simply adding a manifest.json and service worker. Big UX win for parents. |
| Drag-and-drop playlist editing | Reorder tracks within a folder/playlist by dragging | Medium | v2 has move up/down buttons. Drag-and-drop is modern and intuitive. |
| Real-time player state via WebSocket | Instant UI updates when child presses physical button or scans card | Medium | v2 polls every second. WebSocket gives instant feedback. Parents see exactly what the box is doing. |

### Content Acquisition

| Feature | Value Proposition | Complexity | Notes |
|---------|-------------------|------------|-------|
| Pre-configured stream catalog (ARD/NDR Audiothek, Deutschlandfunk, etc.) | Browse and add German children's radio stations with one tap | Medium | v2 requires manual URL entry. A curated catalog of kid-friendly streams is hugely valuable for German market. |
| Podcast RSS subscription | Enter podcast URL, auto-download new episodes | High | Not in v2. Would make Phoniebox a podcast player for bedtime stories. |
| YouTube/URL audio download via yt-dlp | Download audio from URL, save to library | Medium | v2 had this (broken). Restoring and improving it is valuable. |
| Content source browser (Librivox, Freie Hoerspiele, etc.) | Browse free audiobook libraries directly in the UI | High | No competitor does this. Curated links to free children's audiobook sources. |

### Gyro Sensor Integration

| Feature | Value Proposition | Complexity | Notes |
|---------|-------------------|------------|-------|
| Gyro configuration in Web UI | Configure tilt gestures without SSH | Low | phonie-gyro has config file; exposing in UI is straightforward |
| Gyro sensitivity profile selection | Choose from 3 pre-configured profiles | Low | phonie-gyro already supports this |
| Gyro calibration trigger from UI | Start calibration process from web interface | Medium | Currently requires SSH; web trigger would be convenient |
| Gyro gesture visualization | Show which direction does what (visual diagram) | Low | Static informational display with current mapping |

### Smart Automation

| Feature | Value Proposition | Complexity | Notes |
|---------|-------------------|------------|-------|
| Scheduled playback (alarm clock mode) | "Play Einschlafgeschichten at 19:00 every day" | High | No competitor in this space does this. Parents would love automated bedtime stories. |
| Volume schedule (quiet hours) | Auto-limit volume during naptime/bedtime hours | Medium | Extension of existing max volume feature with time awareness. |
| Content rotation | "Play a different audiobook each day from this set" | High | Advanced but unique for children's content management. |

---

## Anti-Features

Features to explicitly NOT build. Common mistakes in this domain.

| Anti-Feature | Why Avoid | What to Do Instead |
|--------------|-----------|-------------------|
| Full-featured music library database (artist/album/genre metadata) | Phoniebox content is folders of audiobooks and children's music, not a music collection. Metadata parsing (ID3 tags, MusicBrainz) adds massive complexity for little value. | Use folder names as primary organization. Show cover art from folder. Do NOT build a metadata indexer. |
| Equalizer / DSP / Audio format settings | Parents don't care about FLAC vs MP3, sample rates, or EQ curves. moOde goes deep here and it scares normal users. | Fixed "good enough" audio settings. One optional "bass boost" toggle at most. |
| Multi-room / multi-box sync | Sonos does this, but Phoniebox is one box for one child. Building multi-room adds enormous complexity (clock sync, network discovery). | If someone has two boxes, they manage them independently. |
| Social features (sharing playlists, ratings) | This is a local family device, not a social network. | Keep it simple and private. |
| Streaming service integration UI (Spotify browse/search) | Spotify's API is unstable for OSS, and building a Spotify browser duplicates their app. Same for Tidal, Apple Music. | Support Spotify Connect (box appears as speaker in Spotify app). Let parents use official apps to cast. |
| Complex user management (multiple accounts, permissions matrix) | Overkill for a family device. Three tiers (open/parent/expert) is enough. | Simple PIN protection, not account management. |
| Theme engine / custom CSS editor | Developers love this, parents don't. One clean theme is better than a customizable ugly one. | One well-designed responsive theme. Light/dark mode toggle is acceptable. |
| Real-time audio visualization (spectrum analyzer, waveforms) | Looks cool, wastes RPi resources, adds no value for parents or children (children interact physically, not via screen). | Static cover art is better than animated distractions. |
| Auto-tagging / metadata scraping from online databases | Unreliable, slow, requires internet, often wrong for children's content (German audiobooks poorly covered in MusicBrainz). | Let parents name folders clearly. The folder name IS the metadata. |
| In-app audio editor (trim, merge, split tracks) | Way out of scope. Complex, buggy, and parents can use Audacity if needed. | Upload pre-prepared files. Maybe offer "split by chapter" for single-file audiobooks as a future differentiator. |

---

## Feature Dependencies

```
Core Infrastructure (must exist first):
  API Layer ──> Player Controls
  API Layer ──> Library Browsing
  API Layer ──> Settings UI

Player Controls (foundation):
  Player Controls ──> Now Playing Display
  Player Controls ──> Volume Controls
  Player Controls ──> Progress Bar + Seek

Library Browsing (builds on API):
  Library Browsing ──> Content Type Filters
  Library Browsing ──> Search
  Library Browsing ──> Play from Library

Content Management:
  Library Browsing ──> File Upload
  File Upload ──> Folder Management
  Folder Management ──> Cover Art Assignment

RFID Card Management (builds on Library):
  Library Browsing ──> Card Assignment (need to browse to select content)
  Card Assignment ──> Card Overview ("card wall")
  Card Assignment ──> Card Edit / Delete

Access Control:
  All Features ──> Three-Tier Access (wraps everything)
  Settings UI ──> PIN Protection (parent area gate)

Streaming Integration:
  Library Browsing ──> Stream Catalog
  Stream Catalog ──> Podcast Subscription
  Content Management ──> YouTube/URL Download

Gyro Sensor:
  Settings UI ──> Gyro Configuration
  Gyro Configuration ──> Gyro Calibration
  Gyro Configuration ──> Sensitivity Profiles

Advanced (Phase 2+):
  Player Controls + Card Management ──> Resume Position
  Settings UI ──> Scheduled Playback
  Settings UI ──> Volume Schedule
```

---

## MVP Recommendation

For MVP (Phase 1), prioritize table stakes that fix the biggest v2 pain points. The goal: **parents can manage the box from their phone without reading a manual.**

### MVP Must-Have (Phase 1 Web Interface)

1. **Player controls with now-playing display** -- Play/pause, prev/next, volume slider, progress bar, cover art, track info. This is the "home screen" parents see.
2. **Library browsing with cover art grid** -- Visual browsing replaces v2's text-only folder tree. Search included.
3. **RFID card registration (simplified wizard)** -- The core Phoniebox workflow, but guided instead of overwhelming.
4. **Card overview ("card wall")** -- See all assigned cards at a glance.
5. **Basic settings** -- Max volume, sleep timer, idle shutdown, WiFi, language. PIN-protected parent area.
6. **File upload** -- Upload audio files to folders.
7. **Mobile-first responsive design** -- Entire UI must work perfectly on phone.

### Defer to Phase 2

- **Stream catalog** (ARD/NDR pre-configured): Valuable but not blocking; parents can still enter URLs manually.
- **Gyro sensor configuration**: Works without UI (existing config file). UI is convenience.
- **Podcast RSS subscription**: Complex, can be manually downloaded and uploaded for now.
- **YouTube/URL download**: Was broken in v2 anyway, not expected.
- **Three-tier access model**: Important for polish but MVP can ship with simple PIN for settings.
- **PWA / Add to Home Screen**: Quick win but not MVP-blocking.
- **Real-time WebSocket updates**: Can use polling initially, upgrade later.

### Defer to Phase 3+

- **Resume position per card** (requires backend changes to track position per RFID card)
- **Scheduled playback / volume schedule** (requires backend timer service)
- **Content source browser** (Librivox etc.) (requires external API integration)
- **Drag-and-drop playlist editing** (polish feature)
- **Content rotation** (advanced automation)

---

## Existing v2 Features to Preserve

These features exist in v2 and must NOT be lost in the new web interface:

| v2 Feature | Location | Must Keep |
|---|---|---|
| Player transport controls | inc.loadControls.php | Yes -- rebuild modern |
| Cover art display | inc.loadCover.php | Yes -- make prominent |
| Loaded playlist view | inc.loadedPlaylist.php | Yes -- improve UX |
| Volume control | inc.setVolume.php | Yes -- add slider |
| Audio folder browser with type filters | index.php filter buttons | Yes -- add grid view |
| Card registration (interactive) | cardRegisterNew.php + inc.formCardEdit.php | Yes -- wizard UX |
| Card export/import CSV | rfidExportCsv.php + cardRegisterNew.php upload | Yes -- keep for backup |
| Card edit | cardEdit.php | Yes -- simplify |
| File/folder upload and creation | manageFilesFolders.php | Yes -- improve UX |
| Search | search.php | Yes -- make inline |
| Settings: language, volume, max volume, startup volume, volume step, boot volume | settings.php sections | Yes |
| Settings: sleep timer, stop timer, idle shutdown, shutdown volume reduction | settings.php sections | Yes |
| Settings: WiFi, WLAN IP display | settings.php sections | Yes |
| Settings: second swipe behavior, second swipe pause | settings.php sections | Yes |
| Settings: web UI config, input devices, debug logging | settings.php sections | Yes -- move to expert tier |
| System info | systemInfo.php | Yes -- move to expert tier |
| Shutdown / Reboot | inc.navigation.php | Yes |
| Multi-language (DE, EN, FR, NL) | lang/*.php | Yes -- use i18n framework |
| Bluetooth audio sink toggle | inc.bluetooth.php | Yes -- if enabled |
| RSS/podcast feed | rss-mp3.php | Yes -- improve |

---

## Complexity Estimates (Relative)

| Complexity | Definition | Examples |
|---|---|---|
| Low | < 1 day, standard patterns, no backend changes | Volume slider, sort options, shutdown button, language selector |
| Medium | 1-3 days, some state management or backend integration | Cover art grid, progress bar with seek, file upload with progress, card wizard |
| High | 3-7 days, new backend capability or external integration | Podcast RSS subscription, scheduled playback, content source browser, WebSocket real-time |

---

## Sources

- **Toniebox/mytonies:** Based on training knowledge of the mytonies app (iOS/Android), Toniebox product documentation, and community reviews. Confidence: MEDIUM -- product is well-established but specific feature details may have changed.
- **Volumio:** Based on training knowledge of Volumio 3 web interface and plugin system. Confidence: MEDIUM -- core features stable for years.
- **moOde Audio:** Based on training knowledge of moOde Audio Player 8.x web interface. Confidence: MEDIUM -- niche product, less likely to have changed dramatically.
- **Sonos:** Based on training knowledge of Sonos S2 app. Confidence: MEDIUM -- redesigned in 2024, specific current features may differ.
- **Phoniebox v2:** Verified directly from codebase analysis. Confidence: HIGH -- first-party source code examined.
- **phonie-gyro:** Verified from project memory and ANALYSIS.md. Confidence: HIGH -- user's own project.

**Note:** WebSearch and WebFetch were unavailable during research. All competitor feature claims are based on training data (cutoff: early 2025) and should be treated as MEDIUM confidence. The Phoniebox v2 analysis is HIGH confidence as it was verified directly from the codebase.

# Domain Pitfalls

**Domain:** Legacy web frontend replacement on Raspberry Pi (Phoniebox v2.8.0)
**Researched:** 2026-02-06
**Overall confidence:** MEDIUM (based on codebase analysis + training data; WebSearch unavailable for verification)

---

## Critical Pitfalls

Mistakes that cause rewrites, broken production boxes, or months of wasted work.

---

### Pitfall 1: SPA Bundle Too Heavy for Pi's Browser and Server

**What goes wrong:** A React/Vue/Svelte SPA with typical dependencies (state management, UI library, icons, animation) produces a 500KB-2MB JavaScript bundle. On a Raspberry Pi 3B with 1GB RAM, the Chromium browser on the Pi itself (kiosk mode) or even Lighttpd serving large static files becomes painfully slow. Worse: the Pi is also running MPD, the RFID daemon, GPIO control, and potentially phonie-gyro -- all competing for the same ARM CPU and RAM.

**Why it happens:** Developers build and test on fast machines (Windows/Mac with 16-32GB RAM). The Pi's quad-core ARM Cortex at 1.2-1.4GHz with 1GB RAM is roughly equivalent to a budget smartphone from 2016. A bundle that loads in 200ms on a dev machine takes 2-5 seconds on Pi.

**Consequences:**
- Parents opening the web UI on their phone see a blank screen for seconds while JS parses
- If anyone uses a browser on the Pi itself (kiosk mode), the entire box stutters
- MPD playback may glitch during heavy page loads because Lighttpd + PHP + JS parsing compete for CPU
- Memory pressure can trigger OOM killer, taking down the RFID daemon

**Prevention:**
1. Set a **bundle size budget of 150KB gzipped** for the initial load. Enforce with bundler configuration (Vite's `build.rollupOptions` with `manualChunks`)
2. Choose a lightweight framework: Preact (3KB) or Svelte (compiled, no runtime) over React (42KB runtime) or Vue (33KB)
3. Use **no component library** (no MUI, no Ant Design). Build 10-15 custom components with vanilla CSS. Phoniebox needs a player, a card list, a file browser, and settings -- not a design system
4. Lazy-load admin pages. The player view (90% of usage) should be under 50KB gzipped
5. Test on actual Pi hardware at least weekly during development -- not just at the end
6. Use `lighthouse` or `bundlesize` in CI with Pi-equivalent CPU throttling (4x slowdown)

**Detection (warning signs):**
- `npm run build` output showing chunks over 100KB
- Dev tools "Coverage" tab showing >50% unused JS on player page
- Any import of `@mui/*`, `antd`, `bootstrap`, or similar heavyweight UI libraries
- `node_modules` having 500+ packages for a frontend that shows a play button

**Phase relevance:** Phase 1 (Web Interface) -- must be a founding constraint, not a late optimization.

---

### Pitfall 2: Breaking the Shell Script Bridge During API Replacement

**What goes wrong:** The new SPA needs a JSON API. Developers build a clean REST API (e.g., with Flask, FastAPI, or Express) that directly talks to MPD via python-mpd2 or a socket. This bypasses `playout_controls.sh`, which is the central hub that RFID daemon, GPIO buttons, and phonie-gyro ALL route through. Now the web UI can control MPD directly, but it creates a split-brain: the API changes state that `playout_controls.sh` doesn't know about, and vice versa.

**Why it happens:** `playout_controls.sh` is 1,153 lines of ugly Bash. The natural instinct is "let's not use that." But this script is not just a player controller -- it manages resume position, logging, second-swipe behavior, volume management (amixer vs MPD), and triggers that other components depend on.

**Consequences:**
- RFID card swipe triggers `playout_controls.sh` which sets resume position, but web UI "pause" via direct MPD command doesn't save resume position -- child loses their place in an audiobook
- GPIO "volume up" button goes through `playout_controls.sh` using amixer, but web API uses MPD volume -- two volume systems fight each other
- phonie-gyro calls `mpc` commands directly (yet another path) -- three different control paths with different side effects
- "It works in the browser but the buttons don't work" / "The card worked but the web UI shows wrong state"

**Prevention:**
1. **Keep `playout_controls.sh` as the single source of truth for ALL player commands** in Phase 1. The new API layer should be a thin wrapper that calls `playout_controls.sh` the same way the current PHP API does (via `exec()` / `subprocess`)
2. Document every command that `playout_controls.sh` handles (there are ~30 based on the `determineCommand()` switch in `player.php`). The new API must support ALL of them, not just play/pause/next
3. Only in a later phase (Phase 3+), consider replacing `playout_controls.sh` with a Python service -- but ONLY after all consumers (RFID, GPIO, gyro, web) are migrated to the new API simultaneously
4. Create an **integration test** that verifies: "Web API play" and "RFID card swipe" and "GPIO button" all produce identical state changes

**Detection (warning signs):**
- API endpoint that imports `python-mpd2` or opens a socket to port 6600 directly
- Any command path that doesn't go through `playout_controls.sh`
- Resume position not being saved on web-initiated pause
- Volume behaving differently between web UI and physical buttons

**Phase relevance:** Phase 1 (Web Interface) -- the API layer design is the most critical architectural decision.

---

### Pitfall 3: Carrying Over Command Injection Vulnerabilities

**What goes wrong:** The new API layer wraps `playout_controls.sh` calls. But the developer passes user input (card IDs, folder names, stream URLs) directly into shell commands, exactly like the current PHP code does. The command injection vulnerabilities from `cardRegisterNew.php` (lines 118, 140, 145) and `daemon_rfid_reader.py` (line 99) get replicated in the new codebase.

**Why it happens:** When wrapping shell scripts, the easiest pattern is string interpolation:
```python
subprocess.call(f"playout_controls.sh --cardid={cardid}", shell=True)  # VULNERABLE
```
This is exactly what the current code does. Copy-paste or "just make it work like before" perpetuates the vulnerability.

**Consequences:**
- A malicious RFID card (crafted card ID containing shell metacharacters) could execute arbitrary commands as root (since PHP runs `sudo` before shell commands)
- A crafted stream URL or folder name in the web UI could wipe the SD card
- On a children's device in a home network, the attack surface is limited -- but any IoT device on a network is a potential pivot point

**Prevention:**
1. **Never use `shell=True`** in Python subprocess calls. Use argument lists:
   ```python
   subprocess.call(['/path/to/playout_controls.sh', '-c=playerplay'], shell=False)
   ```
2. **Whitelist all command names** in the API layer. The `determineCommand()` switch in the current `player.php` already does this for player commands -- replicate that pattern (only allow known commands, reject everything else)
3. **Validate card IDs** as numeric-only (regex `^[0-9]+$`) before passing to any script
4. **Validate folder names** against a strict pattern (alphanumeric + hyphens + underscores only)
5. **Never pass user input through `sed`** the way `cardRegisterNew.php` does on lines 145 and 253. Use proper config file writing (Python `configparser`, or write files directly)
6. Create a **security boundary**: all shell calls go through ONE function that sanitizes arguments

**Detection (warning signs):**
- `shell=True` anywhere in the new codebase
- String concatenation or f-strings building shell commands
- User input reaching `exec()`, `os.system()`, or `subprocess` without validation
- Direct `sed` or `echo` commands with interpolated user values

**Phase relevance:** Phase 1 (Web Interface) AND Phase 4 (Security) -- the new API MUST NOT introduce new injection vectors, even if the old ones aren't fixed yet.

---

### Pitfall 4: Polling MPD Too Aggressively (or Not Enough)

**What goes wrong:** The current frontend polls `api/player.php` every 5 seconds (`loadStatusRepeat` in `jukebox.js`), with client-side time interpolation between polls. A new SPA developer either:
- (a) Polls every 500ms for "real-time" feel, creating 120 HTTP requests/minute per connected browser, each spawning a PHP process + MPD socket + shell exec for volume -- overwhelming the Pi's CPU
- (b) Switches to WebSockets or SSE without understanding that this requires a persistent server process (Lighttpd + PHP doesn't support this natively)

**Why it happens:** Modern web dev assumes WebSocket support (nginx + Node.js makes it trivial). But Phoniebox runs Lighttpd + PHP, which is a CGI model -- no persistent connections. Adding a WebSocket server means adding a new process (Node.js or Python) that must also run on the Pi's limited resources.

**Consequences:**
- Aggressive polling: Pi CPU sits at 30-60% just serving status requests, less headroom for MPD playback, RFID reading, and GPIO
- WebSocket approach: Requires running an additional server process (e.g., a small Python/Node WebSocket server alongside Lighttpd), increasing memory usage by 30-80MB and adding deployment complexity
- SSE approach: Better than WebSocket for this use case (server-push, no bidirectional needed), but still requires a persistent process and won't work with Lighttpd's PHP CGI model

**Prevention:**
1. **Phase 1: Keep polling, but optimize it.** Poll every 3-5 seconds (like current code) with client-side interpolation. This is good enough for a music player -- 3-second delay on track change is acceptable
2. **Optimize the status endpoint**: Read MPD status via a single socket command batch (the current `execMPDCommand("status\ncurrentsong\nclose")` is already batched -- keep this pattern). Avoid spawning shell scripts for status reads
3. **If adding real-time features later**: Use SSE (Server-Sent Events) via a lightweight Python process (Flask-SSE or raw `asyncio` HTTP) that subscribes to MPD's idle protocol. MPD natively supports `idle` command that blocks until state changes -- this is the correct pattern
4. **Never poll from multiple components independently**: One status poll, shared state. Don't have the player widget poll and the volume widget poll separately
5. **Adaptive polling**: Poll frequently (1s) when playing, infrequently (10s) when paused/stopped

**Detection (warning signs):**
- Network tab showing more than 20 requests/minute during idle playback
- Pi CPU above 15% when nothing is happening except serving the web UI
- Multiple independent `setInterval` timers for different status aspects
- WebSocket library in `package.json` without a clear WebSocket server implementation

**Phase relevance:** Phase 1 (Web Interface) -- polling strategy must be decided upfront, not retrofitted.

---

### Pitfall 5: The Development-to-Pi Deployment Gap

**What goes wrong:** The developer builds the SPA on Windows (this project's dev environment), tests in Chrome on Windows, and the app works perfectly. Then deploys to the Pi and discovers:
- Node.js/npm is not installed on the Pi (and shouldn't be -- it's a production device)
- The build output assumes paths that don't exist on Pi (`C:\Users\...` in source maps)
- The Pi runs Lighttpd, not nginx or a dev server -- SPA routing (history API) breaks
- `fetch()` calls use `localhost:3000` (dev proxy) instead of relative paths
- Font files and images are 5MB because nobody checked the production build

**Why it happens:** There's no "compile and deploy" pipeline established. On the current v2, PHP files are just copied to the Pi and they work. An SPA requires a build step (`npm run build`) that produces static files, and those files need to be served correctly by Lighttpd.

**Consequences:**
- "Works on my machine" syndrome
- Days lost debugging Lighttpd URL rewriting for SPA client-side routing
- Build artifacts accidentally committed to git (or not committed, and lost)
- No way to test on Pi without SSH + manual copy, slowing iteration to minutes per change

**Prevention:**
1. **Establish the build-and-deploy pipeline in week 1**, before writing any UI code:
   - `npm run build` produces static files in `htdocs-new/` (or similar)
   - A deploy script (`scp` or `rsync`) copies build output to Pi
   - Lighttpd config serves the SPA with a fallback to `index.html` for client-side routing
2. **Use hash-based routing** (`/#/settings`) instead of history API routing (`/settings`). Hash routing works without any server configuration -- critical for Lighttpd where URL rewriting is awkward
3. **All API calls use relative paths** (`/api/player.php` not `http://localhost:3000/api/player.php`). Use Vite's proxy config for development only
4. **Pre-built static files get committed to git** (or use a GitHub Action to build). The Pi should never need Node.js installed
5. **Test the production build locally** before deploying: `npx serve dist/` to verify the built files work without the dev server
6. **Cross-platform script compatibility**: Deploy scripts must work from Windows (use PowerShell or cross-platform tools, not bash-only scripts)

**Detection (warning signs):**
- No `deploy` script exists after first week
- API URLs containing `localhost` and port numbers in source code
- SPA returning 404 on page refresh (history routing without server config)
- `node_modules` or `.env` files on the Pi
- Build output not tested outside of `npm run dev`

**Phase relevance:** Phase 1 (Web Interface) -- Sprint 0 / first task. Everything else depends on this.

---

### Pitfall 6: File Upload Crashes on Pi's Limited Storage and Memory

**What goes wrong:** Parents upload audiobooks (200-800MB per book) or music collections through the web UI. The upload goes through Lighttpd -> PHP -> filesystem. PHP's default `upload_max_filesize` is 2MB. Even after increasing it, a 500MB upload through PHP loads the entire file into RAM, exhausting the Pi's 1GB (shared with GPU). The upload process gets OOM-killed, potentially taking down MPD mid-playback. The child's music stops because a parent uploaded a file.

**Why it happens:** PHP file upload is designed for small files (profile pictures, documents). Large file uploads on PHP require chunked upload handling, which the current code doesn't implement. The new SPA inherits this limitation if it continues to upload through PHP/Lighttpd.

**Consequences:**
- Upload of files >100MB fails silently or crashes Lighttpd
- Pi becomes unresponsive during large uploads (CPU + I/O + RAM all maxed)
- SD card fills up without warning (no space check before upload)
- Partial uploads leave corrupted files
- MPD playback stutters or stops during upload I/O

**Prevention:**
1. **Phase 1 (MVP): Don't implement file upload through the web UI at all.** Users can upload via SMB/Samba (already configured on most Phonieboxes) or SCP. Document this clearly. File upload is a Phase 3 feature
2. **When implementing upload later**: Use chunked upload (tus protocol or resumable.js) that sends 1MB chunks. The server writes each chunk to disk immediately without buffering in RAM
3. **Check available disk space** before accepting upload: `df` on the audio folder, reject if <500MB free
4. **Set upload size limits** in Lighttpd config AND the API layer (defense in depth)
5. **Run uploads at low I/O priority** (`ionice -c3`) so MPD playback isn't affected
6. **Show upload progress** in the UI -- parents need to know a 500MB upload will take 5 minutes over WiFi

**Detection (warning signs):**
- PHP `$_FILES` handling for audio files (means entire file buffered in memory)
- No disk space check before write operations
- Upload endpoint without size limits
- No progress indication for uploads
- Pi becoming unresponsive when testing file upload

**Phase relevance:** Phase 3 (Card Management) -- explicitly defer from Phase 1.

---

## Moderate Pitfalls

Mistakes that cause delays, user frustration, or accumulate as technical debt.

---

### Pitfall 7: Over-Engineering Authentication for a Family Device

**What goes wrong:** The developer implements a full authentication system: user accounts, password hashing, JWT tokens, session management, CSRF protection, password reset flow. This takes 2-3 weeks and creates a login screen that parents must navigate every time they want to skip a track on their child's music box.

**Why it happens:** "Security best practices" training says: always authenticate, always use HTTPS, always hash passwords. These are correct for internet-facing applications. But a Phoniebox runs on a home WiFi network, used by 2-5 family members, with no internet exposure.

**Consequences:**
- Parents abandon the web UI because logging in on their phone every time is friction
- Token expiry causes "your session has expired" errors when a parent picks up their phone after an hour
- The child somehow triggers the login screen and can't get past it
- HTTPS is needed for secure auth, but HTTPS on a local IP requires a self-signed certificate, which browsers flag as dangerous -- parents see "THIS SITE IS NOT SECURE" warnings

**Prevention:**
1. **Three-tier access model (from project spec):**
   - **Open tier**: Player controls (play/pause/skip/volume). No auth. Anyone on the network can control playback. This is the 90% use case
   - **PIN tier**: Card management, settings changes. Simple 4-6 digit PIN, stored as a hash. No username. Entered once per session (stored in `sessionStorage`, cleared on tab close)
   - **Expert tier**: System settings, SSH-level operations. Require full password or SSH access
2. **No user accounts**. This is a family device, not a multi-tenant app. One PIN for admin functions
3. **No HTTPS** for Phase 1. The device is on a local network. The PIN protects against curious children, not network attackers
4. **Session persistence**: Store the PIN-verified state in a browser cookie with a long expiry (7 days). Parents shouldn't need to re-enter the PIN on their regular phone

**Detection (warning signs):**
- `bcrypt`, `jwt`, or `passport` in dependencies for a local-network device
- A "Create Account" or "Register" page
- HTTPS/TLS certificate setup in the deployment guide
- More than 50 lines of code dedicated to authentication in Phase 1

**Phase relevance:** Phase 1 (minimal -- open access for player, PIN for admin) and Phase 4 (Roles & Security).

---

### Pitfall 8: PWA/Service Worker Confusion on Local Network Devices

**What goes wrong:** The developer adds a Service Worker for "offline support" and "app-like experience." But a Phoniebox is always accessed over the local network -- it IS the server. The Service Worker caches old versions of the SPA, and after a deploy, parents see the old UI until they clear their browser cache. Worse: the Service Worker intercepts API calls and serves stale cached responses, showing the wrong currently-playing track.

**Why it happens:** Every modern SPA template (Create React App, Vite PWA plugin) includes Service Worker boilerplate. The "Add to Home Screen" feature is appealing for a device parents use frequently. But the caching model assumes the app needs to work offline -- which doesn't make sense when the app only works while connected to the Pi's network anyway.

**Consequences:**
- After deploying an update, some users see old UI, others see new -- debugging nightmare
- API responses get cached, showing stale player state
- "Clear cache" becomes part of the update instructions -- unacceptable for non-technical parents
- Service Worker update lifecycle (install -> waiting -> activate) is notoriously hard to get right

**Prevention:**
1. **Do NOT add a Service Worker in Phase 1.** The Phoniebox web UI only works when connected to the Pi's network. There is no "offline" scenario
2. **Do add a Web App Manifest** (for "Add to Home Screen" on phones). This does NOT require a Service Worker
3. **Set proper HTTP cache headers** on Lighttpd:
   - HTML files: `Cache-Control: no-cache` (always check for updates)
   - JS/CSS with content hashes in filenames: `Cache-Control: max-age=31536000` (immutable, Vite does this by default)
4. **If adding a Service Worker later** (Phase 5+): Use a "network-first" strategy for API calls and a "stale-while-revalidate" strategy for static assets. Never cache API responses

**Detection (warning signs):**
- `workbox` or `vite-plugin-pwa` in dependencies
- `navigator.serviceWorker.register()` in application code
- Users reporting they see old UI after update
- API calls returning data that doesn't match current player state

**Phase relevance:** Phase 1 -- explicitly do NOT add. Revisit in Phase 5+ if needed.

---

### Pitfall 9: RFID Card Registration Workflow Breaks in SPA Model

**What goes wrong:** The current card registration flow is a polling hack: `ajax.refresh_id.php` reads `shared/latestID.txt` every 1 second to detect when a card is swiped on the reader. The page shows the last-swiped card ID in an input field. In the new SPA, this workflow breaks because:
- The SPA doesn't know when a card is swiped (no push mechanism)
- The `latestID.txt` file is written by the RFID daemon, read by PHP -- the new API needs to replicate this file-based IPC
- If the user is on the card registration page and swipes a card, the RFID daemon also triggers playback -- the card both "registers" and "plays," confusing the workflow

**Why it happens:** The RFID card flow is a side effect of the daemon's main loop. There's no "registration mode" in the RFID daemon -- it always triggers playback. The web UI just happens to also read the last card ID as a separate concern.

**Consequences:**
- Card registration page doesn't show the swiped card ID (if the new API doesn't read `latestID.txt`)
- Swiping a card to register it also starts playing music -- parent has to stop playback, then register the card
- Race condition: parent swipes card, UI shows ID, parent fills in folder assignment, but during that time another card is swiped (child playing), overwriting `latestID.txt`
- No feedback: parent doesn't know if the card was successfully read by the RFID reader

**Prevention:**
1. **Preserve the `latestID.txt` file-reading mechanism** in the new API. Add an endpoint like `GET /api/rfid/latest` that reads and returns this file's contents
2. **Poll this endpoint at 1-second intervals** ONLY on the card registration page (not globally)
3. **Document the dual-trigger problem**: When a card is swiped during registration, it WILL trigger playback. This is a known limitation of v2. Fixing it requires modifying `daemon_rfid_reader.py` to support a "registration mode" -- defer to Phase 3
4. **For Phase 3 (Card Management)**: Add a "registration mode" flag file (e.g., `settings/registration_mode`). When set, the RFID daemon writes the card ID to `latestID.txt` but does NOT trigger playback. The web UI sets/clears this flag
5. **Show clear UI feedback**: "Waiting for card..." -> "Card detected: 0012345678" -> "Assign content to this card"

**Detection (warning signs):**
- Card registration page that doesn't auto-detect swiped cards
- No API endpoint for reading latest card ID
- RFID daemon modifications that break the existing card-to-playback flow
- No handling of the "card triggers playback during registration" conflict

**Phase relevance:** Phase 1 (basic: read latest card ID) and Phase 3 (improved: registration mode).

---

### Pitfall 10: Replacing the Frontend Without Feature Parity

**What goes wrong:** The new SPA launches with a beautiful player view and basic controls. But it's missing features that the old PHP UI had: chapter navigation (audiobook seeking), playlist reordering, audio folder browsing, sleep timer configuration, WiFi settings, system info page, cover art display, and CSV card export/import. Parents who used the old UI feel the new one is a downgrade.

**Why it happens:** The old PHP UI has 77 files accumulated over years. Nobody catalogs all features before starting the replacement. The developer focuses on the most visible features (player, volume) and misses the long tail of utility features that some users depend on.

**Consequences:**
- Users must keep the old PHP UI accessible for features not yet in the new SPA -- but running both UIs adds confusion
- "Where did feature X go?" complaints
- Pressure to rush missing features leads to sloppy implementation
- Feature parity takes 3x longer than estimated because the old UI's feature surface is larger than it appears

**Prevention:**
1. **Create a feature inventory** of every PHP page and every AJAX endpoint. Map each to: "Must have in Phase 1" / "Defer to Phase N" / "Drop entirely." Based on the current codebase:
   - **Phase 1 must-haves**: Player controls (play/pause/next/prev/stop), volume control, current track display with elapsed time, chapter navigation (audiobooks are a core use case), cover art, basic playlist view
   - **Phase 1 nice-to-haves**: Repeat/shuffle toggles, sleep timer display
   - **Phase 2+**: Card registration, folder browser, stream management, system info, WiFi settings
   - **Drop**: Mopidy status page (not used), debug printing in production
2. **Run both UIs in parallel** during migration: new SPA at `/app/` or root, old PHP UI at `/legacy/`. This is trivial with Lighttpd URL routing. Remove old UI only when feature parity is confirmed
3. **Test with actual users** (the parents) after Phase 1. Their feedback about missing features is more valuable than any inventory

**Detection (warning signs):**
- No feature inventory document exists
- Old PHP UI has been deleted or disabled before new SPA covers its features
- Users asking "how do I do X now?"
- Chapter navigation missing (audiobooks are the #1 use case for Phonieboxes)

**Phase relevance:** Phase 1 (define scope) and all subsequent phases.

---

### Pitfall 11: Ignoring Mobile-First for a Phone-Primary UI

**What goes wrong:** The developer designs the UI on a desktop browser at 1920x1080. The player controls look great with wide layouts, sidebars, and hover effects. Then a parent opens it on their iPhone SE (375px wide) and the volume slider is too small to hit, the track name is truncated, and the navigation requires horizontal scrolling.

**Why it happens:** Desktop-first development is the default mode (developer has a big monitor). The current Bootstrap 3 UI is at least nominally responsive, so the new SPA seems like it should be "even better." But custom SPA layouts without Bootstrap's grid need explicit mobile design.

**Consequences:**
- Core UX failure: "Parents can effortlessly manage their children's music box from any phone" -- the project's core value proposition -- is broken
- Touch targets too small (minimum 44x44px per Apple HIG, 48x48dp per Material)
- Player controls fighting for space on narrow screens
- Landscape orientation on phones wastes vertical space on a player that needs it

**Prevention:**
1. **Design mobile-first**: Start at 375px width, expand up. The primary use case is a parent on their phone, not a desktop browser
2. **Test on real phone** from day one. Chrome DevTools mobile emulation misses touch target size, scroll momentum, and safe area insets
3. **Minimum touch target: 48x48px** for all interactive elements (play, pause, volume buttons)
4. **No hover effects** as the only interaction feedback -- phones don't have hover
5. **Volume control**: Use a vertical slider or large +/- buttons, not a thin horizontal slider (hard to drag precisely on phone)
6. **Player view as a single screen**: No scrolling needed for play/pause/skip/volume on the smallest phone (320px)

**Detection (warning signs):**
- CSS media queries that start with `max-width` (desktop-first) instead of `min-width` (mobile-first)
- Interactive elements smaller than 44x44px
- Hover-only interactions (`:hover` without `:active` or `:focus` equivalent)
- Horizontal scrolling on any page at 375px width
- "Let me check on desktop first, I'll do mobile later"

**Phase relevance:** Phase 1 (Web Interface) -- non-negotiable from day one.

---

### Pitfall 12: MPD State and Web UI State Diverge

**What goes wrong:** The SPA maintains its own player state in a frontend store (Zustand, Redux, or Svelte store). When a user presses "pause" in the web UI, the store updates immediately (optimistic update) and sends a command to the API. But the API call goes through `playout_controls.sh` which takes 200-500ms. Meanwhile, an RFID card is swiped, starting a new playlist. The frontend store shows "paused" but MPD is actually playing a new track. The UI is now wrong until the next poll.

**Why it happens:** MPD is the single source of truth for player state, but there are FOUR ways to change that state: web UI, RFID card, GPIO buttons, and phonie-gyro. The web UI only knows about its own actions.

**Consequences:**
- UI shows wrong state (paused when playing, wrong track title)
- User presses play/pause repeatedly because the UI seems unresponsive
- Duplicate commands sent to MPD causing glitchy playback
- "The web UI lies" -- parents lose trust in the interface

**Prevention:**
1. **MPD is truth, frontend is display.** The frontend should NOT maintain its own player state independently. After every command, immediately poll MPD for the actual state
2. **Optimistic updates with rollback**: Show the expected state change immediately for responsiveness, but verify with the next poll (3-5 seconds). If MPD state doesn't match expectation, update to MPD's state (this handles external state changes from RFID/GPIO/gyro)
3. **Debounce rapid commands**: If the user taps "next" 5 times quickly, queue them and send sequentially with 200ms spacing (MPD's command processing is sequential anyway)
4. **Show a "syncing" indicator** when a command is in-flight (subtle spinner on the control that was pressed). This prevents impatient double-taps

**Detection (warning signs):**
- Frontend store with `isPlaying`, `currentTrack`, `volume` that doesn't sync from MPD
- Commands that update local state without waiting for server confirmation
- No handling of "external state change" (state changed by something other than this browser tab)
- Tests that mock MPD responses instead of testing against real MPD state flow

**Phase relevance:** Phase 1 (Web Interface) -- core state management pattern.

---

### Pitfall 13: Lighttpd Configuration for SPA Serving

**What goes wrong:** The developer builds the SPA, copies files to the Pi, but:
- `/settings` returns 404 instead of serving `index.html` (SPA routing broken)
- API calls to `api/player.php` fail because the PHP handler isn't configured for the new directory structure
- CORS errors because the dev server and Lighttpd have different origins
- Static assets (JS, CSS, images) aren't gzip-compressed, doubling load times
- Lighttpd doesn't set correct MIME types for `.woff2` fonts or `.webp` images

**Why it happens:** Nobody configures Lighttpd correctly because all development happens on Vite's dev server. The first real Lighttpd test happens during deployment, and things break.

**Consequences:**
- Days of debugging server configuration instead of building features
- Broken SPA on Pi while everything works on dev machine
- Performance issues from missing compression
- Confusion between "is this a frontend bug or a server config bug?"

**Prevention:**
1. **Use hash-based routing** (`/#/settings`). This eliminates the need for server-side URL rewriting entirely. Lighttpd serves `index.html` and the frontend handles all routing. This is the simplest and most reliable approach
2. **Create the Lighttpd config file as part of Phase 1, Sprint 0**:
   ```lighttpd
   # Serve new SPA from /app/
   $HTTP["url"] =~ "^/app/" {
       server.document-root = "/home/pi/RPi-Jukebox-RFID/htdocs-app/"
   }
   # Keep old PHP API accessible
   $HTTP["url"] =~ "^/api/" {
       server.document-root = "/home/pi/RPi-Jukebox-RFID/htdocs/"
   }
   # Enable gzip
   server.modules += ("mod_deflate")
   deflate.mimetypes = ("text/html", "text/css", "application/javascript", "application/json")
   ```
3. **Proxy API calls in development**: Vite's `server.proxy` to forward `/api/*` to the Pi's IP during development
4. **Test the production build on Lighttpd** (on Pi) at least once per week

**Detection (warning signs):**
- No Lighttpd configuration file in the repository
- SPA works on `npm run dev` but returns 404s on Pi
- CORS errors in browser console
- API calls using full URLs instead of relative paths

**Phase relevance:** Phase 1, Sprint 0 -- before any UI work begins.

---

## Minor Pitfalls

Mistakes that cause annoyance or minor rework, but are easily fixable.

---

### Pitfall 14: Not Testing on Multiple Phone Browsers

**What goes wrong:** The SPA works in Chrome on Android but breaks in Safari on iPhone (which most parents likely use). Common Safari issues:
- `gap` in flexbox not supported in older Safari (iOS < 14.5)
- `input type="range"` (volume slider) has different styling/behavior
- Safe area insets for iPhones with notches cut off UI elements
- `audio` API differences if using Web Audio for preview
- 300ms tap delay on older Safari versions without `touch-action: manipulation`

**Prevention:**
1. Test on Safari (iPhone/iPad) and Chrome (Android) at minimum
2. Add `viewport-fit=cover` and `env(safe-area-inset-*)` CSS for iPhones with notches
3. Use `touch-action: manipulation` globally to eliminate tap delay
4. Avoid CSS features that Safari lags on (check caniuse.com)

**Phase relevance:** Phase 1 -- test early, not at the end.

---

### Pitfall 15: Forgetting the Sleep Timer and Idle Power States

**What goes wrong:** The new player view doesn't show or control the sleep timer. Parents can't set "turn off after 30 minutes" for bedtime. Meanwhile the Pi runs all night playing silence or repeating an album. On a battery-powered Phoniebox (mobile setup), this drains the battery.

**Prevention:**
1. Include sleep timer in the Phase 1 player view (it's a core parenting feature)
2. Show remaining sleep timer time prominently
3. The current implementation is in `playout_controls.sh` (`-c=playerstopafter`) -- just expose it through the API

**Phase relevance:** Phase 1 -- this is a table-stakes parent feature, not a nice-to-have.

---

### Pitfall 16: Cover Art Path Assumptions

**What goes wrong:** The current cover art system reads images from audio folders. The new SPA constructs image URLs that don't match the Pi's folder structure, or serves images without caching headers, causing re-downloads on every page view (significant on slow Pi I/O).

**Prevention:**
1. Replicate the existing `api/cover.php` logic in the new API
2. Set `Cache-Control: max-age=86400` on cover art responses (covers don't change frequently)
3. Serve cover art at appropriate resolution (200x200px is enough for a phone screen, don't serve the original 3000x3000 album art)
4. Show a default placeholder immediately while cover art loads (avoids layout shift)

**Phase relevance:** Phase 1 -- cover art is part of the player view.

---

## Phase-Specific Warnings

| Phase Topic | Likely Pitfall | Mitigation |
|-------------|---------------|------------|
| Phase 1: SPA framework choice | Choosing React+MUI (like v3 did) adds 200KB+ runtime weight | Use Svelte or Preact with custom CSS. Budget: 150KB gzipped total |
| Phase 1: API layer | Bypassing playout_controls.sh breaks RFID/GPIO integration | Thin wrapper over existing shell scripts; do NOT talk to MPD directly |
| Phase 1: Build pipeline | No deploy script = "works on my machine" for weeks | Establish scp/rsync deploy to Pi in Sprint 0, before any UI code |
| Phase 1: Player state | Optimistic UI updates without MPD verification | Always treat MPD as source of truth; poll to verify after commands |
| Phase 1: Routing | History API routing breaks on Lighttpd without rewrite rules | Use hash-based routing (#/) -- zero server config needed |
| Phase 2: Gyro config | Breaking phonie-gyro's mpc integration by changing MPD config | phonie-gyro is independent (calls mpc directly); verify it still works after any MPD-related changes |
| Phase 3: Card registration | RFID daemon triggers playback when user wants to register | Add "registration mode" flag that suppresses playback trigger |
| Phase 3: File upload | Large audiobook uploads crash Pi (OOM) | Chunked upload with disk space checks; defer to Phase 3, not Phase 1 |
| Phase 4: Authentication | Full auth system creates friction for family device | PIN-based admin access, open player access. No user accounts |
| Phase 4: Security | New API replicates command injection from old PHP code | No shell=True, whitelist commands, validate all input as first-class concern |
| Phase 5: Spotify | Expecting Spotify to "just work" | Spotify is an industry-wide unsolved problem; plan for Spotify Connect at best |
| Phase 6: Updates | SPA cached in Service Worker, update doesn't reach users | No Service Worker. Content-hashed filenames + no-cache on HTML |

---

## Sources

- Codebase analysis: `C:\Users\herrkraft\Repos\rfid-jukebox\htdocs\api\player.php` (polling and command pattern)
- Codebase analysis: `C:\Users\herrkraft\Repos\rfid-jukebox\htdocs\api\common.php` (shell execution pattern, MPD socket)
- Codebase analysis: `C:\Users\herrkraft\Repos\rfid-jukebox\htdocs\cardRegisterNew.php` (command injection, file upload, card registration)
- Codebase analysis: `C:\Users\herrkraft\Repos\rfid-jukebox\htdocs\inc.processCheckCardEditRegister.php` (card workflow, security issues)
- Codebase analysis: `C:\Users\herrkraft\Repos\rfid-jukebox\scripts\daemon_rfid_reader.py` (RFID flow, shell=True vulnerability)
- Codebase analysis: `C:\Users\herrkraft\Repos\rfid-jukebox\htdocs\js\jukebox.js` (current polling strategy: 5s interval + interpolation)
- Codebase analysis: `C:\Users\herrkraft\Repos\rfid-jukebox\htdocs\ajax.refresh_id.php` (card ID detection via file-based IPC)
- Architecture docs: `C:\Users\herrkraft\Repos\rfid-jukebox\.planning\codebase\ARCHITECTURE.md`
- Stack docs: `C:\Users\herrkraft\Repos\rfid-jukebox\.planning\codebase\STACK.md`
- Concerns docs: `C:\Users\herrkraft\Repos\rfid-jukebox\.planning\codebase\CONCERNS.md`
- Project analysis: `C:\Users\herrkraft\Repos\rfid-jukebox\.planning\ANALYSIS.md`

**Confidence note:** All pitfalls are derived from direct codebase evidence and established patterns in embedded/IoT web development. WebSearch was unavailable for external verification, so community-sourced patterns (e.g., specific Lighttpd gotchas, Pi 3B benchmarks) are based on training data and should be validated. The command injection vulnerabilities and shell integration patterns are HIGH confidence as they are directly observed in the code.

---

*Pitfalls research: 2026-02-06*