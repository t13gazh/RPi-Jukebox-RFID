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

## v3 (Future3) — Vorläufige Einschätzung

**Status:** Separater Branch (`future3/main`, `future3/develop`)

### Bekannt (aus README)
- Kompletter Rewrite in Python
- Plugin-System geplant
- Responsive Web-Client
- "Becoming a lot more stable" aber "not all features from v2.x ported"
- Sucht "adopters, testers and contributors"

### Bewertung
- **Positiv:** Klare architektonische Ziele, Python-only Stack
- **Bedenken:** Jahrelange Entwicklung, Features nicht komplett, niedrige Adoption
- **TODO:** v3-Branch separat evaluieren (Architektur, Code-Qualität, Vollständigkeit)

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

## Vergleichsmatrix

| Feature | Phoniebox v2 | Mopidy+Custom | Von Null | v3 beitragen |
|---------|:---:|:---:|:---:|:---:|
| RFID → Musik | ✓ | ✓ (custom) | ✓ (custom) | ✓ |
| Einfaches Web-UI | ✗ | ✓ | ✓ | ? |
| Spotify | ✗ kaputt | ✓ | ✓ | ? |
| Radio-Streams | eingeschränkt | ✓ | ✓ | ? |
| GPIO/Gyro | ✓ | ✓ (custom) | ✓ (custom) | ✓ |
| Einzeldatei→Karte | ✗ | ✓ | ✓ | ? |
| Rollen (Admin/User) | ✗ | ✓ | ✓ | ? |
| Stabilität | ✗ Bugs | ✓ ausgereift | ✗ neu | ? |
| Config-Persistenz | ✗ | ✓ | ✓ | ✓ |
| **Aufwand** | 4-6 Monate | **2-4 Wochen** | 3-6 Monate | 2-4 Monate |

---

## Empfehlung

**Primär: Mopidy + eigene RFID-Schicht** — beste Balance aus Aufwand und Ergebnis.

**Nächster Schritt:** v3-Branch evaluieren. Falls v3-Architektur solide ist, könnte das eine Alternative oder spätere Migration sein.

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
*Nächster Schritt: v3-Branch evaluieren*
