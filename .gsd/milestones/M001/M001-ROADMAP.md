# M001: Phoniebox Web-Interface

**Vision:** A modern, mobile-first web interface for the Phoniebox (RPi-Jukebox-RFID v2.8.0) — replacing the outdated PHP/jQuery frontend while keeping the working backend (RFID, GPIO, Gyro, MPD) intact. Parents can effortlessly manage their children's music box from any phone — no manual needed, no terminal required.

## Success Criteria

- New SPA replaces all 77 PHP files in `htdocs/`
- FastAPI gateway wraps `playout_controls.sh` via subprocess (no `shell=True`)
- WebSocket delivers real-time player state updates within 500ms
- Three-tier access control (open player / PIN parent / PIN expert) enforced
- RFID card scan-and-select registration works through browser
- Music upload, folder management, and cover art assignment work via browser
- Gyro sensor configurable through web UI (sensitivity profiles, calibration)
- PWA installable, offline-capable for local music
- JS bundle under 150KB gzipped, initial load under 3 seconds on Pi


## Slices

<!-- PR grouping for upstream contribution (D002):
     PR 1: S01+S02  — API + Real-Time (universal value, standalone backend)
     PR 2: S03+S04  — Player UI + Library (minimum viable web-UI)
     PR 3: S05-S08  — Card/Content Mgmt + Settings (feature parity with old PHP)
     PR 4: S09-S11  — Access Control + Gyro + PWA (polish layer)
-->

### PR 1 — API Foundation & Real-Time Layer

- [ ] **S01: API Foundation & Deploy Pipeline** `risk:medium` `depends:[]`
  > After this: FastAPI server runs on Pi, wraps playout_controls.sh safely, Nginx proxies /api/, one-command deploy works
- [ ] **S02: Real-Time Layer** `risk:medium` `depends:[S01]`
  > After this: WebSocket at /ws pushes MPD state changes to browsers within 500ms, auto-reconnects

### PR 2 — Player UI & Library Browser

- [ ] **S03: Player UI** `risk:medium` `depends:[S02]`
  > After this: Mobile-first player view with controls, volume, progress, cover art works on 375px phone screens
- [ ] **S04: Library Browser** `risk:medium` `depends:[S03]`
  > After this: Parents browse music library as cover art grid/list, search, and start playback directly

### PR 3 — Card/Content Management & Settings

- [ ] **S05: Card Management** `risk:medium` `depends:[S04]`
  > After this: Card wall shows all RFID assignments, scan-and-select wizard registers new cards
- [ ] **S06: Content Management** `risk:medium` `depends:[S05]`
  > After this: Parents upload audio files, create folders/playlists, assign cover art via browser
- [ ] **S07: Parent Settings** `risk:medium` `depends:[S06]`
  > After this: Max volume limit, sleep timer, startup volume, idle shutdown, resume position configurable
- [ ] **S08: Expert Settings** `risk:medium` `depends:[S07]`
  > After this: WiFi, system info, GPIO/MPD config, export/import, debug logging accessible in expert area

### PR 4 — Access Control, Gyro & PWA

- [ ] **S09: Access Control** `risk:medium` `depends:[S08]`
  > After this: Player open without auth, parent/expert areas require separate PINs with session timeout
- [ ] **S10: Gyro Integration** `risk:medium` `depends:[S09]`
  > After this: phonie-gyro configurable via web UI — sensitivity profiles, calibration, gesture visualization
- [ ] **S11: PWA & Polish** `risk:medium` `depends:[S10]`
  > After this: App installable on home screen, offline-capable for local music, Leichte Sprache UI text
