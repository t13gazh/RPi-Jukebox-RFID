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
