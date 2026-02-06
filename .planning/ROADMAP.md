# Roadmap: Phoniebox Web-Interface

## Overview

This roadmap replaces the outdated PHP/jQuery web interface of the Phoniebox v2.8.0 with a modern, mobile-first SPA. The journey starts with the API gateway and deployment pipeline (foundation that everything else depends on), builds up through real-time player state, the player UI, library browsing, card management, content management, parent and expert settings, access control, and finally gyro integration and PWA polish. All existing backend systems (RFID daemon, GPIO, MPD, playout_controls.sh) remain untouched -- the new frontend wraps them through a FastAPI gateway.

## Phases

**Phase Numbering:**
- Integer phases (1, 2, 3): Planned milestone work
- Decimal phases (2.1, 2.2): Urgent insertions (marked with INSERTED)

Decimal phases appear between their surrounding integers in numeric order.

- [ ] **Phase 1: API Foundation & Deploy Pipeline** - FastAPI gateway wrapping shell scripts, Nginx config, build-deploy pipeline to Pi
- [ ] **Phase 2: Real-Time Layer** - WebSocket hub with MPD idle bridge for instant player state updates
- [ ] **Phase 3: Player UI** - Mobile-first player view with controls, volume, progress, cover art, now-playing
- [ ] **Phase 4: Library Browser** - Cover art grid/list browsing with search, sort, filter, and direct playback
- [ ] **Phase 5: Card Management** - Card wall, scan-and-select registration wizard, edit/delete assignments
- [ ] **Phase 6: Content Management** - File upload, folder/playlist CRUD, cover art assignment
- [ ] **Phase 7: Parent Settings** - Volume limits, sleep timer, startup volume, idle shutdown, playback tweaks
- [ ] **Phase 8: Expert Settings** - WiFi, system info, GPIO/MPD config, export/import, debug logging, language
- [ ] **Phase 9: Access Control** - Three-tier model with PIN protection for parent and expert areas
- [ ] **Phase 10: Gyro Integration** - Gyro sensor configuration, sensitivity profiles, calibration, gesture visualization
- [ ] **Phase 11: PWA & Polish** - Offline capability, PWA install, resume position, performance hardening

## Phase Details

### Phase 1: API Foundation & Deploy Pipeline
**Goal**: A working API server runs on the Pi, wrapping shell scripts safely, with a repeatable build-deploy pipeline from the dev machine
**Depends on**: Nothing (first phase)
**Requirements**: INFR-01, INFR-02, INFR-04, INFR-05
**Success Criteria** (what must be TRUE):
  1. FastAPI server starts on the Pi and responds to health check at `/api/health`
  2. Player control endpoints (play, pause, next, prev, volume) call `playout_controls.sh` via subprocess without `shell=True` and return correct responses
  3. Nginx serves a static placeholder page and proxies `/api/` requests to FastAPI
  4. A single command on the dev machine builds and deploys to the Pi via rsync/scp
  5. SQLite database stores configuration, replacing flat-file reads for settings consumed by the API
**Plans**: TBD

Plans:
- [ ] 01-01: FastAPI project structure, shell service wrapper, MPD service wrapper
- [ ] 01-02: Player REST endpoints (play/pause/next/prev/volume/status)
- [ ] 01-03: Nginx config, systemd service, build-deploy script
- [ ] 01-04: SQLite config migration (settings files to database)

### Phase 2: Real-Time Layer
**Goal**: The browser receives instant player state updates when anything changes (physical button, RFID card, or web UI action)
**Depends on**: Phase 1
**Requirements**: PLAT-05
**Success Criteria** (what must be TRUE):
  1. WebSocket endpoint at `/ws` accepts connections and sends player state on connect
  2. When a physical button or RFID card changes MPD state, all connected browsers update within 500ms
  3. WebSocket reconnects automatically with exponential backoff after connection loss
  4. MPD idle bridge runs as a background task with zero CPU usage when player is idle
**Plans**: TBD

Plans:
- [ ] 02-01: WebSocket connection manager and MPD idle bridge
- [ ] 02-02: RFID event forwarding (file-watch on latestID.txt)

### Phase 3: Player UI
**Goal**: Parents can control playback from their phone with a responsive, touch-friendly player view that shows what is playing in real time
**Depends on**: Phase 1, Phase 2
**Requirements**: PLAY-01, PLAY-02, PLAY-03, PLAY-04, PLAY-05, PLAY-06, PLAY-07, PLAT-01, PLAT-06, PLAT-07, INFR-03
**Success Criteria** (what must be TRUE):
  1. Player view shows cover art, track title, artist/album, and "Track X of Y" on a phone screen (375px width)
  2. Play/pause, previous/next, shuffle, and repeat controls work and reflect current MPD state
  3. Volume slider adjusts volume smoothly and shows current level
  4. Progress bar shows playback position and allows seeking by tap/drag
  5. UI is in German with plain language labels, initial page load completes in under 3 seconds on Pi, and the JS bundle is under 150KB gzipped
**Plans**: TBD

Plans:
- [ ] 03-01: SvelteKit project setup, Tailwind config, WebSocket store, bottom navigation
- [ ] 03-02: NowPlaying component (cover art, metadata, track position)
- [ ] 03-03: Player controls (play/pause, prev/next, shuffle, repeat)
- [ ] 03-04: Volume slider and progress bar with seek
- [ ] 03-05: German i18n setup and bundle size enforcement

### Phase 4: Library Browser
**Goal**: Parents can browse the music library visually and start playback directly from the browser
**Depends on**: Phase 1, Phase 3
**Requirements**: LIB-01, LIB-02, LIB-03, LIB-04, LIB-05, LIB-06
**Success Criteria** (what must be TRUE):
  1. Library displays folders as a cover art grid with album/folder titles
  2. User can switch between grid view and list view
  3. User can search across the library and see results instantly
  4. User can sort by name or recently added, and filter by content type
  5. Tapping a library item starts playback and navigates to the player view
**Plans**: TBD

Plans:
- [ ] 04-01: Library REST endpoints (browse folders, list files, cover art)
- [ ] 04-02: Library grid and list views with cover art
- [ ] 04-03: Search, sort, and filter functionality

### Phase 5: Card Management
**Goal**: Parents can see all RFID card assignments at a glance and register new cards through a guided wizard
**Depends on**: Phase 1, Phase 2, Phase 4
**Requirements**: CARD-01, CARD-02, CARD-03, CARD-04, CARD-05, CARD-06, CARD-07
**Success Criteria** (what must be TRUE):
  1. Card wall displays all registered cards as a grid with cover art and assigned content name
  2. Scan-and-select wizard guides the user: hold card to reader, pick content from library, confirm assignment
  3. User can edit an existing card's assigned content (folder, stream URL, or system command)
  4. User can delete a card assignment
  5. Card can be assigned to a folder, a stream URL, or a system command (volume, shutdown, etc.)
**Plans**: TBD

Plans:
- [ ] 05-01: Card REST endpoints (list, create, update, delete)
- [ ] 05-02: Card wall UI with cover art grid
- [ ] 05-03: Scan-and-select registration wizard with RFID event bridge
- [ ] 05-04: Edit and delete card assignment flows

### Phase 6: Content Management
**Goal**: Parents can upload music, organize folders, and manage playlists through the browser without needing SMB or SSH
**Depends on**: Phase 1, Phase 4
**Requirements**: CONT-01, CONT-02, CONT-03, CONT-04, CONT-05
**Success Criteria** (what must be TRUE):
  1. User can upload audio files via drag-and-drop with a progress indicator, using chunked upload that works within Pi memory limits
  2. User can create, rename, and delete folders
  3. User can create playlists and reorder tracks via drag-and-drop
  4. User can assign cover art to a folder by uploading an image
**Plans**: TBD

Plans:
- [ ] 06-01: Chunked file upload endpoint with disk space checks
- [ ] 06-02: Upload UI with drag-and-drop and progress indicator
- [ ] 06-03: Folder CRUD and cover art assignment
- [ ] 06-04: Playlist creation and drag-and-drop track reordering

### Phase 7: Parent Settings
**Goal**: Parents can configure playback limits and behaviors that protect their children's listening experience
**Depends on**: Phase 1, Phase 3
**Requirements**: SETT-01, SETT-02, SETT-03, SETT-04, SETT-05, SETT-06, PLAY-08
**Success Criteria** (what must be TRUE):
  1. Parent can set a maximum volume limit that the box cannot exceed
  2. Parent can set a sleep timer (countdown) and a scheduled stop time
  3. Parent can configure startup volume, idle shutdown timer, and volume step size
  4. Parent can configure second-swipe RFID behavior (restart vs. resume)
  5. Audiobook resume position works per RFID card (child picks up where they stopped)
**Plans**: TBD

Plans:
- [ ] 07-01: Settings REST endpoints (read/write config values)
- [ ] 07-02: Parent settings UI (volume limits, sleep timer, idle shutdown)
- [ ] 07-03: Resume position tracking per card

### Phase 8: Expert Settings
**Goal**: Advanced users can configure system, network, and hardware settings through the web UI instead of SSH
**Depends on**: Phase 1, Phase 7
**Requirements**: XPRT-01, XPRT-02, XPRT-03, XPRT-04, XPRT-05, XPRT-06, XPRT-07
**Success Criteria** (what must be TRUE):
  1. Expert can view system info (IP, disk space, uptime) and trigger restart/shutdown
  2. Expert can configure WiFi settings from the browser
  3. Expert can configure GPIO pin assignments and MPD/audio settings
  4. Expert can export all settings and card configurations as a file, and import them on another box
  5. Expert can toggle debug logging and select UI language
**Plans**: TBD

Plans:
- [ ] 08-01: System info, restart, shutdown endpoints and UI
- [ ] 08-02: WiFi configuration endpoints and UI
- [ ] 08-03: GPIO and MPD/audio configuration UI
- [ ] 08-04: Export/import settings and card configurations
- [ ] 08-05: Debug logging toggle and language selection

### Phase 9: Access Control
**Goal**: The web UI enforces three tiers of access so children see only the player and parents/experts must enter a PIN
**Depends on**: Phase 3, Phase 7, Phase 8
**Requirements**: AUTH-01, AUTH-02, AUTH-03
**Success Criteria** (what must be TRUE):
  1. Player view is accessible without any authentication (open tier)
  2. Navigating to parent settings requires entering a 4-6 digit PIN
  3. Expert settings require a separate PIN (can differ from parent PIN)
  4. PIN session persists across page reloads but expires after configurable timeout
**Plans**: TBD

Plans:
- [ ] 09-01: PIN storage (bcrypt hash), JWT middleware with tier claims
- [ ] 09-02: PIN entry dialog, tier enforcement on routes, session timeout

### Phase 10: Gyro Integration
**Goal**: Parents can configure the phonie-gyro tilt sensor through the web UI instead of editing config files
**Depends on**: Phase 1, Phase 7
**Requirements**: GYRO-01, GYRO-02, GYRO-03, GYRO-04
**Success Criteria** (what must be TRUE):
  1. Web UI shows current gyro configuration and allows editing
  2. User can select from three sensitivity profiles
  3. User can trigger sensor calibration from the browser and see the result
  4. Gesture visualization shows which tilt direction triggers which action
**Plans**: TBD

Plans:
- [ ] 10-01: phonie-gyro config REST endpoints (read/write config, trigger calibration)
- [ ] 10-02: Gyro settings UI with sensitivity profiles and gesture visualization

### Phase 11: PWA & Polish
**Goal**: The web UI works offline for local music, can be installed as a home screen app, and feels polished
**Depends on**: Phase 3, Phase 4
**Requirements**: PLAT-02, PLAT-03, PLAT-04
**Success Criteria** (what must be TRUE):
  1. User can add the web UI to their phone's home screen and it opens fullscreen like a native app
  2. Service Worker caches the app shell so the UI loads even without internet (local music still controllable)
  3. UI text follows "Leichte Sprache" principles: short sentences, common words, self-explanatory labels
**Plans**: TBD

Plans:
- [ ] 11-01: Service Worker (app shell caching, network-first for API)
- [ ] 11-02: PWA manifest and icons, add-to-home-screen prompt
- [ ] 11-03: Leichte Sprache audit and UI text refinement

## Progress

**Execution Order:**
Phases execute in numeric order: 1 -> 2 -> 3 -> 4 -> 5 -> 6 -> 7 -> 8 -> 9 -> 10 -> 11

| Phase | Plans Complete | Status | Completed |
|-------|----------------|--------|-----------|
| 1. API Foundation & Deploy Pipeline | 0/4 | Not started | - |
| 2. Real-Time Layer | 0/2 | Not started | - |
| 3. Player UI | 0/5 | Not started | - |
| 4. Library Browser | 0/3 | Not started | - |
| 5. Card Management | 0/4 | Not started | - |
| 6. Content Management | 0/4 | Not started | - |
| 7. Parent Settings | 0/3 | Not started | - |
| 8. Expert Settings | 0/5 | Not started | - |
| 9. Access Control | 0/2 | Not started | - |
| 10. Gyro Integration | 0/2 | Not started | - |
| 11. PWA & Polish | 0/3 | Not started | - |
