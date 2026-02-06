# External Integrations

**Analysis Date:** 2026-02-06

## APIs & External Services

**Audio Playback & Streaming:**
- MPD (Music Player Daemon) - Local socket-based IPC on port 6600
  - SDK/Client: Native socket communication via PHP in `htdocs/api/common.php`
  - Purpose: Controls local music playback, volume, queue management
  - Implementation: Direct socket connection to localhost:6600 using TCP/SOCK_STREAM

- Mopidy - HTTP API on port 6680 (optional Spotify alternative)
  - SDK/Client: curl-based HTTP calls to `http://localhost:6680/mopidy/rpc`
  - Method: JSON-RPC over HTTP POST
  - Purpose: Audio engine with Spotify/streaming support (used instead of MPD when Spotify is enabled)
  - Web UI: Iris client accessible at `http://[local_url]:6680/iris`

**Content Download & Streaming:**
- YouTube via yt-dlp
  - Package: `yt-dlp` (from `requirements.txt`)
  - Purpose: Download audio from YouTube directly to local storage
  - Files: `scripts/` directory contains audio processing logic

- Spotify Integration (optional)
  - Package: `Mopidy-Spotify==5.0.0a3` (in `requirements-spotify.txt`)
  - Purpose: Stream Spotify content directly through Phoniebox
  - Implementation: Via Mopidy middleware
  - Access: Web URL conversion from `https://open.spotify.com/` to `spotify:` format

**Web Streaming:**
- Podcast feeds via HTTP/HTTPS URLs
  - Supported: RSS/Atom feeds for podcasts
  - Format: Plain text `.txt` files containing feed URLs
  - Example locations: `podcasts.txt`, `livestream.txt`, `spotify.txt` in audio folders

- Live radio streams via HTTP/HTTPS
  - Format: URLs pointing to stream endpoints (M3U, direct MP3, etc.)
  - Stored in: `livestream.txt` files in audio directory structure

## Data Storage

**Databases:**
- None - This is a file-based system with no traditional database

**File Storage:**
- Local filesystem only
- Configuration: Base path set in `htdocs/config.php` (typically `/home/pi/RPi-Jukebox-RFID`)
- Audio content stored in nested folder structure under `Audio_Folders_Path`
- Metadata files: `.txt` files indicating content type (spotify.txt, podcast.txt, livestream.txt, youtube.txt)

**RFID Card Database:**
- File-based mapping in shell scripts
- Card data stored in configuration files managed by bash scripts
- No SQL database - purely file-based configuration

**Caching:**
- None detected - Direct file system access for audio metadata

## Authentication & Identity

**Auth Provider:**
- None - No built-in authentication
- System relies on local network access (WiFi or wired)
- Optional WiFi hotspot mode for standalone operation
- RFID cards serve as user identification/control method, not authentication

**Authorization:**
- Implicit through RFID card associations
- GPIO buttons allow direct hardware control without authentication

## Monitoring & Observability

**Error Tracking:**
- None detected - No external error tracking service integrated

**Logs:**
- Local logging to `logs/debug.log`
- Debug logging enabled via config: `DEBUG_WebApp_API` setting
- File-based logging in PHP: `file_put_contents("../../logs/debug.log", ...)`

**Health Status:**
- MPD status monitoring via systemctl
- Mopidy status monitoring via systemctl
- Service status checks in AJAX endpoints (`ajax.loadMPDStatus.php`, `ajax.loadMopidyStatus.php`)

## CI/CD & Deployment

**Hosting:**
- Raspberry Pi (bare metal) or compatible Linux device
- No cloud deployment - runs locally

**CI Pipeline:**
- GitHub Actions workflows (in `.github/workflows/`)
- Available workflows:
  - `pythonpackage.yml` - Python code checks and tests
  - `php.yml` - PHP unit tests with PHPUnit
  - `test_docker_debian.yml` - Installation script testing on Debian
  - `test_docker_debian_codename_sub.yml` - Multi-version Debian testing
  - `markdown.yml` - Documentation linting
  - `codeql-analysis.yml` - CodeQL security analysis
  - `release.yml` - Release automation

**Deployment:**
- One-line install script for automated setup
- Installation scripts: `scripts/installscripts/install-jukebox.sh`
- Package installation via apt-get using `packages.txt` manifests
- Configuration via interactive bash scripts during installation

## Environment Configuration

**Required env vars:**
- `DOCKER_RUNNING` (optional) - Set to "true" when running in Docker (in `ci/Dockerfile.debian`)
- `CI_RUNNING` (optional) - Set to "true" in CI environments (in `ci/Dockerfile.debian`)

**Key Configuration Files:**
- `config.php.sample` - Web application base configuration template
  - `base_url` - URL path to application
  - `base_path` - Absolute filesystem path to installation
  - `local_url` - Local network hostname or IP

**System Configuration:**
- Lighttpd web server config (in `/etc/lighttpd/`)
- PHP-CGI config (in `/etc/lighttpd/conf-available/15-fastcgi-php.conf`)
- MPD config (in `/etc/mpd.conf` or Mopidy equivalent)

**Service Configuration:**
- systemd unit files for daemon services:
  - `phoniebox-mqtt-client.service-default.sample` (MQTT daemon)
  - `phoniebox-buttons-usb-encoder.service.sample` (USB encoder support)
  - `i2c-lcd.service.default.sample` (Display support)

**Secrets location:**
- MQTT certificates (optional): `/home/pi/MQTT/mqtt-*.crt` and `.key` files
- No application-level secrets or API keys required for local operation

## Webhooks & Callbacks

**Incoming:**
- None detected

**Outgoing:**
- None detected - System is event-driven via RFID/GPIO, not webhook-based

## GPIO & Hardware Integration

**GPIO Control:**
- RPi GPIO library via `rpi-lgpio` wrapper
- Package: `rpi-lgpio` with `spi-py` support (from `requirements.txt`)
- Button interfaces via USB encoders and direct GPIO pins
- SPI communication for RFID readers (RC522 via SPI)

## MQTT Smart Home Automation

**MQTT Client:**
- Daemon: `components/smart-home-automation/MQTT-protocol/daemon_mqtt_client.py`
- Library: `paho-mqtt` (from MQTT requirements.txt)

**Configuration:**
- MQTT broker hostname and port (default: openHAB on port 8883)
- Base topic: "phoniebox"
- Client ID: "phoniebox"
- Authentication: Supports username/password or certificate-based TLS
- Status update intervals:
  - While playing: 5 seconds
  - While idle: 30 seconds

**MQTT Topics:**
- Base topic: `phoniebox`
- Publishes: Player status, current track, volume, pause state
- Subscribes: Command topics for remote control from home automation systems

## Display & UI Integrations

**Web Interface:**
- Accessible at `http://[local_ip]/` (via Lighttpd)
- AJAX-based responsive interface
- File upload support via jQuery File Upload plugin
- Touch-friendly interface for LCD displays

**Hardware Displays:**
- Optional LCD support (HD44780 via I2C)
- Pirate Audio HAT support for integrated display and buttons
- USB encoder button support

## Serial & PCSC Support

**Serial Communication:**
- Package: `pyserial` (from `requirements.txt`)
- Purpose: Serial RFID readers and other devices

**PC/SC Smart Card Support:**
- RFID reader variant: `Reader.py.pcsc` for generic PC/SC readers
- Purpose: Support for contactless card readers via standard PC/SC interface

---

*Integration audit: 2026-02-06*
