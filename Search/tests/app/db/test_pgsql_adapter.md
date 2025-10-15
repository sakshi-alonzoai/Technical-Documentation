# Documentation for `test_pgsql_adapter.py`

This document provides a comprehensive technical overview and line-by-line explanation of the Python script `tests/app/db/test_pgsql_adapter.py`.

---

## Technical Documentation: `tests/app/db/test_pgsql_adapter.py`

### 1. High-Level Overview

This Python script is a suite of unit tests for the `PGSQLAdapter` class, located at `app.db.pgsql_adapter`. The `PGSQLAdapter` is presumably a module responsible for interacting with a PostgreSQL database.

The primary purpose of this test file is to:
*   Verify the correct behavior of the `PGSQLAdapter`'s `execute_query` method under various conditions, including successful data retrieval and specific operational modes (like `aql_only`).
*   Verify the error handling or default behavior of `PGSQLAdapter`'s `fetch_stat_mapping` method when required configuration files are not found.

The tests are written using the `pytest` framework and extensively utilize `unittest.mock` to isolate the `PGSQLAdapter` from actual database connections, file system operations, and other external dependencies, ensuring that only the logic within `PGSQLAdapter` itself is being tested.

### 2. Imports

```python
import pytest
from unittest.mock import MagicMock, patch
from app.db.pgsql_adapter import PGSQLAdapter
```

*   `import pytest`: Imports the `pytest` testing framework. `pytest` is used for test discovery, running tests, and providing features like fixtures and assertion rewriting.
*   `from unittest.mock import MagicMock, patch`: Imports `MagicMock` and `patch` from Python's standard library `unittest.mock` module.
    *   `MagicMock`: A flexible mock object that can stand in for real objects, automatically creating attributes and methods as they are accessed, and recording calls made to them.
    *   `patch`: A decorator or context manager used to temporarily replace objects in a module with mock objects during tests.
*   `from app.db.pgsql_adapter import PGSQLAdapter`: Imports the `PGSQLAdapter` class from the `app.db.pgsql_adapter` module. This is the "System Under Test" (SUT) – the class whose behavior is being verified.

### 3. Fixtures

```python
@pytest.fixture
def mock_pgsql_adapter():
    with patch("app.db.pgsql_adapter.psycopg2.connect") as mock_connect:
        mock_conn = MagicMock()
        mock_cursor = MagicMock()
        mock_connect.return_value = mock_conn
        mock_conn.cursor.return_value = mock_cursor
        adapter = PGSQLAdapter()
        adapter.conn = mock_conn
        adapter.cursor = mock_cursor
        yield adapter
```

*   `@pytest.fixture`: This decorator marks the `mock_pgsql_adapter` function as a `pytest` fixture. Fixtures are used to set up a baseline for tests, providing data or initialized objects to test functions. `pytest` automatically discovers and provides fixtures to test functions that request them by name.
*   `def mock_pgsql_adapter():`: Defines the fixture function.
*   `with patch("app.db.pgsql_adapter.psycopg2.connect") as mock_connect:`: This is a crucial line for test isolation. It uses `patch` as a context manager to replace the `connect` function within the `psycopg2` module (which `PGSQLAdapter` is expected to use for database connections) with a `MagicMock` object named `mock_connect`. This prevents `PGSQLAdapter` from attempting to establish a real database connection during tests. The `patch` is active only within this `with` block.
*   `mock_conn = MagicMock()`: Creates a `MagicMock` object to simulate a database connection object. This mock will stand in for the actual `psycopg2.Connection` object.
*   `mock_cursor = MagicMock()`: Creates a `MagicMock` object to simulate a database cursor object. This mock will stand in for the actual `psycopg2.Cursor` object.
*   `mock_connect.return_value = mock_conn`: Configures the `mock_connect` (which is replacing `psycopg2.connect`). When `PGSQLAdapter` calls `psycopg2.connect()`, it will receive `mock_conn` instead of a real connection object.
*   `mock_conn.cursor.return_value = mock_cursor`: Configures the `mock_conn`. When `PGSQLAdapter` calls `mock_conn.cursor()`, it will receive `mock_cursor` instead of a real cursor object.
*   `adapter = PGSQLAdapter()`: Instantiates the `PGSQLAdapter` class. During its initialization, if `PGSQLAdapter` attempts to connect to a database, it will interact with `mock_connect` and `mock_conn`, ensuring no real database interaction occurs.
*   `adapter.conn = mock_conn`: Explicitly sets the `conn` attribute of the `adapter` instance to the `mock_conn`. This ensures that any subsequent operations on `adapter.conn` within the `PGSQLAdapter` instance will interact with the mocked connection.
*   `adapter.cursor = mock_cursor`: Explicitly sets the `cursor` attribute of the `adapter` instance to the `mock_cursor`. This ensures any subsequent operations on `adapter.cursor` within the `PGSQLAdapter` instance will interact with the mocked cursor.
*   `yield adapter`: The `yield` keyword makes this a `pytest` fixture that provides the `adapter` instance. Test functions that request `mock_pgsql_adapter` as an argument will receive this fully configured mocked `PGSQLAdapter` instance. After a test finishes, any cleanup code (if present after `yield`) would run, and the `patch` would be reverted.

### 4. Test Functions

#### 4.1. `test_execute_query_success`

```python
@patch("app.db.pgsql_adapter.MappingsHandler")
def test_execute_query_success(mock_mappings, mock_pgsql_adapter):
    # Setup mock values
    mock_pgsql_adapter.cursor.fetchall.return_value = [
        (1, "John", "TeamX", "QB", "2022-10-01", "TeamY", "W", 120, 2)
    ]
    mock_pgsql_adapter.cursor.description = [
        ("id",), ("playername",), ("teamname",), ("position",), ("gamedate",),
        ("opponentteamname",), ("gameresult",), ("rushing_yards",), ("touchdowns",)
    ]
    mock_pgsql_adapter.cursor.fetchone.return_value = [1]

    mock_mappings().get_metadata_mapping.return_value = {
        "playername": "PLAYER", "teamname": "TEAM", "position": "POS"
    }

    # Patch stat_mapping
    mock_pgsql_adapter.fetch_stat_mapping = MagicMock(return_value={
        "rushing_yards": "RUSH_YDS", "touchdowns": "TDS"
    })

    sql = "SELECT * FROM table;"
    count_sql = "SELECT COUNT(*) FROM table;"
    selected_cols = ["playername", "teamname"]
    stat_cols = ["rushing_yards", "touchdowns"]

    results, count = mock_pgsql_adapter.execute_query(
        sql, count_sql, selected_cols, "MFB", "Player", stat_cols
    )

    assert count == 1
    assert isinstance(results, list)
    assert "metadata" in results[0]
    assert "stats" in results[0]
    assert results[0]["stats"]["RUSH_YDS"] == 120
    assert results[0]["stats"]["TDS"] == 2
```

This test case verifies the successful execution of `PGSQLAdapter.execute_query` by mocking all its external dependencies and asserting the expected structured output.

*   `@patch("app.db.pgsql_adapter.MappingsHandler")`: This decorator patches the `MappingsHandler` class (assumed to be imported or defined within `app.db.pgsql_adapter`) with a `MagicMock` for the duration of this test. The mocked class is passed as `mock_mappings` to the test function.
*   `def test_execute_query_success(mock_mappings, mock_pgsql_adapter):`: Defines the test function. It receives `mock_mappings` from the decorator and `mock_pgsql_adapter` from the `pytest` fixture.
*   **Setup mock values for `cursor`:**
    *   `mock_pgsql_adapter.cursor.fetchall.return_value = [...]`: Configures the mocked cursor's `fetchall()` method to return a list of tuples, simulating the raw rows retrieved from a database query.
    *   `mock_pgsql_adapter.cursor.description = [...]`: Configures the mocked cursor's `description` attribute. This is crucial for `PGSQLAdapter` to map the raw tuple data to dictionary keys based on column names. It simulates `psycopg2`'s column description.
    *   `mock_pgsql_adapter.cursor.fetchone.return_value = [1]`: Configures the mocked cursor's `fetchone()` method to return `[1]`, simulating the result of a `COUNT(*)` query.
*   `mock_mappings().get_metadata_mapping.return_value = {...}`: Configures the mocked `MappingsHandler`. When `MappingsHandler()` is instantiated (`mock_mappings()`) and its `get_metadata_mapping()` method is called, it will return a dictionary that maps internal database column names (`playername`, `teamname`, `position`) to more user-friendly external names (`PLAYER`, `TEAM`, `POS`).
*   `mock_pgsql_adapter.fetch_stat_mapping = MagicMock(return_value={...})`: This line directly replaces the `fetch_stat_mapping` method on the `mock_pgsql_adapter` instance with a `MagicMock`. This mock is configured to return a dictionary that maps internal statistical column names (`rushing_yards`, `touchdowns`) to external names (`RUSH_YDS`, `TDS`). This is done to isolate `execute_query`'s logic from the actual implementation details of `fetch_stat_mapping` during this test.
*   `sql = "SELECT * FROM table;"`: Defines a sample SQL query string that `execute_query` would theoretically execute.
*   `count_sql = "SELECT COUNT(*) FROM table;"`: Defines a sample SQL query string for counting rows.
*   `selected_cols = ["playername", "teamname"]`: A list of column names that are expected to be treated as "metadata" and undergo mapping.
*   `stat_cols = ["rushing_yards", "touchdowns"]`: A list of column names that are expected to be treated as "stats" and undergo mapping.
*   `results, count = mock_pgsql_adapter.execute_query(...)`: Calls the `execute_query` method of the `PGSQLAdapter` instance with the defined SQL, count SQL, column lists, and additional parameters (`"MFB"`, `"Player"`).
*   **Assertions:**
    *   `assert count == 1`: Verifies that the returned `count` matches the mocked `fetchone` result.
    *   `assert isinstance(results, list)`: Ensures the primary results are returned as a list.
    *   `assert "metadata" in results[0]`: Checks if the first result dictionary contains a `metadata` key.
    *   `assert "stats" in results[0]`: Checks if the first result dictionary contains a `stats` key.
    *   `assert results[0]["stats"]["RUSH_YDS"] == 120`: Verifies that the "rushing_yards" column (mapped to "RUSH_YDS") is correctly extracted and its value matches the mock data.
    *   `assert results[0]["stats"]["TDS"] == 2`: Verifies that the "touchdowns" column (mapped to "TDS") is correctly extracted and its value matches the mock data.

#### 4.2. `test_execute_query_aql_only`

```python
def test_execute_query_aql_only(mock_pgsql_adapter):
    results, count = mock_pgsql_adapter.execute_query(
        "SQL", "COUNT", [], "MFB", "Player", [], aql_only=True
    )
    assert results == []
    assert count == 0
```

This test case verifies the behavior of `PGSQLAdapter.execute_query` when the `aql_only` parameter is set to `True`, which presumably indicates that no actual database queries should be executed for data retrieval.

*   `def test_execute_query_aql_only(mock_pgsql_adapter):`: Defines the test function, receiving the `mock_pgsql_adapter` from the fixture.
*   `results, count = mock_pgsql_adapter.execute_query(...)`: Calls `execute_query` with dummy SQL strings and empty column lists, critically passing `aql_only=True`.
*   `assert results == []`: Asserts that the `results` list is empty. This is the expected behavior when `aql_only` is true, as no data should be fetched from the database.
*   `assert count == 0`: Asserts that the `count` is zero, consistent with no data being retrieved.

#### 4.3. `test_fetch_stat_mapping_file_not_found`

```python
@patch("app.db.pgsql_adapter.os.path.exists", return_value=False)
def test_fetch_stat_mapping_file_not_found(_, mock_pgsql_adapter):
    result = mock_pgsql_adapter.fetch_stat_mapping("MFB", "Player")
    assert result == {}
```

This test case verifies the behavior of `PGSQLAdapter.fetch_stat_mapping` when a specific configuration file (presumably for stat mappings) cannot be found on the file system.

*   `@patch("app.db.pgsql_adapter.os.path.exists", return_value=False)`: This decorator patches the `os.path.exists` function (which `PGSQLAdapter` or a dependency is expected to use to check if a file exists) with a `MagicMock` that always returns `False`. This simulates the scenario where the mapping file is missing. The `_` argument signifies that the mock object itself is not used directly within the test function.
*   `def test_fetch_stat_mapping_file_not_found(_, mock_pgsql_adapter):`: Defines the test function, receiving the unused mock (`_`) and the `mock_pgsql_adapter`.
*   `result = mock_pgsql_adapter.fetch_stat_mapping("MFB", "Player")`: Calls the `fetch_stat_mapping` method on the `mock_pgsql_adapter` with sample `game_type` and `player_type` parameters.
*   `assert result == {}`: Asserts that the `result` returned by `fetch_stat_mapping` is an empty dictionary. This confirms the expected behavior: if the mapping file isn't found, the method should return an empty mapping rather than raising an error or returning incomplete data.

### 5. Main Execution Flow (`__main__` block)

This script does not contain an `if __name__ == "__main__":` block. It is designed to be run as part of a test suite using `pytest`. When `pytest` is executed (e.g., `pytest tests/app/db/test_pgsql_adapter.py` or just `pytest` from the project root), it automatically discovers and executes all functions prefixed with `test_` within this file.

### 6. Important Implementation Details, Dependencies, or Configuration Variables

*   **Testing Framework**: `pytest` is the cornerstone of this test suite, providing the structure and execution environment.
*   **Mocking**: `unittest.mock` (specifically `MagicMock` and `patch`) is extensively used to create isolated testing environments by simulating external dependencies like database connections (`psycopg2.connect`), file system checks (`os.path.exists`), and other helper classes (`MappingsHandler`).
*   **System Under Test (SUT)**: The `PGSQLAdapter` class is the main component being tested. Its `execute_query` and `fetch_stat_mapping` methods are specifically targeted.
*   **Database Interaction Abstraction**: The `PGSQLAdapter` is designed to abstract database interactions. The tests imply it uses `psycopg2` for PostgreSQL connectivity.
*   **Mapping Logic**: The tests reveal that `PGSQLAdapter` (or its internal components) handles complex data mapping:
    *   **Metadata Mapping**: Appears to use a `MappingsHandler` to map internal database column names to external, user-friendly "metadata" names (e.g., `playername` to `PLAYER`).
    *   **Stat Mapping**: Appears to have a `fetch_stat_mapping` method that looks for configuration files (based on `game_type` and `player_type`) to map internal "stat" column names to external names (e.g., `rushing_yards` to `RUSH_YDS`).
*   **Structured Output**: `execute_query` is expected to return results as a list of dictionaries, where each dictionary contains nested dictionaries for "metadata" and "stats", using the mapped column names.
*   **AQL Only Mode**: The `aql_only=True` parameter in `execute_query` suggests an optimization or alternative mode where actual database queries are bypassed for data retrieval, perhaps for schema validation or specific API-only operations.
