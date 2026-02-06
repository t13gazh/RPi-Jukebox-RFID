# Analyse: RPi-Jukebox-RFID (Phoniebox)

**Erstellt:** 2026-02-06
**Zweck:** Entscheidungsgrundlage — Weiterentwickeln vs. Alternativen

---

## Ausgangslage

- Phoniebox v2.8.0 ist produktiv im Einsatz
- v3 (Future3) ist in Entwicklung, seit ~2 Jahren nicht fertiggestellt
- Ziel: Open-Source Toniebox-Alternative — Kinder spielen mit RFID-Karten Musik/Hörspiele, Eltern verwalten per Web-UI

## Anforderungen des Nutzers

### Must-Haves
- RFID-Karte auflegen → Musik spielt sofort
- Einfaches Web-UI für Eltern (nicht-technisch)
- Karten-Management: Einzeldateien UND Ordner zuweisbar
- Spotify-Integration (funktionierend)
- Streaming: ARD/NDR/ZDF Streams hinzufügen/herunterladen
- GPIO-Buttons für Player-Steuerung
- Gyro-Sensor für Player-Funktionen (bestehendes Plugin)
- Rollenbasierter Zugriff (User = Musik, Admin = Konfiguration)
- Stabiler Betrieb (kein Aufhängen, korrektes Herunterfahren)
- Installer mit Update/Reset ohne Konfig-Verlust

### Pain Points mit v2.8.0
- Web-Interface zu technisch, nicht endanwender-gerecht
- Karten-Management: nur Ordner, keine Einzeldateien
- Spotify kaputt
- YouTube-Downloader kaputt
- Streaming-Anbieter hinzufügen unkomfortabel
- Viele Einstellungen nur per Terminal erreichbar
- Jeder im Netzwerk hat vollen Zugriff (keine Rollentrennung)
- Hardware-spezifische Bugs (Aufhängen, Shutdown-Probleme)
- Installer verliert Konfiguration bei Updates

---

## v2.8.0 Tiefenanalyse

### Stärken
- Hardware-Abstraktion gut isoliert in `components/`
- Config-driven: RFID-Reader, GPIO-Devices über INI-Dateien hinzufügbar
- MPD als Audio-Backend = architektonisch korrekt
- Große Community, diverse Hardware-Setups dokumentiert
- Event-Flow: RFID-Daemon → Script → MPD funktioniert grundsätzlich

### Kritische Schwächen

#### Shell Script Hell
- `playout_controls.sh`: **1.153 Zeilen Bash** mit komplexen case-Statements
- Jede Logikänderung erfordert Modifikation dieses Monolithen
- Unmöglich zu testen

#### 3-Sprachen-Chaos
- Bash (Steuerung) + PHP (Web) + Python (Hardware) + JavaScript (Frontend)
- Kein einheitlicher Stack, hohe kognitive Last für Entwickler

#### Sicherheitslücken
- **Command Injection** in `cardRegisterNew.php` (Zeilen 118, 140, 145): User-Input direkt an `sed`
- **Unsafe Shell Execution** in `daemon_rfid_reader.py` (Zeile 99): `shell=True` mit ungeprüften Karten-IDs
- **Keine Authentifizierung** auf Web-Interface
- **chmod 777** wird großzügig verwendet
- **Path Traversal Risiken** in API `common.php`

#### Web-UI veraltet & eng gekoppelt
- jQuery 1.12.4 (2016), Bootstrap 3 (EOL 2019)
- 77 PHP-Dateien mit gemischtem HTML/PHP
- Direktes `exec("sudo ...")` aus PHP
- Frontend kann nicht ohne Backend-Rewrite ersetzt werden

#### Keine Tests
- **3 Testdateien** für gesamte Codebase
- 0% Coverage: Shell-Scripts, RFID-Daemon, Config, GPIO, Web-UI
- "real-env" Tests explizit ausgeschlossen

#### Config-Management fragil
- 50+ Einzeldateien in `settings/`
- Updates via `git pull` können Config überschreiben
- Kein Schema, kein Backup, kein Migrations-Mechanismus
- `inc.writeGlobalConfig.sh` (150+ Zeilen) liest/merged bei jedem Start

#### Spotify-Integration
- Pinned auf `Mopidy-Spotify==5.0.0a3` (Alpha-Release!)
- Spotify hat Password-Auth abgeschafft → OAuth nötig
- Wird bei jedem Spotify-API-Update brechen

### Fundamentale Architektur-Limits
1. File-basierte Config verhindert Echtzeit-Updates ohne Restart
2. Polling-basierter RFID-Daemon: 200ms Latenz durch `time.sleep(0.2)`
3. Keine Datenbank: Karten-Management skaliert nicht
4. MPD-Abhängigkeit begrenzt Audio-Features
5. Hardcoded Pfade: Raspberry Pi OS Lock-in

### Geschätzter Modernisierungsaufwand: 500-800 Stunden (4-6 Monate)

---

## v3 (Future3) — Tiefenanalyse (verifiziert am Code)

**Status:** Branch `future3/develop`, letzter Commit: Nov 2025
**Aktivität:** 94 Commits seit Jan 2024, nur 16 seit Jan 2025 — abnehmend

### Architektur

```
┌─────────────────────────────────────────────────────────────┐
│                    React SPA (MUI 5)                        │
│              react 17 + react-scripts + jszmq               │
│      Components: Cards, Library, Player, Settings           │
└──────────────┬────────────────────┬─────────────────────────┘
               │ REST API           │ ZMQ (via jszmq)
               ▼                    ▼
┌─────────────────────────────────────────────────────────────┐
│              Flask + Waitress (WSGI)                         │
│              Custom RPC System (jukebox.plugs)              │
│              ZeroMQ PubSub intern                           │
└──────────────┬──────────────────────────────────────────────┘
               │ Python-Module
┌──────────────▼──────────────────────────────────────────────┐
│                  Jukebox Components                          │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌───────────────┐  │
│  │playermpd │ │  rfid/   │ │  gpio/   │ │  cards/       │  │
│  │(MPD)     │ │ hardware │ │ controls │ │  (Card→URI)   │  │
│  └──────────┘ └──────────┘ └──────────┘ └───────────────┘  │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌───────────────┐  │
│  │ volume/  │ │ timers/  │ │ hostif/  │ │ mqtt/         │  │
│  └──────────┘ └──────────┘ └──────────┘ └───────────────┘  │
│  ┌──────────┐ ┌──────────┐ ┌────────────────────────────┐  │
│  │ jingle/  │ │ battery/ │ │ synchronisation/           │  │
│  └──────────┘ └──────────┘ └────────────────────────────┘  │
└──────────────┬──────────────────────────────────────────────┘
               ▼
┌─────────────────────────────────────────────────────────────┐
│              MPD (Music Player Daemon)                       │
│          ALSA → Audio Output                                │
└─────────────────────────────────────────────────────────────┘
```

### Verifizierte Fakten

**Backend (90 Python-Dateien):**
- Komplett Python — kein Bash für Steuerungslogik mehr
- ZeroMQ PubSub für interne Kommunikation (deutlich besser als v2)
- Custom RPC-Dekorator-System: Python-Methoden werden automatisch zu API-Endpoints
- YAML-basierte Konfiguration (statt 50+ Einzeldateien)
- Typ-Hints teilweise vorhanden, nicht durchgängig
- Python `logging` Modul korrekt eingesetzt

**Frontend (94 JS-Dateien):**
- React 17 (nicht 18!) + Material UI 5
- react-scripts (Create React App, nicht Vite — veraltet)
- i18next für Internationalisierung
- jszmq für ZeroMQ-Kommunikation im Browser
- UI-Bereiche: Cards, Library, Player, Settings, Navigation

**RFID-Reader (7 Implementierungen):**
- `rc522_spi` — SPI-basiert
- `generic_usb` — USB-HID
- `generic_nfcpy` — NFC über nfcpy
- `pn532_i2c_py532` — I2C
- `rdm6300_serial` — Seriell
- `fake_reader_gui` — Test/Entwicklung
- `template_new_reader` — Template für neue Reader

**Tests: 6 Dateien total**
- `test/cfghandler/test_cfghandler.py`
- `test/evdev/test_evdev_init.py`
- `test/gpioz/test_twinbutton.py`
- 3 GitHub Actions CI-Workflows (Docker-Build, Webapp-Build)
- Keine Integration-Tests, keine API-Tests, keine UI-Tests

### Feature-Vollständigkeit (v3 vs. Anforderungen)

| Anforderung | v3 Status | Details |
|-------------|-----------|---------|
| RFID → Musik | ✓ Vorhanden | 7 Reader-Implementierungen, gut abstrahiert |
| Einfaches Web-UI | ✗ Unvollständig | React-App existiert, aber nicht feature-complete |
| Karten-Management UI | ✗ Teilweise | Cards-Komponente existiert, Umfang unklar |
| Einzeldatei → Karte | ✗ Unklar | Card→URI Mapping unterstützt es theoretisch |
| Spotify | ✗ **Nicht vorhanden** | Code-Kommentar: "So far this is only for MPD (no spotify)" |
| Radio-Streams | ✓ Via MPD | MPD kann HTTP-Streams abspielen |
| GPIO-Buttons | ✓ Vorhanden | Saubere Python-Implementierung |
| Gyro-Sensor | ✗ Nicht vorhanden | Müsste als neues Component gebaut werden |
| Rollen (Admin/User) | ✗ **Nicht vorhanden** | Kein Auth, kein Login, keine Rollentrennung |
| Stabilität | ? Ungetestet | 6 Testdateien, keine Integration-Tests |
| Config-Persistenz | ✓ Verbessert | YAML, separiert von Code |
| Installer/Update | ? Teilweise | Install-Scripts vorhanden, kein Migrations-System |
| MQTT | ✓ Vorhanden | IoT/Smart-Home Integration |
| Battery Monitor | ✓ Vorhanden | Für mobile Boxen |

### Bewertung: 5.5/10

**Positiv:**
- Architektur ist ein massiver Sprung von v2 (Python-only, ZMQ, Components)
- RFID-Reader-Abstraktion hervorragend (7 Implementierungen + Template)
- YAML-Config statt Datei-Chaos
- Logging korrekt implementiert
- Component-System theoretisch erweiterbar

**Negativ:**
- **Spotify explizit nicht implementiert** — Quellcode-Kommentar bestätigt dies
- **Keine Authentifizierung** — Rollentrennung müsste komplett gebaut werden
- **React 17 + CRA** — Frontend-Stack bereits veraltet (CRA ist deprecated)
- **6 Testdateien** für 90 Python + 94 JS Dateien — nicht testbar
- **Custom RPC-System** erhöht Einstiegshürde massiv
- **Abnehmende Commit-Aktivität** — Projekt möglicherweise im Stillstand
- **Nie als Stable released** — nach Jahren kein v3.0.0
- **Dokumentation mangelhaft** — kein OpenAPI, wenig Dev-Docs

### Spotify-Problem (branchenübergreifend)

Spotify hat `libspotify` eingestellt. Betrifft ALLE Open-Source-Player:
- `mopidy-spotify` nutzt fragile inoffizielle API
- `spotifyd`/`librespot` = Spotify Connect Receiver (kein programmatischer Zugriff)
- Spotify ToS verbieten technisch einige Use Cases
- **Kein Open-Source-Projekt löst Spotify zuverlässig**
- Realistisch: Spotify Connect als Receiver oder Premium-Features akzeptieren

---

## Alternativen-Recherche

### Option 1: Phoniebox v2 weiterentwickeln
- **Aufwand:** 4-6 Monate (500-800h)
- **Risiko:** Hoch — kämpft gegen technische Schulden
- **Ergebnis:** Funktional aber nie "sauber"
- **Urteil:** Nicht empfohlen

### Option 2: Phoniebox v3 evaluieren & beitragen
- **Aufwand:** 2-4 Monate (300-500h)
- **Risiko:** Mittel — abhängig von v3-Qualität und Community
- **Ergebnis:** Moderner Stack, Community-Support
- **Urteil:** Erst evaluieren, dann entscheiden

### Option 3: Mopidy + eigene RFID-Schicht (EMPFOHLEN)
- **Aufwand:** 2-4 Wochen für Kern, dann iterativ
- **Risiko:** Niedrig — Mopidy ist ausgereift und aktiv
- **Ergebnis:** Stabil, modern, alle Pain Points gelöst

**Architektur:**
```
Kinder:  RFID-Karte → sofort Musik
Eltern:  Web-UI → Karten verwalten, Streaming, Einstellungen

┌─────────────────────────────────┐
│  Eigenes Web-UI (React/Svelte) │  ← Einfach, rollenbasiert
│  Admin vs. User-Ansicht        │
└──────────┬──────────────────────┘
           │ REST API / WebSocket
┌──────────▼──────────────────────┐
│  Mopidy (Audio-Server)          │  ← Spotify, Radio, Lokal
│  + Iris Web-UI (für Admin)      │
└──────────┬──────────────────────┘
     ┌─────┴─────┬──────────┐
  ┌──▼──┐  ┌─────▼────┐ ┌───▼───┐
  │RFID │  │GPIO/Gyro │ │Dateien│
  │RC522│  │Buttons   │ │Streams│
  └─────┘  └──────────┘ └───────┘
```

**Vorteile:**
- Spotify über `mopidy-spotify`
- Radio (ARD/NDR/ZDF) über `mopidy-tunein`
- YouTube über `mopidy-youtube`
- ~800-1100 Zeilen eigener Code statt 1.153 Zeilen Bash
- Stabile Audio-Engine
- Config geht nie verloren

**Mopidy-Plugins:**
- `mopidy-spotify` — Spotify Premium
- `mopidy-tunein` — Internet Radio
- `mopidy-youtube` — YouTube Audio
- `mopidy-soundcloud` — SoundCloud
- `mopidy-iris` — Modernes React-basiertes Web-UI
- `mopidy-local` — Lokale Dateien
- `mopidy-mpd` — MPD-Protokoll-Kompatibilität

### Option 4: Komplett von Null bauen
- **Aufwand:** 3-6 Monate (1000-1500h)
- **Risiko:** Mittel — alles selbst maintainen
- **Ergebnis:** Maximale Kontrolle
- **Stack:** FastAPI + React/Svelte + MPD/VLC + SQLite
- **Urteil:** Nur für Lernprojekt oder einzigartige Anforderungen

### Weitere evaluierte Plattformen

| Plattform | RFID | Web-UI | Spotify | GPIO | Bewertung |
|-----------|------|--------|---------|------|-----------|
| **Volumio** | Plugin (unsicher) | Exzellent | Premium ($$) | Plugins | Viable, aber Kosten |
| **moOde Audio** | Nein | Gut | Via Spotifyd | Ja | Viable, aber PHP-Stack |
| **TonUINO** | Ja | Nein | Nein | Arduino | Falsche Plattform (Arduino) |
| **Home Assistant** | Via Automation | Komplex | Ja | Ja | Overkill |
| **Jellyfin** | Nein | Ja | Nein | Nein | Overkill für Audio |

---

## Vergleichsmatrix (aktualisiert nach v3-Evaluation)

| Feature | Phoniebox v2 | Phoniebox v3 | Mopidy+Custom | Von Null |
|---------|:---:|:---:|:---:|:---:|
| RFID → Musik | ✓ | ✓ | ✓ (custom) | ✓ (custom) |
| Einfaches Web-UI | ✗ | ✗ unvollständig | ✓ | ✓ |
| Spotify | ✗ kaputt | ✗ nicht impl. | ~fragil | ~fragil |
| Radio-Streams | eingeschränkt | ✓ via MPD | ✓ | ✓ |
| GPIO/Gyro | ✓/✗ | ✓/✗ | ✓ (custom) | ✓ (custom) |
| Einzeldatei→Karte | ✗ | ~möglich | ✓ | ✓ |
| Rollen (Admin/User) | ✗ | ✗ | ✓ | ✓ |
| Stabilität | ✗ Bugs | ? ungetestet | ✓ ausgereift | ✗ neu |
| Config-Persistenz | ✗ | ✓ | ✓ | ✓ |
| Test-Coverage | 3 Tests | 6 Tests | ✓ Mopidy-Core | ✓ eigene |
| Community-Aktivität | Maintenance | Abnehmend | Aktiv | N/A |
| **Aufwand** | 4-6 Monate | 2-4 Monate | **2-4 Wochen** | 3-6 Monate |

Legende: ✓ = vorhanden, ✗ = fehlt, ~ = fragil/eingeschränkt, ? = unklar

---

## Empfehlung (nach v3-Evaluation)

### Ergebnis: v3 ändert die Empfehlung NICHT

v3 hat eine bessere Architektur als v2 (Python-only, ZMQ, Components), aber:
- **Spotify fehlt komplett** (im Quellcode bestätigt)
- **Keine Authentifizierung/Rollentrennung**
- **Web-UI unvollständig**
- **Abnehmende Community-Aktivität** (16 Commits in 11 Monaten)
- **Nie als stable released**
- **6 Tests für 184 Dateien**

**Primäre Empfehlung bleibt: Mopidy + eigene RFID-Schicht**

### Hybrid-Option (pragmatisch)

Die beste Elemente aus v3 übernehmen, aber eigene Basis bauen:
- **Übernehmen aus v3:** RFID-Reader-Implementierungen (exzellent, 7 Varianten)
- **Eigene Basis:** FastAPI + Mopidy + React/Svelte
- **Ergebnis:** Schnellster Weg zu einem vollständigen, wartbaren System

### Entscheidungsmatrix nach Priorität

| Wenn dir wichtig ist... | Dann wähle... |
|-------------------------|---------------|
| Schnellstes Ergebnis | Mopidy + Custom (2-4 Wochen) |
| Community & langfristiger Support | Mopidy + Custom (aktive Community) |
| Maximale Kontrolle | Von Null (3-6 Monate) |
| Minimaler eigener Code | Phoniebox v3 Fork (aber Lücken bleiben) |
| Spotify-Integration | Keiner löst es sauber — Spotify Connect als Workaround |

### Spotify-Realität

Spotify ist in der Open-Source-Welt ein ungelöstes Problem:
- `mopidy-spotify`: Fragil, inoffizielle API, bricht regelmäßig
- `spotifyd`/`librespot`: Spotify Connect Receiver, kein programmatischer Zugriff
- **Pragmatische Lösung:** Spotify Connect Receiver + RFID triggert "Play on this device"
- Alternative: `yt-dlp` zum Herunterladen von Playlists als lokale Dateien

---

## Toniebox als UX-Vorbild

Lektionen für das eigene Projekt:
1. **Trennung ist kritisch:** Kind-Interface ≠ Admin-Interface
2. **Physisch ist intuitiv:** Karten > Menüs für Kinder
3. **Position merken:** Wo wurde gestoppt bei jedem RFID-Tag
4. **Einfach = zuverlässig:** Weniger Features = weniger Bugs
5. **Offline-first:** Muss ohne Internet funktionieren

---
*Erstellt: 2026-02-06*
*v3-Evaluation abgeschlossen: 2026-02-06*
*Status: Entscheidung offen — alle Optionen dokumentiert*
