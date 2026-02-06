# Architecture

**Analysis Date:** 2026-02-06

## Pattern Overview

**Overall:** Event-driven multi-layer architecture with separation between hardware interface, backend control, and web frontend.

**Key Characteristics:**
- Event-based triggering: RFID cards trigger playback via daemon process
- Microservice-style shell scripts: Each player control function is a modular script
- Configuration-driven behavior: Settings stored as individual files merged into global config
- Web API abstraction: HTTP endpoints proxy commands to shell scripts via PHP
- Hardware abstraction: GPIO and RFID readers isolated in separate device layers
- Asynchronous processing: Shell scripts handle long-running operations without blocking web UI

## Layers

**Hardware Abstraction Layer:**
- Purpose: Interface with physical RFID readers, GPIO buttons, displays, and audio hardware
- Location: `components/` (gpio_control, rfid-reader, displays, audio)
- Contains: Python and shell device drivers, hardware-specific configuration
- Depends on: RPi GPIO library, evdev, hardware-specific SDKs
- Used by: Daemon scripts and GPIO control system

**Daemon/Event Processor Layer:**
- Purpose: Read RFID cards and trigger playback by executing control scripts
- Location: `scripts/daemon_rfid_reader.py`, `scripts/activate_amplifier.py`
- Contains: Continuous event listeners that poll hardware and dispatch actions
- Depends on: Reader.py (RFID interface), subprocess calls to playout_controls.sh
- Used by: System startup (runs as service)

**Control/Command Layer:**
- Purpose: Execute all player control operations (play, pause, volume, etc.)
- Location: `scripts/playout_controls.sh` (1153 lines - the core control hub)
- Contains: Command parser and executors for MPD/Mopidy, file operations, system controls
- Depends on: MPD socket commands, amixer for volume, shell utilities
- Used by: Web API, RFID daemon, GPIO controls

**Configuration Management Layer:**
- Purpose: Aggregate and persist all system settings
- Location: `settings/` directory (individual .conf files), `scripts/inc.writeGlobalConfig.sh`
- Contains: INI-style config files, script that merges them into global.conf
- Depends on: File I/O, shell configuration scripts
- Used by: All layers for reading behavior settings

**Web API Layer:**
- Purpose: HTTP endpoints that receive web app requests and execute commands
- Location: `htdocs/api/` (player.php, playlist.php, volume.php, cover.php)
- Contains: REST endpoints that parse JSON requests and dispatch to playout_controls.sh
- Depends on: PHP 7.x, MPD socket access, shell script execution
- Used by: Web frontend (jukebox.js)

**Web UI Layer:**
- Purpose: User interface for controlling Phoniebox, managing files, registering cards
- Location: `htdocs/` PHP pages and JavaScript
- Contains: Bootstrap 3 UI templates, jQuery AJAX handlers, file management forms
- Depends on: jQuery 1.12.4, PHP templates (func.php provides utilities)
- Used by: End users via browser

## Data Flow

**RFID Card Swipe Flow:**

1. RFID daemon (`daemon_rfid_reader.py`) polls Reader.py for card IDs
2. Card ID matched against `global.conf` for assigned playlist/function
3. daemon_rfid_reader.py executes shell command: `playout_controls.sh --cardid={id}`
4. playout_controls.sh sources configuration, looks up assigned action in shortcuts
5. playout_controls.sh executes play command via MPD socket or executes control function
6. Music playback starts (or function executes: volume change, shutdown, etc.)

**Web App Control Flow:**

1. User clicks button in web UI (jukebox.js)
2. AJAX PUT request sent to `htdocs/api/player.php` with command (e.g., `{"command":"play"}`)
3. player.php calls common.php helper `execScript("playout_controls.sh -c=playerplay")`
4. common.php executes script via `exec("sudo {script}")` with response capture
5. playout_controls.sh executes MPD command via socket or amixer command
6. Response sent back to jukebox.js as JSON
7. UI updates with new player state

**Configuration Load Flow (on startup):**

1. System boots or daemon starts
2. `inc.writeGlobalConfig.sh` is called (manually or via startup)
3. Script reads sample config: `settings/rfid_trigger_play.conf.sample`
4. For each config option, checks if individual settings file exists
5. If file missing, creates with default value from sample
6. Reads all individual settings and merges into `settings/global.conf`
7. All processes source global.conf for current configuration

**Player State Retrieval:**

1. Web UI or external system requests current status
2. API endpoint (player.php) calls `execMPDCommand("status\ncurrentsong\nclose")`
3. Direct socket connection to MPD at localhost:6600
4. MPD returns key-value pairs of current state
5. Response formatted and returned as JSON
6. UI updates with player position, volume, current track, etc.

**State Management:**
- Persistent: Configuration stored in individual files in `settings/`
- Transient: Player state managed by MPD daemon (external process), queried on demand
- Event-driven: RFID cards and GPIO buttons trigger immediate state changes via shell scripts
- No local application state: Phoniebox is stateless - config files + MPD daemon = current state

## Key Abstractions

**Card Assignment:**
- Purpose: Map RFID card ID to audio content or control function
- Examples: `shared/shortcuts/`, entries in `global.conf` with `CMD{ID}=action`
- Pattern: Card ID → lookup in global.conf → maps to folder path, playlist, or function name

**Settings File Pairs:**
- Purpose: Provide defaults while allowing user customization
- Examples: `rfid_trigger_play.conf.sample` paired with `rfid_trigger_play.conf`
- Pattern: Sample files provide structure; installation copies to non-.sample; scripts read non-sample

**Function Calls (GPIO):**
- Purpose: Abstract physical button/encoder actions to named functions
- Examples: `components/gpio_control/function_calls.py` defines callable operations
- Pattern: GPIO event → mapped to function name → dispatch via phoniebox_function_calls class

**Device Configuration:**
- Purpose: Allow flexible hardware attachment without code changes
- Examples: `components/gpio_control/example_configs/` show INI-style device definitions
- Pattern: Config specifies pin, type (Button/LED/Rotary), and function callbacks

**Shell Script Modularity:**
- Purpose: Keep control logic independent of invocation method
- Examples: `playout_controls.sh` handles commands whether called from daemon, web, or GPIO
- Pattern: Single entry point script with `-c=command` parameter, command lookup, execution

## Entry Points

**RFID Trigger (Physical Cards):**
- Location: `scripts/daemon_rfid_reader.py`
- Triggers: System startup (as background daemon)
- Responsibilities:
  - Continuous polling of RFID hardware
  - Card ID deduplication and rate limiting
  - Subprocess invocation of rfid_trigger_play.sh
  - Signal handling for card removal (place-not-swipe mode)

**GPIO Control (Hardware Buttons/Rotary):**
- Location: `components/gpio_control/gpio_control.py`
- Triggers: System startup (as background daemon)
- Responsibilities:
  - Parse INI configuration for attached devices
  - Listen for GPIO pin state changes
  - Execute mapped function calls from phoniebox_function_calls class
  - Dynamic LED status feedback

**Web Application:**
- Location: `htdocs/index.php`
- Triggers: HTTP request to root path
- Responsibilities:
  - Load configuration and language
  - Render navigation and current player status
  - Display file browser, playlist, settings forms
  - Include JavaScript for AJAX control

**Installation:**
- Location: `scripts/installscripts/install-jukebox.sh`
- Triggers: Manual execution during setup
- Responsibilities:
  - Detect OS version and hardware
  - Clone/download Phoniebox code
  - Install dependencies (Python, PHP, MPD, etc.)
  - Create initial configuration
  - Set up systemd services for daemons

**Configuration Merge:**
- Location: `scripts/inc.writeGlobalConfig.sh`
- Triggers: Manual execution or after UI settings change
- Responsibilities:
  - Read sample config files for structure
  - Create missing individual settings with defaults
  - Merge all settings into global.conf
  - Update permissions for web server access

## Error Handling

**Strategy:** Graceful degradation with logging, subprocess error codes checked in PHP.

**Patterns:**

- **Shell script exit codes:** playout_controls.sh returns 0 on success, non-zero on failure; PHP common.php checks RC and logs
- **Python exceptions:** Reader.py exits if device not found; daemon_rfid_reader.py catches OSError and logs via logger module
- **Config missing:** inc.writeGlobalConfig.sh creates defaults if file not found; system continues with safe values
- **MPD unavailable:** API catches socket errors and returns HTTP 500; web UI shows alert to user
- **RFID device missing:** RegisterDevice.py checks if device registered; daemon refuses to start without it

## Cross-Cutting Concerns

**Logging:**
- Debug logging controlled by `settings/debugLogging.conf` with granular per-component flags (DEBUG_WebApp_API, DEBUG_inc_writeGlobalConfig_sh, etc.)
- Destination: `logs/debug.log` via `file_put_contents(..., FILE_APPEND | LOCK_EX)`
- Pattern: Conditional logging with global $debugLoggingConf checked at runtime

**Validation:**
- RFID card IDs validated against regex pattern in daemon_rfid_reader.py (numeric extraction)
- Card delay checks prevent rapid re-triggers via same_id_delay setting
- MPD responses parsed with regex for key-value extraction in PHP API
- GPIO config validation in ConfigCompatibilityChecks class

**Authentication:**
- Web UI assumes running on trusted local network (no authentication in version 2)
- Shell script execution via PHP uses `sudo` for privileged operations (GPIO, shutdown)
- RFID daemon runs as system service with appropriate permissions

**File Permissions:**
- Installation script sets 775 on config files, owned by pi:www-data
- Shared audio/shortcut folders world-readable for SMB/NFS access
- Debug logs world-readable, archived in logs/ directory

---

*Architecture analysis: 2026-02-06*
