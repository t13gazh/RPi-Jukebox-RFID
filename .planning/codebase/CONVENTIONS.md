# Coding Conventions

**Analysis Date:** 2026-02-06

## Naming Patterns

**Files:**
- Python: lowercase with underscores (`gpio_control.py`, `simple_button.py`, `rotary_encoder.py`)
- PHP: camelCase with extensions (`player.php`, `common.php`, `PlayListTest.php`)
- Test files: prefix with `test_` for Python (`test_gpio_control.py`, `test_LED.py`) or suffix with `Test` for PHP (`PlayerTest.php`, `PlayListTest.php`)

**Functions:**
- Python: snake_case (`get_all_devices()`, `generate_device()`, `getFunctionCall()` - mixed in some older code, see `_Callback()` with underscore prefix for private)
- PHP: camelCase (`handleGet()`, `handlePut()`, `execScript()`, `execSuccessfully()`)
- Private functions: Python uses underscore prefix (`_Callback()`, `_is_active`)
- Classes: PascalCase in both languages (`GPIOControl`, `SimpleButton`, `StatusLED`, `RotaryEncoder`, `PlayListTest`)

**Variables:**
- Python: snake_case (`pin_a`, `pin_b`, `time_base`, `is_active`, `mock_init`)
- PHP: camelCase mixed with underscore use (`$debugLoggingConf`, `$globalConf`, `$Audio_Folders_Path`, `$Latest_Folder_Played`, `$functionCall1Args`)
- Constants: UPPER_CASE when used (`KeyIncr = 0b00000010`, `KeyDecr = 0b00000001`)
- Mocked objects: prefix with `Mock` or `mocked` (`MockFunctionCalls`, `mockedAction`, `mockCallDecr`)

**Types:**
- Python: Type hints not enforced but used in some files
- PHP: Uses type declarations in function signatures (`: void`, type hints in parameters)

## Code Style

**Formatting:**
- EditorConfig (`.editorconfig`) enforces:
  - Unix-style newlines (LF)
  - Insert final newline on all files
  - Trim trailing whitespace
  - UTF-8 charset
  - 4-space indentation for Python and PHP
  - 2-space indentation for YAML and JavaScript
  - Markdown trim trailing whitespace disabled (allow on content lines)

**Linting:**
- Python: Flake8 with custom rules in `.flake8`
  - Max line length: 127 characters
  - Max complexity: 12 (McCabe)
  - Ignores: E126, E127, E128 (indentation issues), W503 (line break before binary), D400, D401 (docstring rules)
  - Per-file ignores: `__init__.py:F401` (imported but unused), components-specific exceptions for MQTT and USB encoder
  - Reports: count, statistics, show-source enabled

- PHP: Composer-based testing with PHPUnit 11, uses phpmock for function mocking

## Import Organization

**Python Order:**
1. Standard library imports (`sys`, `os`, `configparser`, `logging`, `signal`)
2. Third-party library imports (`RPi.GPIO`, `evdev`, `pytest`, `mock`)
3. Local imports (relative and absolute from components)

Example from `gpio_control.py`:
```python
import configparser
import os
import logging

from signal import pause
from RPi import GPIO
from GPIODevices import (RotaryEncoder,
                         TwoButtonControl,
                         ShutdownButton,
                         SimpleButton,
                         LED,
                         StatusLED)
from function_calls import phoniebox_function_calls
from config_compatibility import ConfigCompatibilityChecks
```

**PHP Order:**
1. Opening tag `<?php`
2. Namespace declaration `namespace JukeBox\Api;` or `namespace JukeBox;`
3. `use` statements for imports
4. Configuration/globals loading
5. Function definitions
6. Closing tag `?>`

Example from `player.php`:
```php
<?php
namespace JukeBox\Api;

/***
 * Comment block
 */
include 'common.php';

$debugLoggingConf = parse_ini_file("../../settings/debugLogging.conf");
$globalConf = parse_ini_file("../../settings/global.conf");
```

**Path Aliases:**
- Python: Uses relative imports with direct module references (`from GPIODevices import ...`, `from function_calls import ...`)
- PHP: Uses namespaces and file includes (`include 'common.php'`, `use JukeBox\Utils\Files;`)

## Error Handling

**Patterns:**
- Python: Try-except blocks with logging, returns None for failure conditions
  - `try/except AttributeError` for dynamic function calls (line 39-40 in `gpio_control.py`)
  - `try/except KeyError` for dictionary/config parsing
  - `except Exception` for broad catching with logging warning (buttons_usb_encoder.py)
- PHP: Conditional checks with HTTP response codes
  - `http_response_code(500)` on exec failure
  - `http_response_code(405)` for unsupported HTTP methods
  - `http_response_code(400)` for bad request (missing body command)
  - Suppression operator `@` used (e.g., `@json_decode()`)

## Logging

**Framework:**
- Python: Standard `logging` module
- PHP: File-based logging to `../../logs/debug.log`

**Patterns:**
- Python: Logger created per module with `logging.getLogger(__name__)` (seen in `led.py`, `rotary_encoder.py`)
  - Levels used: `debug()`, `info()`, `warning()`, `error()`
  - Format includes function name and context in messages
- PHP: Debug logging checks config value before writing
  ```php
  if($debugLoggingConf['DEBUG_WebApp_API'] == "TRUE") {
      file_put_contents("../../logs/debug.log", "\n# message", FILE_APPEND | LOCK_EX);
  }
  ```

## Comments

**When to Comment:**
- Block comments for major sections (seen in `player.php` and test setup)
- Inline comments explain non-obvious logic (e.g., GPIO flag handling in `rotary_encoder.py`)
- Comments for workarounds or known issues (TODO comments in test files like line 58 in `test_SimpleButton.py`)

**JSDoc/TSDoc:**
- PHP uses DocBlock style (seen in `PlayerTest.php` and `PlayListTest.php`):
  ```php
  /**
   * @runInSeparateProcess
   * @group real-env
   */
  ```
- Python uses minimal docstrings, focus on clear function names

## Function Design

**Size:**
- Python: Functions range from 5-20 lines for simple operations, 40+ lines for complex logic (e.g., `generate_device()` spans lines 43-101)
- Methods are generally focused on single responsibility

**Parameters:**
- Python: Uses positional + keyword arguments with defaults
  ```python
  def __init__(self, pinA, pinB, functionCallIncr=None, functionCallDecr=None, timeBase=0.1, name='RotaryEncoder'):
  ```
- Avoid excessive parameters, group related ones
- PHP: Similar approach with type hints in newer code
  ```php
  public function testHandlePutSuccess() {
  ```

**Return Values:**
- Python: Explicit returns, None for no result
- PHP: Functions may echo directly (imperative style) or return values
- Consistent return type handling (always return value or None, not mixed)

## Module Design

**Exports:**
- Python: Uses `__init__.py` barrel files to export classes
  ```python
  # GPIODevices/__init__.py
  from .rotary_encoder import RotaryEncoder
  from .two_button_control import TwoButtonControl
  from .shutdown_button import ShutdownButton
  from .simple_button import SimpleButton
  from .led import LED, StatusLED
  ```

**Barrel Files:**
- `components/gpio_control/GPIODevices/__init__.py` exports all GPIO device classes
- `components/controls/__init__.py` exists (empty or minimal)
- Enables clean imports: `from GPIODevices import (RotaryEncoder, LED, ...)`

## Recommended Conventions for New Code

- Follow the 4-space indentation standard (enforced by EditorConfig)
- Use snake_case for Python functions/variables, camelCase for PHP
- Add logging at INFO level for major operations, DEBUG for detailed flow
- Always set `logging.getLogger(__name__)` in Python modules
- Use relative imports with barrel files in `__init__.py`
- Keep functions under 50 lines; break complex logic into helpers
- Use type hints in Python where reasonable (not enforced but helpful)
- For PHP, always declare namespace early and use proper function signatures with type hints
- Mock external dependencies in tests; avoid system calls where possible

---

*Convention analysis: 2026-02-06*
