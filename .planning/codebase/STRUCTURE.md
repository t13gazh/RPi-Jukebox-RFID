# Codebase Structure

**Analysis Date:** 2026-02-06

## Directory Layout

```
rfid-jukebox/
├── components/                 # Hardware-specific drivers and integrations
│   ├── audio/                  # Audio HAT drivers (PirateAudioHAT, etc.)
│   ├── bluetooth-sink-switch/  # Bluetooth audio sink switching
│   ├── controls/               # Control interface integrations
│   ├── displays/               # Display drivers (LCD, dot-matrix)
│   ├── gpio_control/           # GPIO button/encoder/LED control system
│   ├── rfid-reader/            # RFID reader drivers (RC522, PN532)
│   ├── smart-home-automation/  # MQTT and home automation integration
│   └── synchronisation/        # Audio folder synchronization
├── docs/                       # Documentation files
├── htdocs/                     # Web application (PHP + JavaScript)
│   ├── _assets/                # CSS, fonts, jQuery libraries
│   ├── api/                    # REST API endpoints (player.php, playlist.php, etc.)
│   ├── js/                     # Frontend JavaScript (jukebox.js main)
│   ├── lang/                   # Language translation files
│   ├── utils/                  # PHP utility functions
│   ├── index.php               # Main web interface entry point
│   ├── func.php                # Shared PHP helper functions (600+ lines)
│   └── settings.php            # Settings management UI
├── logs/                       # Runtime logs (debug.log, application logs)
├── misc/                       # Miscellaneous assets and configs
│   └── sampleconfigs/          # Example configuration templates
├── scripts/                    # Core daemon and control scripts
│   ├── helperscripts/          # Utility scripts (file organization, analytics)
│   ├── installscripts/         # Installation and setup scripts
│   ├── userscripts/            # User-definable post-trigger scripts
│   ├── daemon_rfid_reader.py   # Main RFID polling daemon (entry point)
│   ├── playout_controls.sh     # Central control hub for all player operations (1153 lines)
│   ├── Reader.py               # RFID hardware interface abstraction
│   ├── activate_amplifier.py   # Amplifier/GPIO power management
│   └── inc.*.sh                # Configuration and utility includes
├── settings/                   # Configuration files and defaults
│   ├── global.conf             # Aggregated configuration (generated)
│   ├── rfid_trigger_play.conf  # RFID action mappings (from sample)
│   └── debugLogging.conf       # Debug logging settings (from sample)
├── shared/                     # Shared data directories
│   ├── audiofolders/           # User's music library (SMB/NFS accessible)
│   └── shortcuts/              # Card assignment shortcuts (metadata)
├── tests/                      # Test suite
│   └── htdocs/                 # PHP unit tests for API
├── .planning/                  # GSD planning documents
│   └── codebase/               # Architecture analysis documents
├── .github/                    # GitHub workflows and issue templates
├── composer.json               # PHP dependency manager (PHPUnit for testing)
├── package*.txt                # System package dependencies for Debian/Raspbian
├── requirements*.txt           # Python package dependencies
└── README.md                   # Main project documentation
```

## Directory Purposes

**components/**
- Purpose: Modular hardware drivers and integration plugins for RPi peripherals
- Contains: Python device drivers, shell setup scripts, README for each component
- Key files:
  - `gpio_control/gpio_control.py`: Main GPIO event processing engine
  - `rfid-reader/RC522/`, `PN532/`: Reader-specific implementations
  - `displays/HD44780-i2c/i2c_lcd_driver.py`: Display control
  - Each component self-contained with its own requirements.txt

**htdocs/**
- Purpose: Complete web application (UI + API backend)
- Contains: PHP templates, REST endpoints, JavaScript, CSS, third-party libraries
- Key files:
  - `index.php`: Entry point, loads navigation and player UI
  - `api/player.php`: Player control endpoint (PUT), status retrieval (GET)
  - `api/common.php`: Shared execution helpers for shell script dispatch
  - `func.php`: 600-line library of PHP utilities for UI generation and logic
  - `settings.php`: Web UI for configuration management

**scripts/**
- Purpose: Daemon processes, control logic, and system integration
- Contains: Python daemons, bash automation, configuration scripts
- Key files:
  - `daemon_rfid_reader.py`: Continuous RFID polling loop, main entry point
  - `playout_controls.sh`: Command dispatcher for all player/system operations
  - `inc.writeGlobalConfig.sh`: Configuration aggregation on startup
  - `activiate_amplifier.py`: Power management for audio amplifier
  - `Reader.py`: RFID hardware abstraction (uses evdev, device name lookup)

**settings/**
- Purpose: Configuration file storage and defaults
- Contains: INI-style configuration and sample files
- Key files (auto-generated from samples):
  - `global.conf`: Master configuration (created by inc.writeGlobalConfig.sh)
  - `rfid_trigger_play.conf`: RFID card → action mappings
  - `debugLogging.conf`: Per-component debug logging flags

**shared/**
- Purpose: User data directories accessible over network (SMB/NFS)
- Contains: Music files and card assignment metadata
- Key subdirectories:
  - `audiofolders/`: User's music library (can be single folder or organized by category)
  - `shortcuts/`: Card metadata files defining what each RFID card triggers

**tests/**
- Purpose: Automated test suite (PHP unit tests)
- Contains: PHPUnit test cases for API endpoints
- Key files:
  - `htdocs/api/PlayerTest.php`: Tests player.php API
  - `htdocs/api/PlayListTest.php`: Tests playlist.php API
  - `htdocs/TrackEditTest.php`: Tests track editing functionality

## Key File Locations

**Entry Points:**
- `htdocs/index.php`: Web UI main page (loads inc.header.php, renders navigation)
- `scripts/daemon_rfid_reader.py`: RFID daemon entry point (subprocess main loop)
- `components/gpio_control/gpio_control.py`: GPIO listener entry point (signal.pause())
- `scripts/installscripts/install-jukebox.sh`: Installation entry point (bash)

**Configuration:**
- `settings/global.conf`: Master config (sourced by shell scripts, parsed by PHP/Python)
- `settings/rfid_trigger_play.conf.sample`: Sample RFID mappings (copied to .conf on first run)
- `components/gpio_control/example_configs/`: GPIO device configuration examples (INI format)
- `.editorconfig`, `.flake8`: Code style configuration

**Core Logic:**
- `scripts/playout_controls.sh`: Player control dispatcher (1153 lines, handles all commands)
- `scripts/daemon_rfid_reader.py`: RFID event loop (reads cards, applies debouncing, triggers play)
- `htdocs/func.php`: Web UI utilities (600+ lines, forms, display helpers)
- `htdocs/api/common.php`: Shell script execution wrapper (exec, socket communication)

**Testing:**
- `tests/htdocs/api/PlayerTest.php`: API tests
- `components/gpio_control/test/`: GPIO unit tests (test_SimpleButton.py, test_RotaryEncoder.py)
- `composer.json`: PHPUnit configuration for `composer test`

## Naming Conventions

**Files:**

- **Shell scripts:** `lowercase-with-hyphens.sh` (e.g., `playout_controls.sh`, `idle-watchdog.sh`)
- **Python files:** `lowercase_with_underscores.py` (e.g., `daemon_rfid_reader.py`, `Reader.py`)
- **PHP files:** `camelCase.php` (e.g., `cardEdit.php`, `settingS.php`) or `lowercase.php` (e.g., `index.php`)
- **Include files:** `inc.descriptiveName.php` (e.g., `inc.header.php`, `inc.loadCover.php`)
- **API endpoints:** `{resource}.php` (e.g., `player.php`, `playlist.php`)
- **Config samples:** `filename.conf.sample` (e.g., `rfid_trigger_play.conf.sample`)
- **Config actual:** `filename.conf` (created from .sample on installation)

**Directories:**

- **Lowercase with hyphens:** Component directories (e.g., `gpio_control`, `rfid-reader`, `smart-home-automation`)
- **Lowercase:** System directories (e.g., `htdocs`, `scripts`, `shared`, `logs`)
- **Component pattern:** Each component has its own README.md, install.sh, requirements.txt

## Where to Add New Code

**New Feature (Audio/Control):**
- Primary code: Add control commands to `scripts/playout_controls.sh` (case statement in command dispatcher)
- Tests: Add test case to `tests/htdocs/api/PlayerTest.php` if API-exposed
- Config: Add setting to `settings/rfid_trigger_play.conf.sample` if user-configurable
- Documentation: Update component README.md or create new one in `docs/`

**New Component (Hardware Driver):**
- Implementation: Create `components/{component-name}/` with:
  - `{component-name}.py` or `.sh` (main driver)
  - `README.md` (usage documentation)
  - `requirements.txt` (Python dependencies)
  - `install.sh` (setup script)
  - `example_configs/` (sample configurations if applicable)
- Integration: Add import in `components/gpio_control/function_calls.py` if it needs to be callable
- Testing: Add `test/test_{component}.py` for unit tests

**New Web Page/Feature:**
- HTML/UI: Create `htdocs/{feature-name}.php` using inc.header.php wrapper
- Styling: Add CSS to `htdocs/_assets/css/` (Bootstrap 3 base)
- JavaScript: Add to `htdocs/js/jukebox.js` or create `htdocs/js/{feature}.js`
- API: If needed, add endpoint file `htdocs/api/{resource}.php` calling common.php helpers

**Utilities/Helpers:**
- Shared PHP: Add function to `htdocs/func.php` or `htdocs/utils/` file
- Shared bash: Create new `scripts/inc.{helper-name}.sh` and source it with `. inc.{name}.sh`
- Shared Python: Add to `components/{related-component}/` or create `scripts/lib/` module

**Tests:**
- PHP unit tests: `tests/htdocs/{feature}/Test.php` using PHPUnit
- Python unit tests: `components/{component}/test/test_{module}.py` using unittest
- Run with: `composer test` (PHP) or `python -m pytest` (Python, if configured)

## Special Directories

**settings/**
- Purpose: Persistent configuration store
- Generated: Yes - global.conf created by inc.writeGlobalConfig.sh
- Committed: No - only .sample files committed, actual .conf files generated per installation
- Ownership: pi:www-data (set during installation)
- Permissions: 775 (readable/writable by both daemon user and web server)

**logs/**
- Purpose: Runtime logs and debug output
- Generated: Yes - created on first run, appended to continuously
- Committed: No - .gitignore excludes logs/
- Pattern: debug.log is append-only with timestamps and component prefixes
- Retention: No auto-cleanup; manual log rotation recommended

**shared/audiofolders/ and shared/shortcuts/**
- Purpose: User data (music library and card metadata)
- Generated: No - manually populated by user via web UI or SMB/NFS
- Committed: No - contains user's personal data
- Accessibility: Shared via SMB/NFS to LAN for easy file management
- Ownership: www-data or pi user depending on web server configuration

**components/{component}/example_configs/**
- Purpose: Template configurations for hardware setup
- Generated: No - reference documentation
- Committed: Yes - shipped with codebase
- Pattern: Show INI format and required fields for GPIO device definitions
- Usage: Users copy to actual config location and customize

**tests/**
- Purpose: Automated test suite
- Generated: No - statically defined test cases
- Committed: Yes - part of CI/CD pipeline
- Run: Via PHPUnit (`composer test`) or GitHub Actions workflow
- Real-env tests: Excluded by default (marked with `@group real-env`)

---

*Structure analysis: 2026-02-06*
