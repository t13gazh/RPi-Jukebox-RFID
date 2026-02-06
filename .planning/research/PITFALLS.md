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
