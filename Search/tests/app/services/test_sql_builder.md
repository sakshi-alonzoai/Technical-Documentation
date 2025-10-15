# Documentation for `test_sql_builder.py`

This document provides a comprehensive technical overview and detailed explanation of the provided Python script, which implements unit tests for the `SQLBuilder` service class.

---

## Technical Documentation: `TestSQLBuilder` Unit Tests

### 1. High-Level Overview

This Python script `test_sql_builder.py` serves as a suite of unit tests for the `SQLBuilder` class, located in `app.services.sql_builder`. The `SQLBuilder` class is responsible for converting an "Analytics Query Language" (AQL) output into actual SQL queries suitable for a database.

The tests ensure that:
*   The `SQLBuilder` correctly constructs various types of SQL queries (e.g., `BASIC`, `MIN`).
*   It properly handles different components of a query, such as `WHERE` clauses (conditions, qualifiers, filters), selected columns, and table names.
*   It integrates correctly with its internal dependencies (mocked using `unittest.mock`) to retrieve necessary metadata like table names, field mappings, and SQL templates.
*   It raises appropriate errors for invalid inputs or unsupported query configurations.

The script uses the `unittest` framework and `MagicMock` for isolating the `SQLBuilder` logic from its external dependencies.

### 2. Module Imports and Dependencies

The script begins by importing necessary modules and classes:

*   **`unittest`**: The standard Python library for creating and running unit tests.
*   **`unittest.mock.MagicMock`**: A class used to create mock objects for testing. These mocks simulate the behavior of real objects without their actual implementation, allowing tests to focus on specific units of code in isolation.
*   **`unittest.mock.patch`**: A decorator/context manager that allows for patching classes or objects within the scope of a test. Although imported, `patch` is not explicitly used in this particular script, but `MagicMock` is used directly.
*   **`app.services.sql_builder.SQLBuilder`**: The core class under test. This class is expected to contain the logic for generating SQL queries.
*   **`app.constants.QueryTypes`**: An enumeration (likely) defining various types of queries that `SQLBuilder` can handle (e.g., `BASIC`, `MIN`).
*   **`app.constants.SortingFields`**: An enumeration (likely) defining specific keys used within the AQL output, such as `CONDITIONS` and `QUALIFIERS`.

### 3. Class: `TestSQLBuilder(unittest.TestCase)`

This class defines a collection of test methods for the `SQLBuilder` functionality. It inherits from `unittest.TestCase`, which provides a rich set of assertion methods for verifying test outcomes.

#### 3.1. `setUp(self)`

*   **Purpose**: This method is part of the `unittest` lifecycle. It runs before each test method in the `TestSQLBuilder` class. Its primary role is to set up a clean, consistent environment for each test, ensuring test isolation.
*   **Implementation Logic**:
    *   `self.builder = SQLBuilder()`: An instance of the `SQLBuilder` class (the subject under test) is created and stored as `self.builder`.
    *   `self.mock_table_name = "player_game_statistics"`: A string representing a mock table name is defined. This will be used by the mocked dependencies.
    *   `self.mock_template = "SELECT {selected_columns} FROM {table_name} {where_clause} LIMIT {limit} OFFSET {offset};"`: A mock SQL template string is defined. The `SQLBuilder` is expected to use such templates to construct the final SQL query.
    *   **Mocking Dependencies**: The following lines are critical for isolating the `SQLBuilder`'s logic. `SQLBuilder` relies on other services (like `mappings_handler` and `sql_loader`) to get metadata and templates. By using `MagicMock`, these external dependencies are simulated, preventing actual file I/O or complex data lookup during tests.
        *   `self.builder.mappings_handler.get_table_name = MagicMock(return_value=self.mock_table_name)`: Mocks the `get_table_name` method of `self.builder.mappings_handler`. Whenever `get_table_name` is called within the `SQLBuilder` instance, it will immediately return `self.mock_table_name` without executing its original logic.
        *   `self.builder.mappings_handler.get_fields_for_query = MagicMock(return_value=["playername", "teamname"])`: Mocks `get_fields_for_query` to return a predefined list of column names, simulating the fields available for a query.
        *   `self.builder.mappings_handler.load_filter_mappings = MagicMock(return_value={"season": ("season", "=")})`: Mocks `load_filter_mappings` to return a dictionary representing how specific filters (e.g., "season") map to database columns and operators.
        *   `self.builder.sql_loader.load_template = MagicMock(return_value=self.mock_template)`: Mocks `load_template` from `self.builder.sql_loader` to return the predefined `self.mock_template` string.

#### 3.2. `test_convert_aql_to_sql_basic(self)`

*   **Purpose**: Tests the `convert_aql_to_sql` method of `SQLBuilder` for a `QueryTypes.BASIC` query. This method is the primary entry point for converting AQL-like input into SQL.
*   **Implementation Logic**:
    *   `aql_output`: A dictionary representing a simulated AQL output, containing `stat_period`, `CONDITIONS` (a raw SQL condition string), and `QUALIFIERS` (a dictionary of key-value pairs for filtering).
    *   `sql_query, count_query, selected_columns, stat_columns = self.builder.convert_aql_to_sql(...)`: Calls the `convert_aql_to_sql` method with the AQL output, along with mocked `sport_code`, `entity`, `filters`, `query_type`, and `team_code`. It expects four return values: the main SQL query, a count query, a list of selected columns, and a list of stat columns.
    *   **Assertions**:
        *   `self.assertIn("SELECT", sql_query)`: Verifies that the generated SQL query starts with `SELECT`.
        *   `self.assertIn("teamname = 'Alabama'", sql_query)`: Checks if the qualifier `teamname = 'Alabama'` is correctly included in the `WHERE` clause.
        *   `self.assertIn("sPoints", sql_query)`: Ensures the condition `sPoints >= 10` is present in the query.
        *   `self.assertIn("playername", sql_query)`: Checks if the `playername` (from `get_fields_for_query` mock) is included in the query.
        *   `self.assertEqual("playername" in selected_columns, True)`: Confirms `playername` is in the list of `selected_columns`.
        *   `self.assertEqual("sPoints" in stat_columns, True)`: Confirms `sPoints` is in the list of `stat_columns`, indicating it was identified as a statistic.

#### 3.3. `test_build_where_clause_conditions(self)`

*   **Purpose**: Tests the `_build_where_clause` private helper method of `SQLBuilder`. This method is responsible for constructing the `WHERE` clause string from various components.
*   **Implementation Logic**:
    *   `where_clause = self.builder._build_where_clause(...)`: Calls the private method with a condition string, a dictionary of qualifiers, and a dictionary of filters, along with a `QueryTypes.BASIC.value`.
    *   **Assertions**:
        *   `self.assertIn("sPoints >= 5", where_clause)`: Verifies that the raw condition string is included.
        *   `self.assertIn("teamname = 'Texas'", where_clause)`: Checks if the qualifier is correctly formatted and included.
        *   `self.assertIn("season = '2023'", where_clause)`: Checks if the filter (which maps "season" to `season = '2023'` based on `load_filter_mappings` mock) is correctly formatted and included.

#### 3.4. `test_generate_query_sql(self)`

*   **Purpose**: Tests the `_generate_query` private helper method of `SQLBuilder`. This method is expected to take pre-processed components (like selected columns, where clause) and combine them with the SQL template to form the final queries.
*   **Implementation Logic**:
    *   `aql_output`: A minimal AQL output dictionary.
    *   `selected_columns`, `where_clause`: Pre-defined lists/strings representing parts of the query, simulating the output of earlier processing steps.
    *   `sql_query, count_query, _, _ = self.builder._generate_query(...)`: Calls the `_generate_query` method with all necessary parameters including the mocked `table_name`, `selected_columns`, `where_clause`, `aql_output`, `limit`, `offset`, and other context variables. It captures the main `sql_query` and `count_query`.
    *   **Assertions**:
        *   `self.assertIn("SELECT", sql_query)`: Ensures the generated query starts with `SELECT`.
        *   `self.assertIn("FROM player_game_statistics", sql_query)`: Verifies that the correct table name (from `self.mock_table_name`) is used in the `FROM` clause.

#### 3.5. `test_invalid_query_type(self)`

*   **Purpose**: Tests the error handling of `convert_aql_to_sql` when an invalid or unsupported query type is provided.
*   **Implementation Logic**:
    *   `aql_output`: A minimal AQL output.
    *   `with self.assertRaises(ValueError):`: This context manager asserts that the code block within it will raise a `ValueError`. If no `ValueError` is raised, or a different error is raised, the test will fail.
    *   `self.builder.convert_aql_to_sql(..., "invalid_query", ...)`: Calls `convert_aql_to_sql` with an `invalid_query` type string. This is expected to trigger a `ValueError` within `SQLBuilder`.

#### 3.6. `test_missing_stat_field_min_query(self)`

*   **Purpose**: Tests error handling for specific query types, particularly `QueryTypes.MIN`, when required fields (like a stat field in its `CONDITIONS`) are missing. `MIN` queries typically operate on a specific statistic.
*   **Implementation Logic**:
    *   `aql_output`: An AQL output where `CONDITIONS` is an empty string, simulating a scenario where no statistic is specified for a `MIN` query.
    *   `with self.assertRaises(ValueError):`: Again, this context manager asserts that a `ValueError` will be raised within the enclosed block.
    *   `self.builder.convert_aql_to_sql(..., QueryTypes.MIN.value, ...)`: Calls `convert_aql_to_sql` with the `QueryTypes.MIN` value and the `aql_output` lacking a stat condition. This is expected to result in a `ValueError`.

### 4. Main Execution Block

```python
if __name__ == "__main__":
    unittest.main()
```

*   **Purpose**: This standard Python construct ensures that the `unittest.main()` function is called only when the script is executed directly (not when it's imported as a module).
*   **Implementation Logic**:
    *   `unittest.main()`: Discovers and runs all test methods within classes that inherit from `unittest.TestCase` in the current file. It provides command-line interface for running tests, reporting results, and handling test failures.
