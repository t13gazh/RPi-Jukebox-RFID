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
