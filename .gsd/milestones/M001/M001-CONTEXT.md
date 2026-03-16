# M001 Context

## Overview

Replace the outdated PHP/jQuery/Bootstrap 3 web interface of Phoniebox v2.8.0 with a modern, mobile-first SPA. All existing backend systems (RFID daemon, GPIO buttons, phonie-gyro, MPD) remain untouched — the new frontend wraps them through a FastAPI gateway.

## Upstream Dependencies

None — this is the first milestone.

## Key Constraints

- **Platform**: Raspberry Pi (3B/4/5) with Raspberry Pi OS — must run on limited hardware
- **Backend**: Keep existing v2 backend (`playout_controls.sh`, RFID daemon, GPIO) — only replace frontend
- **Offline**: Core functionality (local music playback + control) must work without internet
- **Performance**: JS bundle under 150KB gzipped, initial load under 3 seconds on Pi
- **Language**: UI text in German, "Leichte Sprache" (plain language)
- **Security**: No `shell=True` in subprocess calls, no command injection

## Decided Stack

- **Frontend**: Svelte 5 + SvelteKit (adapter-static) + Tailwind CSS 4
- **Backend**: FastAPI + Uvicorn (Python 3.11+, wraps existing shell scripts)
- **Web Server**: Nginx (replaces Lighttpd for WebSocket proxy support)
- **Database**: SQLite (replaces 50+ flat config files)
- **Real-time**: WebSocket with MPD idle bridge (python-mpd2 asyncio)
- **Target OS**: Raspberry Pi OS Bookworm (Python 3.11)

## Stack Versions (verified 2026-03-16)

| Package | Version | Notes |
|---------|---------|-------|
| Svelte | 5.53.0 | Runes API stable |
| SvelteKit | 2.55.0 | `adapter-static`, project setup via `npx sv create` |
| Tailwind CSS | 4.2.1 | CSS-first config (no JS config file), Vite plugin |
| FastAPI | 0.135.1 | Requires Python ≥3.10, `fastapi[standard]` |
| python-mpd2 | 3.1.1 | Asyncio support |
| Uvicorn | latest | Included in `fastapi[standard]` |
