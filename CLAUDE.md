# Phoniebox Modernisierung

## Projekt-Kontext

Dieses Repo ist ein Clone von [RPi-Jukebox-RFID](https://github.com/MiczFlor/RPi-Jukebox-RFID) (Phoniebox) v2.8.0 — eine Raspberry-Pi-basierte Jukebox, die mit RFID-Karten gesteuert wird. Ziel: Open-Source Toniebox-Alternative.

**Produktiv-Setup:** v2.8.0 läuft auf einem Raspberry Pi mit RFID-Reader, GPIO-Buttons, MPU6050 Gyro-Sensor und MPD Audio-Backend.

## Strategie

**v2 als Basis behalten, gezielt die kaputten Teile ersetzen.** Kein Neubau.

Was funktioniert (nicht anfassen): RFID-Reader, GPIO-Buttons, Gyro-Sensor (phonie-gyro), MPD, Karten→Ordner-Zuordnung.

Was repariert/ersetzt wird:
1. **Modernes Web-Interface** (Prio 1) — v2 hat jQuery 1.12/Bootstrap 3/77 PHP-Dateien
2. **Gyro-Sensor Integration** (Prio 2) — phonie-gyro ins Web-UI integrieren
3. Karten-Management (Einzeldateien, Streams)
4. Rollentrennung (User vs Admin)
5. Spotify & Streaming
6. Stabilität & Installer

## Wichtige Dateien

- `.planning/ANALYSIS.md` — Vollständige Analyse (v2, v3, Alternativen, Strategie, Phasenplan)
- `.planning/codebase/` — 7 Dokumente zur Codebase-Analyse (Architektur, Stack, Concerns etc.)
- `htdocs/` — v2 Web-UI (PHP, 77 Dateien)
- `scripts/playout_controls.sh` — Zentrale Steuerung (1.153 Zeilen Bash)
- `scripts/daemon_rfid_reader.py` — RFID-Daemon
- `components/gpio_control/` — GPIO-Steuerung (Python)

## Verwandte Repos

- **phonie-gyro**: https://github.com/t13gazh/phonie-gyro.git (lokal: `C:\Users\herrkraft\Repos\phonie-gyro`)
  - MPU6050 Gyro-Sensor Plugin, eigenständiger systemd-Service
  - Ruft `mpc` Befehle auf (entkoppelt von Phoniebox)
  - Version 1.1.0, funktionsfähig

## Git-Konfiguration

- **User:** t13gazh
- **Email:** 231762220+t13gazh@users.noreply.github.com
- Remote origin: https://github.com/MiczFlor/RPi-Jukebox-RFID.git

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
