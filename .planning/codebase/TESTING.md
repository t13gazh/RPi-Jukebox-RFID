# Testing Patterns

**Analysis Date:** 2026-02-06

## Test Framework

**Python:**
- Runner: pytest (version in requirements.txt)
- Config: No explicit pytest.ini found, configured in `composer.json` scripts
- Assertion Library: Built-in pytest assertions
- Mock Library: `mock` package (python-mock)
- Coverage: pytest-cov with coverage configuration in `.coveragerc`

**PHP:**
- Runner: PHPUnit 11 (via Composer)
- Config: `composer.json` with test scripts defined
- Assertion Library: PHPUnit\Framework\TestCase
- Mock Library: php-mock/php-mock-phpunit v2 for mocking native functions
- Coverage: XDebug enabled in CI workflow

**Run Commands:**

Python:
```bash
pytest --cov --cov-config=.coveragerc --cov-report xml   # Run all Python tests with coverage
pytest --watch                                             # Watch mode (if pytest-watch installed)
flake8 --config .flake8                                   # Lint check
```

PHP:
```bash
composer run-script test                   # Run tests excluding real-env group (from composer.json)
composer run-script test-all               # Run all tests including real-env group
composer validate --strict                 # Validate composer.json
```

## Test File Organization

**Python Location:**
- Co-located pattern: Tests in `test/` subdirectory next to source
- Example: `components/gpio_control/test/` contains tests for `components/gpio_control/`
- Sub-module structure: `components/gpio_control/test/test_SimpleButton.py` mirrors `components/gpio_control/GPIODevices/simple_button.py`

**PHP Location:**
- Separate tree: `tests/htdocs/` mirrors `htdocs/`
- Example: `tests/htdocs/api/PlayerTest.php` tests `htdocs/api/player.php`

**Naming:**
- Python: `test_*.py` prefix pattern (e.g., `test_gpio_control.py`, `test_LED.py`, `test_SimpleButton.py`)
- PHP: `*Test.php` suffix pattern (e.g., `PlayerTest.php`, `PlayListTest.php`, `TrackEditTest.php`)

## Test Structure

**Python Suite Organization:**

Class-based with fixtures:
```python
import pytest
from mock import patch, call, MagicMock

@pytest.fixture
def gpio_control_class():
    class MockFunctionCalls:
        def funcTestWithoutParameter(*args):
            return "funcTestWithoutParameter()"
        def funcTestWithParameter(param1, *args):
            return f"funcTestWithParameter({str(param1)})"

    _gpio_control_class = GPIOControl(MockFunctionCalls)
    return _gpio_control_class

class TestGPIOControl:
    def test_getAllDevices_empty(self, gpio_control_class):
        config = configparser.ConfigParser()
        devices = gpio_control_class.get_all_devices(config)
        assert not devices

    def test_getAllDevices_type_known(self, gpio_control_class):
        # ... test implementation
        pass
```

**Patterns Observed:**
- Fixture setup: `@pytest.fixture` for shared test objects
- Reset mocks between tests: `MockRPi.reset_mock()`, `GPIO.reset_mock()`
- Mock return values: `GPIO.input.side_effect = [...]` for sequenced returns
- Assertion style: Simple `assert` statements with boolean checks
- Callback testing: Side-effect functions for complex mock behavior

**PHP Suite Organization:**

Test case classes with setUp/tearDown:
```php
<?php
namespace JukeBox\Api;

use PHPUnit\Framework\TestCase;
use phpmock\phpunit\PHPMock;

class PlayerTest extends TestCase {
    use PHPMock;

    public function setUp(): void {
        $parse_ini_file = $this->getFunctionMock(__NAMESPACE__, 'parse_ini_file');
        $parse_ini_file->expects($this->atLeastOnce())->willReturn([...]);
        $_SERVER['REQUEST_METHOD'] = '';
        require_once 'htdocs/api/player.php';
    }

    public function testReturnHandleGet() {
        $exec = $this->getFunctionMock(__NAMESPACE__, 'exec');
        $exec->expects($this->once())->willReturnCallback(function(...) {...});
        $header = $this->getFunctionMock(__NAMESPACE__, 'header');
        $header->expects($this->once());
        $this->expectOutputString('{"key":"value"}');
        handleGet();
    }
}
```

**Patterns Observed:**
- setUp method: Run before each test, initializes mocks and includes source file
- PHPMock trait usage: Mock native PHP functions like `parse_ini_file()`, `exec()`, `header()`
- Callback assertions: `willReturnCallback(function...)` for complex behavior
- Output expectations: `$this->expectOutputString()` for testing echo output
- Process isolation: `@runInSeparateProcess` annotation for tests that affect global state

## Mocking

**Python Framework:**
- Library: `mock` package (from unittest.mock or standalone mock)
- Types of mocks: `MagicMock()`, `patch()`, `patch.object()`

**Patterns:**
```python
# Mock object creation
mockedAction = MagicMock()

# Patch function/method at decorator level
@patch.object(GPIOControl, 'getFunctionCall', mock_gpio_control_getFunctionCall)
def test_func(gpio_control_class):
    # ...
    pass

# Patch with context manager
with patch('builtins.print') as mock_print:
    gpio_control_class.print_all_devices()
    mock_print.assert_has_calls([call("test1"), call("test2")])

# Mock side effects
GPIO.input.side_effect = lambda *args: 1
GPIO.input.side_effect = [False, True]  # Sequence of return values

# Assert call counts
mock_print.assert_called_once()
mock_print.assert_not_called()
mock_print.assert_has_calls([...])
```

**PHP Framework:**
- Library: phpmock\phpunit\PHPMock trait
- Mocks native functions in test namespace

**Patterns:**
```php
// Mock native function
$parse_ini_file = $this->getFunctionMock(__NAMESPACE__, 'parse_ini_file');
$parse_ini_file->expects($this->once())->willReturn(['KEY' => 'value']);

// Mock with callback for complex behavior
$exec = $this->getFunctionMock(__NAMESPACE__, 'exec');
$exec->expects($this->once())->willReturnCallback(
    function ($command, &$output, &$returnValue) {
        $output = ["response"];
        $returnValue = 0;
    }
);

// Assert expectations
$this->assertEquals($expected, $actual);
$this->assertTrue(condition);
```

**What to Mock:**
- External system calls: `exec()`, `shell_exec()`, `system()`
- File operations: `file_get_contents()`, `file_exists()`, `fopen()`
- Configuration loading: `parse_ini_file()`
- Hardware access: `GPIO` module in Python
- Network operations: `socket_*` functions

**What NOT to Mock:**
- Pure logic functions (test actual behavior)
- String manipulation, math operations
- Data structure operations
- Helper parsing functions (test with real data)

## Fixtures and Factories

**Python Fixtures:**

Example with parametrization:
```python
# From conftest.py - shared fixtures
import sys, os
from mock import MagicMock, patch

MockRPi = MagicMock()
modules = {
    "RPi": MockRPi,
    "RPi.GPIO": MockRPi.GPIO,
}
MockRPi.GPIO.RISING = 31
MockRPi.GPIO.FALLING = 32
MockRPi.GPIO.BOTH = 33
patcher = patch.dict("sys.modules", modules)
patcher.start()

# Fixture with reset
@pytest.fixture
def simple_button():
    mockedAction.reset_mock()
    mockedSecAction.reset_mock()
    return SimpleButton(pin=1, action=mockedAction, action2=mockedSecAction, name='TestButton')
```

**Test Data:**
- Located in: `conftest.py` (shared) and inline in test files
- Constants defined at module level (e.g., `pinA = 1`, `pinB = 2` in test_RotaryEncoder.py)
- Config objects built via `configparser.ConfigParser()` with inline dicts
- Temporary files/folders created in setUp, cleaned in tearDown (TrackEditTest.php)

**Location:**
- Python: `conftest.py` at package level for shared fixtures
- PHP: setUp/tearDown methods in test classes
- Mock objects: Module-level in both (reused across tests with reset between test runs)

## Coverage

**Configuration:**
- Coverage file: `.coveragerc` at project root
- Omit patterns:
  - `**/test/*` - exclude all test directories
  - `scripts/*` - exclude shell scripts

**View Coverage:**
```bash
pytest --cov --cov-config=.coveragerc --cov-report html   # Generate HTML report
pytest --cov --cov-config=.coveragerc --cov-report term   # Terminal report
```

**CI Integration:**
- Python: Coveralls integration in GitHub Actions (pythonpackage.yml)
  - Runs on multiple Python versions (3.9, 3.10, 3.11, 3.12, 3.13)
  - Uses parallel build strategy
  - Reports to coverallsapp/github-action
- PHP: XDebug coverage in php.yml workflow

**Requirements:**
- No explicit coverage percentage enforced (not detected in config)
- Coverage reports generated and sent to Coveralls for tracking

## Test Types

**Unit Tests:**
- Scope: Single class or function (most common pattern)
- Approach: Mock all dependencies, test in isolation
- Examples:
  - `test_LED.py`: Tests LED class methods directly
  - `test_SimpleButton.py`: Tests button behavior with mocked GPIO
  - `test_RotaryEncoder.py`: Tests encoder state machine with mocked callbacks

**Integration Tests:**
- Scope: Multiple components working together
- Approach: Mock only external services (exec calls, file I/O)
- Examples:
  - `test_gpio_control.py`: Tests device creation and configuration parsing
  - `PlayerTest.php`: Tests handleGet/handlePut with exec and header mocks
  - `PlayListTest.php`: Tests playlist operations with mpd command mocking

**E2E Tests:**
- Framework: Not detected in current codebase
- Shell script tests exist: `tests/test-commandline-shellscripts.sh` (minimal usage detected)

## Common Patterns

**Async Testing:**

Python with time mocking:
```python
with patch('time.sleep'):
    with patch('GPIODevices.led.system') as mock_system:
        mock_system.side_effect = [False]
        _led = StatusLED(pin=1)
        mock_system.assert_called_with('systemctl is-active --quiet phoniebox-startup-scripts.service')
```

**Error Testing:**

Python:
```python
def test_init_edge_invalid(self):
    with pytest.raises(KeyError) as e:
        SimpleButton(pin=1, edge='invalid')
    assert str(e.value) == "'Unknown Edge type invalid'"
```

PHP:
```python
def testHandlePutFails(self):
    $file_get_contents = $this->getFunctionMock(__NAMESPACE__, 'file_get_contents');
    $file_get_contents->expects($this->atLeastOnce())->will($this->returnValue(null));
    $this->expectOutputString('playlist attribute missing');
    handlePut();
```

**Parametrized Tests:**

Pattern observed but not extensively used. Can add with `@pytest.mark.parametrize`:
```python
@pytest.mark.parametrize("edge,expected", [
    ('falling', GPIO.FALLING),
    ('rising', GPIO.RISING),
    ('both', GPIO.BOTH),
])
def test_edge_parsing(edge, expected):
    button = SimpleButton(pin=1, edge=edge)
    # ...
```

## Test Execution in CI/CD

**Python (GitHub Actions - pythonpackage.yml):**
1. Setup Python (matrix: 3.9, 3.10, 3.11, 3.12, 3.13)
2. Install dependencies (requirements.txt, requirements-GPIO.txt, spidev, wheel)
3. Run flake8 linting
4. Run pytest with coverage
5. Report to Coveralls

**PHP (GitHub Actions - php.yml):**
1. Setup PHP 8.x with XDebug
2. Validate composer.json
3. Install dependencies via composer
4. Run `composer run-script test` (excludes real-env tests)

## Recommended Testing Patterns for New Code

- Use fixtures for shared setup via `@pytest.fixture`
- Always reset mocks between tests (call `.reset_mock()` or use fresh MagicMock instances)
- Mock external calls (file I/O, system calls, network)
- Test both happy path and error conditions
- Use descriptive test names following pattern: `test_<function>_<scenario>`
- Group related tests in class (TestGPIOControl, TestButton, etc.)
- For PHP: Use `@runInSeparateProcess` when tests modify globals or require include-once files
- For PHP: Always mock `parse_ini_file` and file operations in setUp
- For Python: Use pytest markers for test categorization (`@pytest.mark.skip`, `@pytest.mark.xfail`)
- Create temporary files/directories in setUp, clean in tearDown
- Use `side_effect` for sequential returns or callable behavior
- Keep individual test assertions focused (one logical assertion per test)

---

*Testing analysis: 2026-02-06*
