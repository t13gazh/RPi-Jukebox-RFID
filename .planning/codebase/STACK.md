# Technology Stack

**Analysis Date:** 2026-02-06

## Languages

**Primary:**
- PHP 7+ - Web application, API endpoints, and UI logic
- Python 3 - RFID reading, daemon scripts, GPIO control, audio processing, MQTT integration
- Bash/Shell - System integration, installation scripts, audio control wrapper scripts
- JavaScript - Frontend interactivity via jQuery and Bootstrap

**Secondary:**
- HTML5 - Web interface structure
- CSS3 - Web interface styling

## Runtime

**Environment:**
- Raspberry Pi OS (Debian-based Linux)
- Supports Bullseye and later Debian codenames
- Also compatible with standard Debian distributions

**Package Manager:**
- PHP: Composer (see `composer.json`)
- Python: pip (with version-specific requirements files)
- System: apt-get for Debian packages

## Frameworks

**Core:**
- Lighttpd - Web server (from `packages.txt`)
- MPD (Music Player Daemon) - Audio playback engine on port 6600
- Mopidy - Alternative audio engine with Spotify support (optional)
- Bootstrap 3 - Frontend CSS framework (in `htdocs/_assets/bootstrap-3/`)
- jQuery 1.12.4 - JavaScript library (in `htdocs/_assets/`)

**Audio Processing:**
- ffmpeg - Audio file conversion and manipulation (from `packages.txt`)
- yt-dlp - YouTube audio downloading
- alsa-utils - ALSA audio utilities (from `packages.txt`)
- mpg123 - Audio playback (from `packages.txt`)

**Testing:**
- PHPUnit 11 - PHP unit testing (in `composer.json`)
- pytest - Python testing framework (in `requirements.txt`)
- pytest-pythonpath - Pytest plugin for code structure (in `requirements.txt`)
- pytest-cov - Code coverage for pytest (in `requirements.txt`)

**Build/Dev:**
- Docker (Dockerfile.debian for CI/testing)
- GitHub Actions (CI/CD pipelines)

## Key Dependencies

**Critical:**
- `python-mpd2` - Python client library for MPD communication (`components/gpio_control/requirements.txt`, `requirements-GPIO.txt`)
- `evdev` - Python library for input device handling (from `requirements.txt`)
- `rpi-lgpio` - Raspberry Pi GPIO library, replaces RPi.GPIO for modern kernels (from `requirements.txt`)
- `spi-py` - GitHub SPI Python wrapper (from `requirements.txt`)
- `pyserial` - Serial port communication (from `requirements.txt`)

**RFID Readers:**
- `pi-rc522==2.3.0` - RC522 RFID reader library (`components/rfid-reader/RC522/requirements.txt`)
- `spidev` - SPI device communication for RC522 (`components/rfid-reader/RC522/requirements.txt`)
- `py532lib` - PN532 RFID reader library (`components/rfid-reader/PN532/requirements.txt`)

**Spotify/Audio Streaming:**
- `Mopidy-Local` - Local music playback for Mopidy (in `requirements-spotify.txt`)
- `Mopidy-MPD` - MPD protocol support in Mopidy (in `requirements-spotify.txt`)
- `Mopidy-Iris==3.69.3` - Web UI for Mopidy (in `requirements-spotify.txt`)
- `Mopidy-Spotify==5.0.0a3` - Spotify integration (in `requirements-spotify.txt`)

**Smart Home:**
- `paho-mqtt` - MQTT client for home automation (`components/smart-home-automation/MQTT-protocol/requirements.txt`)
- `inotify` - File system event monitoring (for MQTT daemon) (`components/smart-home-automation/MQTT-protocol/requirements.txt`)

**Testing & Code Quality:**
- `coverage` - Code coverage measurement (from `requirements.txt`)
- `php-mock/php-mock-phpunit==2` - PHP mocking for tests (in `composer.json`)

## Configuration

**Environment:**
- Configured via installation scripts and configuration files
- Web configuration: `htdocs/config.php.sample` (sample provided)
- PHP configuration includes base URL and file paths
- System-level configuration through bash shell scripts

**Build:**
- `.editorconfig` - Editor configuration for consistent code style
- `.flake8` - Python linting configuration for PEP8 compliance
- `composer.json` - PHP dependency management
- `requirements*.txt` - Multiple Python requirements files for different components:
  - `requirements.txt` - Main Python dependencies
  - `requirements-GPIO.txt` - GPIO-specific dependencies
  - `requirements-spotify.txt` - Spotify/Mopidy dependencies
  - `requirements-excluded.txt` - Excluded/deprecated packages

**Package Requirements:**
- `packages.txt` - Base Debian system packages
- `packages-raspberrypi.txt` - Raspberry Pi-specific packages
- `packages-spotify.txt` - Spotify-related system packages
- `packages-autohotspot_dhcpcd.txt` - Auto-hotspot networking (dhcpcd variant)
- `packages-autohotspot_NetworkManager.txt` - Auto-hotspot networking (NetworkManager variant)

## Platform Requirements

**Development:**
- Python 3.6+
- PHP 7.0+
- Debian-based Linux system (Raspberry Pi OS or standard Debian)
- Git for version control
- Docker (optional, for containerized testing)

**Production:**
- Raspberry Pi (4, 3B, or compatible ARM Linux device)
- Raspberry Pi OS or compatible Debian distribution
- Lighttpd web server
- MPD or Mopidy audio engine
- ALSA audio system
- 1GB+ RAM recommended
- SD card 4GB+ (recommended 16GB+)

**Optional Hardware:**
- RFID readers (USB, RC522, PN532, or PC/SC compatible)
- GPIO buttons for hardware control
- Display modules (HD44780, Pirate Audio HAT)
- Bluetooth audio sink
- Additional audio output devices

---

*Stack analysis: 2026-02-06*
