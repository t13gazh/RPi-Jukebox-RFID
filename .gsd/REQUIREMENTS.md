# Requirements: Phoniebox Web-Interface

**Defined:** 2026-02-06
**Core Value:** Parents can effortlessly manage their children's music box from any phone -- no manual needed.

## v1 Requirements

Requirements for the modern web interface replacing the v2 PHP frontend.

### Player & Now Playing

- [ ] **PLAY-01**: Player shows play/pause/stop controls
- [ ] **PLAY-02**: Player shows previous/next track controls
- [ ] **PLAY-03**: Volume slider with visual level indicator
- [ ] **PLAY-04**: Shuffle and repeat mode toggles
- [ ] **PLAY-05**: Now-playing display with cover art, title, and track info
- [ ] **PLAY-06**: Progress bar with seek functionality
- [ ] **PLAY-07**: Track position indicator ("Track 3 of 12")
- [ ] **PLAY-08**: Resume position per RFID card (audiobook continues where child stopped)

### Library

- [ ] **LIB-01**: Browse library as cover art grid (tiles)
- [ ] **LIB-02**: Browse library as list view (switchable with grid)
- [ ] **LIB-03**: Search across entire library
- [ ] **LIB-04**: Sort options (alphabetical, recently added)
- [ ] **LIB-05**: Play content directly from library browser
- [ ] **LIB-06**: Filter by content type (music/audiobook/radio/podcast)

### Card Management

- [ ] **CARD-01**: Guided scan-and-select card registration wizard
- [ ] **CARD-02**: Card wall showing all assignments as grid with covers
- [ ] **CARD-03**: Edit existing card assignment
- [ ] **CARD-04**: Delete card assignment
- [ ] **CARD-05**: Assign card to folder/audio content
- [ ] **CARD-06**: Assign card to stream URL
- [ ] **CARD-07**: Assign card to system command (volume, shutdown, etc.)

### Content Management

- [ ] **CONT-01**: Upload audio files via browser with progress indicator
- [ ] **CONT-02**: Create, rename, and delete folders
- [ ] **CONT-03**: Create playlists
- [ ] **CONT-04**: Assign cover art to folders
- [ ] **CONT-05**: Drag-and-drop playlist track reordering

### Access Control

- [ ] **AUTH-01**: Three-tier access model (open player / parent / expert)
- [ ] **AUTH-02**: PIN protection for parent area
- [ ] **AUTH-03**: Separate protection for expert area

### Parent Settings

- [ ] **SETT-01**: Maximum volume limit
- [ ] **SETT-02**: Sleep timer and scheduled stop
- [ ] **SETT-03**: Startup volume
- [ ] **SETT-04**: Idle shutdown timer
- [ ] **SETT-05**: Volume step size configuration
- [ ] **SETT-06**: RFID second swipe behavior

### Expert Settings

- [ ] **XPRT-01**: WiFi configuration
- [ ] **XPRT-02**: System info (IP, disk space, uptime), restart, shutdown
- [ ] **XPRT-03**: GPIO/hardware configuration
- [ ] **XPRT-04**: MPD/audio configuration
- [ ] **XPRT-05**: Export/import settings and card configurations
- [ ] **XPRT-06**: Debug logging toggle
- [ ] **XPRT-07**: Language selection

### Gyro Sensor

- [ ] **GYRO-01**: Gyro sensor configuration in web UI
- [ ] **GYRO-02**: Sensitivity profile selection (3 profiles)
- [ ] **GYRO-03**: Calibration trigger from web UI
- [ ] **GYRO-04**: Gesture visualization (which direction = which action)

### Platform

- [ ] **PLAT-01**: Mobile-first responsive design
- [ ] **PLAT-02**: Offline-capable (local music works without internet)
- [ ] **PLAT-03**: "Leichte Sprache" -- plain language UI, self-explanatory
- [ ] **PLAT-04**: PWA / Add to Home Screen
- [ ] **PLAT-05**: Real-time player state via WebSocket
- [ ] **PLAT-06**: Fast initial load (< 3 seconds on Pi)
- [ ] **PLAT-07**: German as primary UI language

### Infrastructure

- [ ] **INFR-01**: Replace old PHP interface completely
- [ ] **INFR-02**: API gateway (FastAPI) bridging frontend to existing shell backend
- [ ] **INFR-03**: Bundle size budget (150KB gzipped max)
- [ ] **INFR-04**: Nginx as web server (replacing Lighttpd)
- [ ] **INFR-05**: SQLite replacing flat-file configuration (50+ settings files)

## v2 Requirements

Deferred to future milestone. Tracked but not in current roadmap.

### Streaming & Content Acquisition

- **STRM-01**: Pre-configured German stream catalog (ARD/NDR/DLF children's radio)
- **STRM-02**: Podcast RSS subscription with auto-download of new episodes
- **STRM-03**: YouTube/URL audio download via yt-dlp
- **STRM-04**: Content source browser (Librivox, Freie Hoerspiele, etc.)
- **STRM-05**: Internet radio stream management via URL

### Smart Automation

- **AUTO-01**: Scheduled playback (alarm clock mode -- "play bedtime story at 19:00")
- **AUTO-02**: Volume schedule (auto-limit during sleep hours)
- **AUTO-03**: Content rotation ("play different audiobook each day from set")

## Out of Scope

Explicitly excluded. Documented to prevent scope creep.

| Feature | Reason |
|---------|--------|
| Spotify integration | libspotify discontinued, industry-wide broken, no reliable OSS solution |
| Music metadata indexer (ID3, MusicBrainz) | Phoniebox uses folders not music collections -- folder name IS the metadata |
| Equalizer / DSP / audio format settings | Parents don't need audio engineering, fixed good-enough settings |
| Multi-room / multi-box sync | One box per child, enormous complexity for no value |
| Social features (sharing, ratings) | Local family device, not a social network |
| Streaming service browser (Spotify/Tidal search) | Use official apps, APIs are unstable for OSS |
| Complex user management (accounts, permissions) | PIN tiers sufficient for family device |
| Theme engine / custom CSS | One clean theme better than customizable ugly one |
| Audio visualizer (spectrum, waveforms) | Wastes Pi resources, adds no value for parents |
| Auto-tagging from online databases | Unreliable for German children's content |
| In-app audio editor | Out of scope, use Audacity if needed |
| Mobile native app | Web-first, responsive browser UI is sufficient |

## Traceability

Which phases cover which requirements. Updated during roadmap creation.

| Requirement | Phase | Status |
|-------------|-------|--------|
| PLAY-01 | S03: Player UI | Pending |
| PLAY-02 | S03: Player UI | Pending |
| PLAY-03 | S03: Player UI | Pending |
| PLAY-04 | S03: Player UI | Pending |
| PLAY-05 | S03: Player UI | Pending |
| PLAY-06 | S03: Player UI | Pending |
| PLAY-07 | S03: Player UI | Pending |
| PLAY-08 | S07: Parent Settings | Pending |
| LIB-01 | S04: Library Browser | Pending |
| LIB-02 | S04: Library Browser | Pending |
| LIB-03 | S04: Library Browser | Pending |
| LIB-04 | S04: Library Browser | Pending |
| LIB-05 | S04: Library Browser | Pending |
| LIB-06 | S04: Library Browser | Pending |
| CARD-01 | S05: Card Management | Pending |
| CARD-02 | S05: Card Management | Pending |
| CARD-03 | S05: Card Management | Pending |
| CARD-04 | S05: Card Management | Pending |
| CARD-05 | S05: Card Management | Pending |
| CARD-06 | S05: Card Management | Pending |
| CARD-07 | S05: Card Management | Pending |
| CONT-01 | S06: Content Management | Pending |
| CONT-02 | S06: Content Management | Pending |
| CONT-03 | S06: Content Management | Pending |
| CONT-04 | S06: Content Management | Pending |
| CONT-05 | S06: Content Management | Pending |
| AUTH-01 | S09: Access Control | Pending |
| AUTH-02 | S09: Access Control | Pending |
| AUTH-03 | S09: Access Control | Pending |
| SETT-01 | S07: Parent Settings | Pending |
| SETT-02 | S07: Parent Settings | Pending |
| SETT-03 | S07: Parent Settings | Pending |
| SETT-04 | S07: Parent Settings | Pending |
| SETT-05 | S07: Parent Settings | Pending |
| SETT-06 | S07: Parent Settings | Pending |
| XPRT-01 | S08: Expert Settings | Pending |
| XPRT-02 | S08: Expert Settings | Pending |
| XPRT-03 | S08: Expert Settings | Pending |
| XPRT-04 | S08: Expert Settings | Pending |
| XPRT-05 | S08: Expert Settings | Pending |
| XPRT-06 | S08: Expert Settings | Pending |
| XPRT-07 | S08: Expert Settings | Pending |
| GYRO-01 | S10: Gyro Integration | Pending |
| GYRO-02 | S10: Gyro Integration | Pending |
| GYRO-03 | S10: Gyro Integration | Pending |
| GYRO-04 | S10: Gyro Integration | Pending |
| PLAT-01 | S03: Player UI | Pending |
| PLAT-02 | S11: PWA & Polish | Pending |
| PLAT-03 | S11: PWA & Polish | Pending |
| PLAT-04 | S11: PWA & Polish | Pending |
| PLAT-05 | S02: Real-Time Layer | Pending |
| PLAT-06 | S03: Player UI | Pending |
| PLAT-07 | S03: Player UI | Pending |
| INFR-01 | S01: API Foundation & Deploy Pipeline | Pending |
| INFR-02 | S01: API Foundation & Deploy Pipeline | Pending |
| INFR-03 | S03: Player UI | Pending |
| INFR-04 | S01: API Foundation & Deploy Pipeline | Pending |
| INFR-05 | S01: API Foundation & Deploy Pipeline | Pending |

**Coverage:**
- v1 requirements: 58 total
- Mapped to phases: 58/58
- Unmapped: 0

---
*Requirements defined: 2026-02-06*
*Last updated: 2026-02-06 after roadmap creation*
