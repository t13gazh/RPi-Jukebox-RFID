# Phoniebox Modernisierung

## Projekt-Kontext

Fork von [RPi-Jukebox-RFID](https://github.com/MiczFlor/RPi-Jukebox-RFID) (Phoniebox) v2.x — eine Raspberry-Pi-basierte Jukebox, die mit RFID-Karten gesteuert wird. Ziel: Open-Source Toniebox-Alternative mit modernem Web-Interface.

**Produktiv-Setup:** v2 läuft auf einem Raspberry Pi mit RFID-Reader, GPIO-Buttons, MPU6050 Gyro-Sensor und MPD Audio-Backend.

## Strategie

**v2 als Basis behalten, gezielt die kaputten Teile ersetzen.** Kein Neubau.

Was funktioniert (nicht anfassen): RFID-Reader, GPIO-Buttons, Gyro-Sensor (phonie-gyro), MPD, `playout_controls.sh`, Karten→Ordner-Zuordnung.

Was repariert/ersetzt wird:
1. **Modernes Web-Interface** (Prio 1) — v2 hat jQuery 1.12/Bootstrap 3/77 PHP-Dateien → Svelte 5 + FastAPI
2. **Gyro-Sensor Integration** (Prio 2) — phonie-gyro ins Web-UI integrieren
3. Karten-Management (Scan-and-Select Wizard)
4. Rollentrennung (Open/Parent-PIN/Expert-PIN)
5. Stream-URLs an Karten zuweisen (Internet-Radio, beliebige URLs via CARD-06)
6. PWA & Offline-Fähigkeit

## Wichtige Dateien

### Projektsteuerung (GSD-2)
- `.gsd/PROJECT.md` — Living Project Description
- `.gsd/REQUIREMENTS.md` — 58 v1 + 8 v2 Requirements mit IDs
- `.gsd/DECISIONS.md` — Entscheidungsregister (D001-D003: Git-Strategie, PR-Gruppierung)
- `.gsd/milestones/M001/M001-ROADMAP.md` — 11 Slices in 4 PR-Gruppen
- `.gsd/milestones/M001/M001-RESEARCH.md` — Stack, Architektur, Pitfalls (umfangreich)
- `.gsd/codebase/STRUCTURE.md` — **Aktuelles Codebase-Mapping** (aktualisiert 2026-03-16)

### Alte Analyse (Referenz)
- `.planning/ANALYSIS.md` — Vollständige v2/v3-Analyse und Strategieentscheidung
- `.planning/codebase/` — 7 Detailanalysen der v2-Codebase (Architektur, Stack, Concerns, Conventions, Integrations, Structure, Testing)
- `.planning/phases/01-*/01-CONTEXT.md` — Phase 1 Implementierungs-Entscheidungen (Deploy, Config, Struktur)

### v2 Codebase (bestehendes System)
- `htdocs/` — v2 Web-UI (PHP, 77 Dateien) — WIRD ERSETZT
- `scripts/playout_controls.sh` — Zentrale Steuerung (1.153 Zeilen Bash) — WIRD GEWRAPPED
- `scripts/daemon_rfid_reader.py` — RFID-Daemon — NICHT ANFASSEN
- `components/gpio_control/` — GPIO-Steuerung (Python) — NICHT ANFASSEN
- `settings/` — 50+ Config-Dateien — FORMAT BEIBEHALTEN, SQLite-Overlay

## Verwandte Repos

- **phonie-gyro**: https://github.com/t13gazh/phonie-gyro.git (lokal: `C:\Users\herrkraft\Repos\phonie-gyro`)
  - MPU6050 Gyro-Sensor Plugin, eigenständiger systemd-Service
  - Ruft `mpc` Befehle auf (entkoppelt von Phoniebox)
  - Version 1.1.0, funktionsfähig

## Git-Konfiguration

- **User:** t13gazh
- **Email:** 231762220+t13gazh@users.noreply.github.com
- **Origin:** https://github.com/t13gazh/RPi-Jukebox-RFID.git (unser Fork)
- **Upstream:** https://github.com/MiczFlor/RPi-Jukebox-RFID.git (Original)
- **Aktiver Branch:** `feature/webui-modernization`
- **develop** bleibt upstream-synchron für saubere PRs

## PR-Strategie (→ upstream)

| PR | Slices | Scope |
|----|--------|-------|
| PR 1 | S01+S02 | API Foundation & Real-Time Layer |
| PR 2 | S03+S04 | Player UI & Library Browser |
| PR 3 | S05-S08 | Card/Content Management & Settings |
| PR 4 | S09-S11 | Access Control, Gyro & PWA |

## Bekannte Probleme (v2)

- Security: Command Injection in `cardRegisterNew.php`, `shell=True` in RFID-Daemon
- Web-UI: jQuery 1.12.4 (2016), Bootstrap 3 (EOL 2019), eng gekoppelt an PHP-Backend
- Config: 50+ Einzeldateien in `settings/`, gehen bei `git pull` verloren
- Spotify: Pinned auf Mopidy-Spotify Alpha, Password-Auth von Spotify abgeschafft
- Hardware: Box hängt sich manchmal auf, Shutdown-Probleme

## Konventionen

- Analyse-Dokumentation auf Deutsch
- Code-Kommentare und Commits auf Englisch
- Commit Co-Author: `Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>`
