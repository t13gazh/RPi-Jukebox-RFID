# Codebase Concerns

**Analysis Date:** 2026-02-06

## Tech Debt

**GPIO Button State Logic Bug:**
- Issue: In `components/gpio_control/GPIODevices/simple_button.py`, line 85, `self.pull_up` is hardcoded to `True` regardless of the `pull_up_down` parameter passed to constructor. Lines 84 and 181 have TODO comments indicating this is a known bug.
- Files: `C:\Users\herrkraft\Repos\rfid-jukebox\components\gpio_control\GPIODevices\simple_button.py` (lines 84-85, 181-184)
- Impact: Button press detection will always invert based on hardcoded `pull_up=True`, even when `pull_up_down='pull_down'` is configured. This causes GPIO buttons to be pressed in reverse when pull-down is configured.
- Fix approach: Replace hardcoded `self.pull_up = True` with `self.pull_up = (pull_up_down == GPIO.PUD_UP)` to correctly reflect the configuration.

**Shell Script Configuration File Paths Not Quoted:**
- Issue: Multiple shell scripts use unquoted variable paths in command substitution and file operations, violating shell best practices.
- Files: `scripts/rfid_trigger_play.sh` (lines 70, 43-44, 56, 60), `scripts/single_play.sh`, `scripts/playout_controls.sh`, and many others
- Impact: Paths with spaces or special characters will break script execution. Configuration files may not be found if they contain spaces.
- Fix approach: Quote all variable paths: `". $PATHDATA/../settings/global.conf"` → `". "$PATHDATA/../settings/global.conf""`

**Missing Error Handling in Shell Scripts:**
- Issue: Shell scripts lack error checking after critical operations (file reads, commands, variable assignments).
- Files: `scripts/daemon_rfid_reader.py` (lines 32-35, 40-41, 47-48), `scripts/rfid_trigger_play.sh` (line 38 sources config without checking existence)
- Impact: If configuration files are missing or corrupted, the daemon silently continues with undefined behavior. File operations may fail silently.
- Fix approach: Add `set -e` or check return codes: `[ -f "$file" ] || exit 1`

**Command Injection Vulnerability in PHP Files:**
- Issue: User input from POST/form data is directly interpolated into shell commands via `sed` and `exec()` without sanitization.
- Files: `htdocs/cardRegisterNew.php` (lines 118, 140, 145), `htdocs/inc.processCheckCardEditRegister.php` (lines 122, 248, 253, 268)
- Impact: Malicious RFID mappings or form inputs can execute arbitrary shell commands. For example, a CSV import with card ID `"'; rm -rf /"` would be executed directly.
- Fix approach: Use `escapeshellarg()` on all user inputs: `sed -i 's/%".$key."%/'.escapeshellarg($val).'/'`

**Unsafe Shell Execution in Python:**
- Issue: In `scripts/daemon_rfid_reader.py`, line 99 uses `subprocess.call()` with `shell=True` and unsanitized card ID.
- Files: `scripts/daemon_rfid_reader.py` (lines 71, 99)
- Impact: If RFID reader returns malicious input or is compromised, arbitrary commands can execute. Example: card ID `"$(rm -rf /)"` would execute.
- Fix approach: Use `subprocess.call()` without `shell=True` and pass arguments as list: `subprocess.call([dir_path + '/rfid_trigger_play.sh', '--cardid=' + cardid])`

**File Permission Issues:**
- Issue: Multiple files are created with overly permissive permissions (chmod 777).
- Files: `htdocs/cardRegisterNew.php` (line 156), `htdocs/inc.header.php` (line 147), `htdocs/inc.processCheckCardEditRegister.php` (line 248)
- Impact: Configuration files with 777 permissions are world-readable and world-writable, exposing RFID card mappings and system settings. Allows any user to modify critical configuration.
- Fix approach: Use restrictive permissions: `chmod 775` for directories, `chmod 644` for files (or `600` for sensitive configs).

**Hardcoded Paths in PHP Realpath Checks:**
- Issue: In `htdocs/api/common.php`, line 14 uses `realpath()` to validate command paths, but relative paths like `{"command"}` could still escape the scripts directory.
- Files: `htdocs/api/common.php` (lines 14, 23)
- Impact: Path traversal attacks possible if command contains `../`. Example: command `"../../../bin/bash"` might not be properly caught.
- Fix approach: Validate that realpath result starts with expected base: `if (strpos($absoluteCommand, $scriptsBase) !== 0) { exit; }`

**PHP Temporary File Upload Not Sanitized:**
- Issue: File uploads use original filename without validation or sanitization.
- Files: `htdocs/cardRegisterNew.php` (line 58, using `$_FILES['importFileUpload']['name']` directly)
- Impact: Uploaded filenames with special characters or path traversal (`../../../etc/passwd`) could be processed unsafely.
- Fix approach: Generate safe random filename: `$filename = bin2hex(random_bytes(8)) . '.csv'`

**Missing File Existence Checks Before Read Operations:**
- Issue: Configuration files are read without checking existence in multiple places.
- Files: `scripts/daemon_rfid_reader.py` (lines 32-35, 40-41, 47-48 - no checks before fopen)
- Impact: If configuration files are deleted, Python daemon crashes with file not found exception. No graceful fallback.
- Fix approach: Wrap file operations in try-catch and provide sensible defaults.

**Log File Permissions:**
- Issue: Debug log files are created without explicit permission controls.
- Files: `htdocs/inc.header.php` (line 88), various scripts writing to `debug.log`
- Impact: Debug logs contain sensitive information (card IDs, folder paths, commands) and are world-readable.
- Fix approach: Set log directory to `chmod 700` or `755` with file permissions `640` (owner read/write, group read, world nothing).

## Known Bugs

**Second Swipe Delay Logic Issue:**
- Symptoms: Second swipe pause not always respected; same RFID card triggers play immediately on second swipe.
- Files: `scripts/daemon_rfid_reader.py` (lines 97-108), `scripts/rfid_trigger_play.sh`
- Trigger: Quick double-swipe of same RFID card within configured `Second_Swipe_Pause` time
- Workaround: Increase `Second_Swipe_Pause` value in `settings/Second_Swipe_Pause` file

**PN532 RFID Reader Segmentation Faults:**
- Symptoms: RFID reading daemon crashes with segmentation fault
- Files: `components/rfid-reader/PN532/reset_pn532.sh` (line 10 has TODO comment acknowledging this)
- Trigger: Occurs during PN532 reader initialization or during certain read operations
- Workaround: Restart the RFID daemon service

**GPIO Pull-Up/Pull-Down Configuration Ignored:**
- Symptoms: GPIO buttons don't respond correctly or respond inverted from configuration
- Files: `components/gpio_control/GPIODevices/simple_button.py` (lines 84-85, 181-184)
- Trigger: Any GPIO button with `pull_up_down='pull_down'` configured will behave as if it has pull-up
- Workaround: Use pull-up configuration only, or manually adjust button sensitivity

## Security Considerations

**Unauthenticated Web Interface:**
- Risk: Web UI at `/htdocs/` has no authentication mechanism. Anyone with network access can control playback, upload music, modify RFID mappings, change system settings, and trigger shutdown.
- Files: All files in `htdocs/` (no auth.php or login mechanism exists)
- Current mitigation: Relies on network isolation (assumes private network only)
- Recommendations:
  1. Implement HTTP Basic Auth or session-based authentication
  2. Add CSRF tokens to all state-changing forms
  3. Restrict access via firewall rules
  4. Use HTTPS instead of HTTP (requires certificate setup)

**SQL Injection in Configuration Parsing:**
- Risk: While this system doesn't use SQL databases, configuration parsing with `parse_ini_file()` could be exploited if user controls the INI file paths.
- Files: `htdocs/inc.header.php` (line 91)
- Current mitigation: Paths are hardcoded, not user-controlled
- Recommendations: Document that INI files must be protected from user modification

**Stored Command Injection via RFID CSV Import:**
- Risk: Importing RFID cards from CSV allows setting system commands (lines starting with `%`). These commands are stored and executed later without validation.
- Files: `htdocs/cardRegisterNew.php` (lines 75-82, 129-147), `settings/rfid_trigger_play.conf`
- Current mitigation: None - any command in the list `CMD_*` is executed
- Recommendations:
  1. Whitelist allowed commands instead of allowing arbitrary entries
  2. Escape/validate command parameters before execution
  3. Add command execution audit logging

**Privilege Escalation via Sudo:**
- Risk: Many PHP scripts execute shell commands via `sudo` without explicit command whitelisting.
- Files: `htdocs/api/common.php` (lines 24, 33), `htdocs/inc.header.php` (lines 146-147), `htdocs/systemInfo.php` (lines 23, 28, 43)
- Current mitigation: Relies on sudoers configuration to limit allowed commands
- Recommendations:
  1. Document required sudoers rules and verify they are restrictive
  2. Create a wrapper script that only allows specific operations
  3. Audit all `exec()` and `shell_exec()` calls

**Unencrypted Configuration Storage:**
- Risk: RFID card mappings, WiFi credentials, and system settings stored in plain text configuration files.
- Files: `settings/global.conf`, `settings/rfid_trigger_play.conf`, WiFi configuration
- Current mitigation: File permissions (644 or 777)
- Recommendations:
  1. Document that configuration files should be on encrypted filesystems
  2. Move sensitive credentials (WiFi passwords) to a separate secrets file with 600 permissions
  3. Consider using environment variables for sensitive values

## Performance Bottlenecks

**Inefficient MPD Socket Communication:**
- Problem: Multiple socket operations in `htdocs/api/common.php` (lines 43-54) don't handle timeouts or connection pooling. Each API call opens a new socket.
- Files: `htdocs/api/common.php` (lines 43-54)
- Cause: Socket is created, used, and closed for every single command. No connection reuse.
- Improvement path:
  1. Implement socket connection pooling
  2. Add configurable timeouts
  3. Batch multiple MPD commands into single socket session

**PHP File Operations in Web Loop:**
- Problem: Web app reads configuration files on every page load without caching.
- Files: `htdocs/inc.header.php` (lines 82-91, reads and parses INI every request)
- Cause: No in-memory or file-based caching of configuration
- Improvement path:
  1. Implement PHP opcode caching (APCu)
  2. Cache parsed INI arrays for 60 seconds
  3. Invalidate cache only when config files change

**Shell Script Overhead in Tight Loops:**
- Problem: `daemon_rfid_reader.py` calls shell scripts for every RFID card (subprocess.call), spawning new processes.
- Files: `scripts/daemon_rfid_reader.py` (lines 71, 99)
- Cause: Each card swipe spawns `playout_controls.sh` and `rfid_trigger_play.sh` as separate processes
- Improvement path:
  1. Convert critical shell scripts to Python modules
  2. Use function calls instead of subprocess when possible
  3. Implement caching for frequently accessed configuration

**Synchronous File Operations:**
- Problem: Shell scripts wait synchronously for file I/O (reading config, writing logs) which blocks RFID reading.
- Files: Multiple shell scripts (rfid_trigger_play.sh, playout_controls.sh)
- Cause: All file operations are synchronous; no async or buffering
- Improvement path:
  1. Use memory-mapped files or in-memory configuration cache
  2. Implement asynchronous logging (queue + background writer)
  3. Batch multiple log writes into single file operation

## Fragile Areas

**RFID Reader Initialization:**
- Files: `scripts/daemon_rfid_reader.py` (lines 20-62), `components/rfid-reader/RC522/setup_rc522.sh`, `components/rfid-reader/PN532/reset_pn532.sh`
- Why fragile: Multiple hardware-specific reader implementations with minimal error handling. Reader initialization depends on exact GPIO pin configuration and driver versions.
- Safe modification:
  1. Add try-catch blocks around all reader operations
  2. Log all initialization steps with timestamps
  3. Implement reader health checks (periodic ping)
  4. Document exact hardware + OS version combinations tested
- Test coverage: Only unit tests for GPIO control exist; no integration tests for actual RFID readers

**Configuration File Parsing:**
- Files: `scripts/daemon_rfid_reader.py` (lines 32-62), `scripts/rfid_trigger_play.sh` (line 56), `htdocs/inc.header.php` (lines 82-91)
- Why fragile: Configuration sourcing via shell `. filename` and ini file parsing assumes exact file format. Missing newlines, encoding issues, or format changes break parsing silently.
- Safe modification:
  1. Wrap all source/include operations in validation
  2. Use schema validation (validate against expected keys)
  3. Provide clear error messages when parsing fails
- Test coverage: No automated tests for configuration parsing

**Database/Library Interaction:**
- Files: `htdocs/api/common.php` (MPD socket communication, lines 43-54)
- Why fragile: Socket operations have no error handling. If MPD is offline or unresponsive, requests hang.
- Safe modification:
  1. Add connection timeouts
  2. Implement retry logic with exponential backoff
  3. Add health check before critical operations
  4. Gracefully degrade if MPD is unavailable
- Test coverage: No integration tests with actual MPD daemon

**Web UI Form Processing:**
- Files: `htdocs/cardRegisterNew.php`, `htdocs/inc.processCheckCardEditRegister.php`
- Why fragile: Multi-step form processing with state stored in file system. If filesystem operations fail mid-process, state becomes inconsistent.
- Safe modification:
  1. Implement transaction-like behavior (write to temp file, move atomically)
  2. Add rollback capability if any step fails
  3. Validate all inputs before processing
- Test coverage: Only two PHP unit test files exist; coverage is minimal

## Scaling Limits

**Single-Threaded RFID Reading:**
- Current capacity: ~1 RFID card per 200ms (sleep 0.2s in daemon loop)
- Limit: If multiple cards are swiped rapidly or multiple readers are attached, cards will be missed.
- Scaling path:
  1. Implement multi-threaded reader support (one thread per reader)
  2. Use async I/O instead of sleep
  3. Add card queueing/buffering

**Configuration File Size:**
- Current capacity: Configuration files (`rfid_trigger_play.conf`, `global.conf`) are sourced with shell `. filename`
- Limit: Very large configuration files (10,000+ cards) will be slow to parse and load into memory
- Scaling path:
  1. Migrate to database or indexed key-value store
  2. Implement lazy loading (only load requested cards)
  3. Use SQLite for local caching with efficient lookup

**Log File Rotation:**
- Current capacity: Debug logging appends to single file indefinitely
- Limit: `logs/debug.log` grows without bound; disk space exhaustion possible on long-running systems
- Scaling path:
  1. Implement log rotation (logrotate or PHP log handlers)
  2. Add configurable log level and rotation policies
  3. Implement structured logging with timestamps for easier filtering

**Playlist Generation:**
- Current capacity: `scripts/playlist_recursive_by_folder.php` recursively scans filesystem for every card swipe
- Limit: Large music libraries (50,000+ files) will cause slow playout initiation (5-10 second delay)
- Scaling path:
  1. Pre-generate and cache playlists
  2. Use file indexing daemon (e.g., mlocate) for faster lookup
  3. Implement pagination for large folders

## Dependencies at Risk

**End-of-Life Python 2 Scripts:**
- Risk: Some legacy scripts may still use Python 2 syntax (though README shows Python 3 focus). Python 2 reached EOL in 2020.
- Impact: Security patches no longer available; incompatible with modern systems (Debian 12+)
- Migration plan: Audit all Python scripts for Python 2 syntax; convert to Python 3.9+

**Unmaintained RFID Reader Libraries:**
- Risk: `git+https://github.com/lthiery/SPI-Py.git` in `requirements.txt` is a GitHub repository without version pinning
- Impact: Upstream changes could break compatibility; no version control
- Migration plan:
  1. Pin SPI-Py to specific commit hash
  2. Consider forking if upstream becomes unmaintained
  3. Evaluate alternative SPI libraries

**yt-dlp Compatibility:**
- Risk: `yt-dlp` in `requirements.txt` unpinned version. YouTube changes their API frequently; yt-dlp must update constantly.
- Impact: YouTube downloads may suddenly fail without warning
- Migration plan:
  1. Pin to specific version: `yt-dlp==2024.01.01`
  2. Implement version checking and auto-update mechanism
  3. Add fallback to local library if download fails

**RPi.GPIO Deprecation:**
- Risk: `rpi-lgpio` is a shim for RPi.GPIO compatibility on newer Raspberry Pi OS. Original RPi.GPIO is deprecated in favor of `lgpio`.
- Impact: Long-term, this shim may not be maintained; GPIO functionality could break on major OS updates
- Migration plan:
  1. Plan migration to native `lgpio` library
  2. Create abstraction layer for GPIO operations
  3. Test on latest Raspberry Pi OS (Bookworm+)

**PHP Version Requirements:**
- Risk: Code uses older PHP patterns; compatibility with PHP 8.0+ is unclear (no version specified)
- Impact: May not run on modern hosting; security features of newer PHP not available
- Migration plan:
  1. Specify `php>=8.0` in installation requirements
  2. Add type hints to all function signatures
  3. Test on PHP 8.1+

## Missing Critical Features

**No Input Validation Framework:**
- Problem: No centralized input validation; sanitization scattered throughout codebase
- Blocks: Security audit; difficulty adding new features safely
- Recommendation: Implement validation helper functions or library

**No Automated Testing for Web UI:**
- Problem: Web UI changes have no automated tests; only manual testing
- Blocks: Confident refactoring; regression detection
- Recommendation: Add Selenium/Cypress tests for critical user flows

**No Configuration Validation Schema:**
- Problem: Configuration files lack schema validation; missing keys silently cause undefined behavior
- Blocks: Detecting configuration errors early
- Recommendation: Implement JSON Schema or similar for all configuration files

**No RFID Card Whitelisting/Blacklisting:**
- Problem: Any card read triggers assigned action; no ability to temporarily disable cards
- Blocks: Testing, maintenance, security lockdown
- Recommendation: Implement card enable/disable flag in configuration

**No Rate Limiting on Sensitive Operations:**
- Problem: No rate limiting on configuration changes, file uploads, or system commands
- Blocks: Protection against brute force or DOS attacks
- Recommendation: Add configurable rate limiting to API endpoints

## Test Coverage Gaps

**No Tests for Web UI Endpoints:**
- What's not tested: Form processing, file uploads, RFID CSV import, configuration changes
- Files: `htdocs/cardRegisterNew.php`, `htdocs/inc.processCheckCardEditRegister.php`, all API endpoints in `htdocs/api/`
- Risk: Refactoring or updates to these endpoints could break functionality silently
- Priority: High

**No Shell Script Integration Tests:**
- What's not tested: Script interactions, configuration file sourcing, error handling in shell scripts
- Files: `scripts/rfid_trigger_play.sh`, `scripts/playout_controls.sh`, `scripts/daemon_rfid_reader.py`
- Risk: Shell script changes could cause playback failures, configuration parsing errors, or infinite loops
- Priority: High

**No RFID Reader Integration Tests:**
- What's not tested: Actual RFID hardware communication, multiple reader support, reader failure scenarios
- Files: `components/rfid-reader/*/` (all reader implementations)
- Risk: Reader initialization bugs or hardware incompatibilities discovered only in production
- Priority: High

**Minimal GPIO Control Testing:**
- What's not tested: Actual GPIO pin communication, debouncing effectiveness, hold modes with real hardware
- Files: `components/gpio_control/test/test_*.py` (only unit tests exist, no integration tests)
- Risk: GPIO button behavior varies based on hardware; button lag or false triggers not detected
- Priority: Medium

**No End-to-End Integration Tests:**
- What's not tested: Full workflow from RFID card swipe to audio playback, web UI to actual system changes
- Files: None exist for E2E testing
- Risk: Interactions between components (RFID daemon → shell scripts → MPD) could be broken
- Priority: High

---

*Concerns audit: 2026-02-06*
