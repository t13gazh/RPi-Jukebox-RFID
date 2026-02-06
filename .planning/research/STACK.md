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
