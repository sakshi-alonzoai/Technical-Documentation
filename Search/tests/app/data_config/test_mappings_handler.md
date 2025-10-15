# Documentation for `test_mappings_handler.py`

The following documentation provides a comprehensive, line-by-line analysis of the Python script `tests/app/data_config/test_mappings_handler.py`.

---

## Technical Documentation: `tests/app/data_config/test_mappings_handler.py`

### 1. High-Level Overview

This Python script is a Pytest-based test suite designed to thoroughly test the `MappingsHandler` class, located at `app.data_config.mappings.mappings_handler`. The `MappingsHandler` class is expected to be responsible for loading, parsing, and providing access to various data mappings (e.g., table names, fields for queries, statistical mappings, query samples, position mappings, and metadata) typically stored in JSON configuration files.

The test suite employs extensive use of the `unittest.mock` library to isolate the `MappingsHandler` class from the actual file system. This is achieved by mocking file I/O operations (`builtins.open` and `os.path.exists`) and JSON parsing (`json.load`) to provide predefined data, ensuring that tests are fast, reliable, and do not depend on the presence of actual configuration files. Each test function focuses on verifying a specific method or aspect of the `MappingsHandler`'s functionality.

### 2. Dependencies and Imports

The script begins by importing necessary modules:

*   **`pytest`**: The testing framework used to discover and run tests.
*   **`builtins`**: This module provides access to all built-in identifiers. It is used here to patch the global `open()` function.
*   **`unittest.mock.patch`**: A decorator/context manager used to temporarily replace objects with mock objects.
*   **`unittest.mock.mock_open`**: A helper function from `unittest.mock` that creates a mock object to replace `open()`, simulating file system operations (like reading from a file).
*   **`app.data_config.mappings.mappings_handler.MappingsHandler`**: The actual class under test.

```python
# tests/app/data_config/test_mappings_handler.py

import pytest
import builtins
from unittest.mock import patch, mock_open
from app.data_config.mappings.mappings_handler import MappingsHandler
```

### 3. Fixtures

Fixtures are functions that Pytest runs before test functions. They are used to set up a baseline for tests.

#### `mock_handler`

```python
@pytest.fixture
def mock_handler():
    with patch("builtins.open", mock_open(read_data='{"key": "value"}')), \
         patch("os.path.exists", return_value=True), \
         patch("json.load", return_value={"query_samples": ["sample query"]}):
        yield MappingsHandler()
```

*   **`@pytest.fixture`**: This decorator marks `mock_handler` as a Pytest fixture.
*   **`def mock_handler():`**: Defines the fixture function.
*   **`with patch("builtins.open", mock_open(read_data='{"key": "value"}'))`**: This line uses `unittest.mock.patch` as a context manager to temporarily replace the global `builtins.open` function.
    *   `mock_open(read_data='{"key": "value"}')` creates a mock object that simulates an open file. When `open()` is called by `MappingsHandler` internally, this mock will be used, and its `read()` method will return the string `{"key": "value"}`. This simulates reading a JSON string from a file.
*   **`patch("os.path.exists", return_value=True)`**: This line patches `os.path.exists`, making it always return `True`. This simulates that any file `MappingsHandler` attempts to check for existence *does* exist, preventing `FileNotFoundError` or similar issues during testing.
*   **`patch("json.load", return_value={"query_samples": ["sample query"]})`**: This line patches `json.load`, which is typically used to parse JSON data from a file-like object. Instead of actually parsing, this mock directly returns a predefined dictionary `{"query_samples": ["sample query"]}`. This ensures that `MappingsHandler` receives a parsed dictionary, regardless of the `read_data` provided by `mock_open`. In this fixture, the `json.load` patch effectively overrides the content `mock_open` would provide to `json.load`.
*   **`yield MappingsHandler()`**: This provides an instance of `MappingsHandler` to any test function that requests `mock_handler` as a parameter. The patching is active for the duration the `MappingsHandler` instance is used by the test. After the test finishes, the patches are automatically reverted.

### 4. Test Functions

Each function prefixed with `test_` is a Pytest test case.

#### `test_get_table_name`

```python
@patch("builtins.open", new_callable=mock_open, read_data='{"MFB": {"Player": {"game": "player_game_table"}}}')
@patch("json.load", return_value={"MFB": {"Player": {"game": "player_game_table"}}})
def test_get_table_name(mock_json, mock_file):
    handler = MappingsHandler()
    assert handler.get_table_name("MFB", "Player", "game") == "player_game_table"
```

*   **`@patch("builtins.open", new_callable=mock_open, read_data='...')`**: This decorator patches `builtins.open`.
    *   `new_callable=mock_open` ensures that `mock_open` is called to create the mock object.
    *   `read_data='{"MFB": {"Player": {"game": "player_game_table"}}}'` specifies the string content that the mocked file will return when read. This simulates the `mappings.json` file content.
*   **`@patch("json.load", return_value={"MFB": {"Player": {"game": "player_game_table"}}})`**: This decorator patches `json.load` to directly return a specific dictionary. This simulates the result of parsing the JSON content from the file.
*   **`def test_get_table_name(mock_json, mock_file):`**: Defines the test function. `mock_json` and `mock_file` are parameters that receive the mock objects created by the `@patch` decorators (though their values are not explicitly used within the function body, the patches themselves are active).
*   **`handler = MappingsHandler()`**: An instance of the `MappingsHandler` is created. During its initialization (or when it first attempts to load mappings), it will interact with the patched `open` and `json.load`.
*   **`assert handler.get_table_name("MFB", "Player", "game") == "player_game_table"`**: This assertion calls the `get_table_name` method with specific parameters and verifies that it returns the expected table name, which comes from the mocked JSON data.

#### `test_get_fields_for_query`

```python
@patch("builtins.open", new_callable=mock_open, read_data='{"MFB": {"Player": {"game": {"basic": ["stat1", "stat2"]}}}}')
@patch("json.load", return_value={"MFB": {"Player": {"game": {"basic": ["stat1", "stat2"]}}}})
def test_get_fields_for_query(mock_json, mock_file):
    handler = MappingsHandler()
    fields = handler.get_fields_for_query("MFB", "Player", "game", "basic")
    assert fields == ["stat1", "stat2"]
```

*   **`@patch(...)`**: Similar to the previous test, `builtins.open` and `json.load` are patched to simulate a JSON structure that defines fields for a query type. The `read_data` and `return_value` reflect this specific structure.
*   **`def test_get_fields_for_query(mock_json, mock_file):`**: Defines the test function.
*   **`handler = MappingsHandler()`**: Instantiates `MappingsHandler`.
*   **`fields = handler.get_fields_for_query("MFB", "Player", "game", "basic")`**: Calls the `get_fields_for_query` method with a team, entity, game type, and query type.
*   **`assert fields == ["stat1", "stat2"]`**: Asserts that the returned list of fields matches the expected list from the mocked JSON data.

#### `test_load_filter_mappings`

```python
def test_load_filter_mappings(mock_handler):
    result = mock_handler.load_filter_mappings()
    assert isinstance(result, dict)
```

*   **`def test_load_filter_mappings(mock_handler):`**: Defines the test function, directly utilizing the `mock_handler` fixture. This means `builtins.open`, `os.path.exists`, and `json.load` are already patched as defined in the fixture.
*   **`result = mock_handler.load_filter_mappings()`**: Calls the `load_filter_mappings` method on the mocked handler instance.
*   **`assert isinstance(result, dict)`**: Asserts that the result of loading filter mappings is a dictionary. The exact content is not checked here, only its type, relying on the fixture's generic `{"key": "value"}` or `{"query_samples": ["sample query"]}` mocks, or the specific `json.load` patch for `MappingsHandler`'s constructor.

#### `test_get_stat_mapping`

```python
@patch("builtins.open", new_callable=mock_open, read_data='[{"stat": "s1", "label": "Stat One"}]')
@patch("json.load", return_value=[{"stat": "s1", "label": "Stat One"}])
def test_get_stat_mapping(mock_json, mock_file):
    handler = MappingsHandler()
    mapping = handler.get_stat_mapping("MFB", "Player")
    assert isinstance(mapping, list)
```

*   **`@patch(...)`**: Patches `builtins.open` and `json.load` to simulate a JSON file containing a list of dictionaries, representing stat mappings.
*   **`def test_get_stat_mapping(mock_json, mock_file):`**: Defines the test function.
*   **`handler = MappingsHandler()`**: Instantiates `MappingsHandler`.
*   **`mapping = handler.get_stat_mapping("MFB", "Player")`**: Calls the `get_stat_mapping` method. The parameters "MFB" and "Player" suggest that stat mappings might be context-specific.
*   **`assert isinstance(mapping, list)`**: Asserts that the returned mapping is a list, as expected from the mocked JSON structure.

#### `test_get_query_samples_dict`

```python
@patch("builtins.open", new_callable=mock_open, read_data='{"query_samples": ["sample1"]}')
@patch("json.load", return_value={"query_samples": ["sample1"]})
def test_get_query_samples_dict(mock_json, mock_file):
    handler = MappingsHandler()
    samples = handler.get_query_samples("MFB", "Player")
    assert samples == ["sample1"]
```

*   **`@patch(...)`**: Patches `builtins.open` and `json.load` to simulate a JSON file that contains a dictionary with a "query_samples" key, whose value is a list of sample queries.
*   **`def test_get_query_samples_dict(mock_json, mock_file):`**: Defines the test function.
*   **`handler = MappingsHandler()`**: Instantiates `MappingsHandler`.
*   **`samples = handler.get_query_samples("MFB", "Player")`**: Calls the `get_query_samples` method.
*   **`assert samples == ["sample1"]`**: Asserts that the returned list of samples matches the expected list from the mocked JSON data. This test specifically checks the case where samples are nested under a key.

#### `test_get_query_samples_list`

```python
@patch("builtins.open", new_callable=mock_open, read_data='["sample1", "sample2"]')
@patch("json.load", return_value=["sample1", "sample2"])
def test_get_query_samples_list(mock_json, mock_file):
    handler = MappingsHandler()
    samples = handler.get_query_samples("MFB", "Player")
    assert "sample1" in samples
```

*   **`@patch(...)`**: Patches `builtins.open` and `json.load` to simulate a JSON file that directly contains a list of sample queries (not nested under a key).
*   **`def test_get_query_samples_list(mock_json, mock_file):`**: Defines the test function.
*   **`handler = MappingsHandler()`**: Instantiates `MappingsHandler`.
*   **`samples = handler.get_query_samples("MFB", "Player")`**: Calls the `get_query_samples` method.
*   **`assert "sample1" in samples`**: Asserts that "sample1" is present in the returned list of samples. This tests the scenario where the query samples JSON is a direct list.

#### `test_get_pos_mapping`

```python
def test_get_pos_mapping(mock_handler):
    result = mock_handler.get_pos_mapping("MFB")
    assert isinstance(result, dict)
```

*   **`def test_get_pos_mapping(mock_handler):`**: Defines the test function, utilizing the `mock_handler` fixture.
*   **`result = mock_handler.get_pos_mapping("MFB")`**: Calls the `get_pos_mapping` method with a team identifier.
*   **`assert isinstance(result, dict)`**: Asserts that the returned position mapping is a dictionary. The specific content depends on the generic mock data from the `mock_handler` fixture.

#### `test_get_metadata_mapping`

```python
def test_get_metadata_mapping(mock_handler):
    result = mock_handler.get_metadata_mapping()
    assert isinstance(result, dict)
```

*   **`def test_get_metadata_mapping(mock_handler):`**: Defines the test function, utilizing the `mock_handler` fixture.
*   **`result = mock_handler.get_metadata_mapping()`**: Calls the `get_metadata_mapping` method.
*   **`assert isinstance(result, dict)`**: Asserts that the returned metadata mapping is a dictionary. As with `test_load_filter_mappings` and `test_get_pos_mapping`, the specific content is determined by the generic `mock_handler` fixture's patched `json.load` return value.

### 5. Main Execution Flow (`__main__` block)

This script does not contain a traditional `if __name__ == "__main__":` block. As a Pytest test file, it is designed to be executed by the `pytest` command-line tool. When `pytest` is run in the directory containing this file (or a parent directory), it will automatically discover and execute all functions prefixed with `test_` and manage the execution of fixtures and the application of patches.

### 6. Important Implementation Details and Dependencies

*   **Extensive Mocking**: The entire test suite heavily relies on `unittest.mock` for isolating the `MappingsHandler` class. This is crucial for unit testing as it ensures tests are not affected by external factors like file system availability or actual file content.
*   **Simulated File System**: `builtins.open` and `os.path.exists` are mocked to simulate reading specific JSON content from files and confirming their existence, respectively.
*   **Simulated JSON Parsing**: `json.load` is frequently patched to return pre-defined Python objects (dictionaries or lists). This ensures that the tests control the exact data `MappingsHandler` receives after "loading" a JSON file, simplifying test logic and making tests robust to potential changes in JSON parsing itself.
*   **Test Case Isolation**: Each test function, especially those using `@patch` decorators, sets up its own isolated environment for the specific `MappingsHandler` method being tested. This prevents side effects between tests.
*   **`read_data` vs. `json.load`**: It's important to note the interaction. `mock_open(read_data='...')` provides the raw string content that `open().read()` would return. However, `json.load` is typically called on a file-like object, which then parses this raw string. By also patching `json.load` to return a specific Python object, these tests explicitly control the *parsed* data `MappingsHandler` receives, which is often more direct than relying on `json.loads` to correctly parse `mock_open`'s `read_data` in every scenario.
*   **Assumptions about `MappingsHandler`**: These tests implicitly reveal assumptions about `MappingsHandler`, such as:
    *   It expects to load various mappings from JSON files.
    *   It uses `os.path.exists` to check for file presence.
    *   It uses `json.load` to parse JSON files.
    *   Its methods (`get_table_name`, `get_fields_for_query`, `load_filter_mappings`, `get_stat_mapping`, `get_query_samples`, `get_pos_mapping`, `get_metadata_mapping`) follow specific argument patterns and return specific data types (dictionaries, lists, strings).
    *   It supports different JSON structures for query samples (a dictionary with a `query_samples` key or a direct list).
