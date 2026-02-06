# Phase 1: API Foundation & Deploy Pipeline - Context

**Gathered:** 2026-02-06
**Status:** Ready for planning

<domain>
## Phase Boundary

FastAPI gateway wrapping existing shell scripts (playout_controls.sh) safely, Nginx reverse proxy config, build-deploy pipeline from Windows dev machine to Pi, and SQLite config migration for settings consumed by the API. All existing backend systems (RFID daemon, GPIO, MPD) remain untouched.

</domain>

<decisions>
## Implementation Decisions

### Deploy-Workflow
- Ein-Befehl-Deployment: ein einzelner Befehl (z.B. `make deploy`) baut lokal und synct per rsync/scp zum Pi, startet Services neu
- Entwicklung nur lokal auf Windows, Testen auf dem Pi nach Deploy
- Pi-Zugang noch unklar — Deploy-Skript muss konfigurierbar sein (Hostname/IP als Variable)
- Deploy-Skript soll bei fehlender Pi-Verbindung sauber fehlschlagen mit hilfreicher Meldung

### Config-Migration
- Dual-Strategie: SQLite ist Quelle für die API, bei Änderung werden Einzeldateien zurückgeschrieben für Bash-Skripte
- Automatische Migration beim ersten API-Start: bestehende Dateien aus `settings/` einlesen und in SQLite importieren
- Bidirektionaler Sync: File-Watch erkennt manuelle Dateiänderungen (per SSH) und importiert sie in SQLite
- Erste Migration umfasst Player-Settings: Volume, Max-Volume, Startup-Volume, Idle-Shutdown — alles was Player-Endpoints brauchen
- Weitere Settings werden in späteren Phasen migriert wenn die jeweiligen Endpoints sie benötigen

### Projekt-Struktur
- Code-Layout: Claude wählt Best-Practice-Struktur — sauber, wiedererkennbar, lesbar
- Altes PHP-UI (htdocs/) bleibt parallel unter /legacy erreichbar, neues UI unter /
- Schrittweiser Übergang: erst wenn Feature-Parität erreicht ist, wird das alte UI abgeschaltet
- Python-Mindestversion: 3.9+ (kompatibel mit Raspberry Pi OS Bullseye)

### Claude's Discretion
- API-Endpoint-Stil und Fehler-Response-Format (nicht besprochen, Claude wählt Best Practice)
- Python-Dependency-Management (venv+requirements.txt vs Poetry/uv — Claude wählt passend für Pi)
- Genaues Verzeichnis-Layout (Top-Level-Ordner vs src/ — Claude wählt lesbare Konvention)
- Deploy-Skript: ob Frontend und Backend in einem oder getrennt deployt werden

</decisions>

<specifics>
## Specific Ideas

- Deploy-Skript muss auf Windows funktionieren (Entwickler-Maschine ist Windows)
- Pi-Verbindungsdaten sollen konfigurierbar sein (nicht hardcoded), da Zugang noch nicht eingerichtet
- File-Watch für bidirektionalen Config-Sync gewünscht — wer per SSH eine Datei ändert, soll die Änderung in der API sehen

</specifics>

<deferred>
## Deferred Ideas

None — discussion stayed within phase scope

</deferred>

---

*Phase: 01-api-foundation-deploy-pipeline*
*Context gathered: 2026-02-06*
