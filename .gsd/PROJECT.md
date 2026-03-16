# Phoniebox Web-Interface

## What This Is

A modern, mobile-first web interface for the Phoniebox (RPi-Jukebox-RFID v2.8.0) — replacing the outdated PHP/jQuery frontend while keeping the working backend (RFID, GPIO, Gyro, MPD) intact. The interface serves parents as a remote control and management tool for their children's music box, with three access tiers: open playback, PIN-protected parent settings, and expert configuration.

## Core Value

Parents can effortlessly manage their children's music box from any phone — no manual needed, no terminal required.

## Requirements

### Validated

- RFID card triggers instant music playback — existing v2 daemon
- GPIO buttons control player (play/pause, volume, skip) — existing v2 components
- Gyro sensor controls player (tilt gestures) — existing phonie-gyro plugin
- MPD plays local audio files reliably — existing v2 backend
- Card-to-folder assignment works — existing v2 shortcuts system
- Shell-based playout control (play, pause, volume, skip, shuffle, repeat) — existing playout_controls.sh
- Internet radio streams via MPD — existing capability
- yt-dlp audio download — existing (broken, needs repair)

### Active

- [ ] Modern responsive web player (play/pause/skip, volume, shuffle, repeat, now-playing status with cover art)
- [ ] Music library browser with switchable grid/list view (album covers as tiles)
- [ ] Three-tier access model: open player, PIN-protected parent area, separate expert area
- [ ] RFID card assignment via scan-and-select workflow (hold card to reader, pick content in browser)
- [ ] Max volume limit configurable by parents
- [ ] Sleep timer and scheduled stop
- [ ] Music upload through browser
- [ ] Folder and playlist creation in parent area
- [ ] Assign stream URLs to RFID cards (internet radio, arbitrary URLs)
- [ ] WiFi settings in expert area
- [ ] System info, restart, updates in expert area
- [ ] GPIO/hardware configuration in expert area
- [ ] MPD/audio configuration in expert area
- [ ] Export/import of settings and card configurations
- [ ] Gyro sensor configuration via web UI (sensitivity profiles, calibration)
- [ ] Fully offline-capable (local music always controllable without internet)
- [ ] "Leichte Sprache" — plain language UI, self-explanatory without documentation
- [ ] Replace old PHP interface completely

### Out of Scope

- Spotify integration — libspotify discontinued, industry-wide broken, no reliable OSS solution
- Mobile native app — web-first, responsive browser UI is sufficient
- Multi-user accounts — PIN tiers are enough for a family device
- Phoniebox v3 migration — evaluated and rejected (incomplete, declining activity)
- Mopidy replacement of MPD — unnecessary when MPD works fine
- Children as web UI users — kids use physical RFID cards and buttons

## Context

**Current state:** Phoniebox v2.8.0 runs on a Raspberry Pi with RC522 RFID reader, GPIO buttons, MPU6050 gyro sensor, and MPD audio backend. The hardware layer, RFID daemon, GPIO control, and audio playback all work. The web interface is the weak point: jQuery 1.12.4, Bootstrap 3, 77 PHP files with mixed HTML, no authentication, command injection vulnerabilities.

**Gyro plugin (phonie-gyro):** Standalone Python daemon as systemd service. Calls `mpc` commands directly (decoupled from Phoniebox). Version 1.1.0, working. Needs integration into web UI for configuration.

**Content sources the family uses:** Local audio files (Hoerspiele, music), internet radio, ARD/NDR Audiothek, free audiobook libraries (e.g. Librivox), podcasts. YouTube as download source.

**Known v2 issues to address:**
- Security: command injection in cardRegisterNew.php, shell=True in RFID daemon, chmod 777
- Config: 50+ individual files in settings/, lost on git pull
- No authentication on web interface
- YouTube downloader broken

## Constraints

- **Platform**: Raspberry Pi (3B/4/5) with Raspberry Pi OS — must run smoothly on limited hardware
- **Backend**: Keep existing v2 backend (playout_controls.sh, RFID daemon, GPIO) — only replace frontend
- **Offline**: Core functionality (local music playback + control) must work without internet
- **Performance**: Web UI must be fast on Pi — lightweight framework, minimal bundle size
- **Language**: UI text in German, "leichte Sprache" (plain language) for accessibility
- **Design**: Modern but warm — between playful and clinical, friendly colors, large touch targets for mobile

## Key Decisions

| Decision | Rationale | Outcome |
|----------|-----------|---------|
| Keep v2 backend, replace only frontend | RFID, GPIO, Gyro, MPD all work — no reason to rebuild | -- Pending |
| Three-tier access (open/PIN/expert) | Kids don't use web UI, parents need simple settings, experts need deep config | -- Pending |
| PIN protection (not password) | Quick to enter on phone, sufficient for family device on local network | -- Pending |
| Scan-and-select card assignment | Most intuitive: hold card, pick content — no manual ID entry | -- Pending |
| Complete replacement of old PHP UI | No parallel operation — clean cut avoids maintenance burden | -- Pending |
| Offline-first architecture | Music box must work without internet, streams are naturally online-only | -- Pending |

---
*Last updated: 2026-03-16 after fork setup and GSD-2 migration*
