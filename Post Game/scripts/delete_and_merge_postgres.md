# Documentation for `delete_and_merge_postgres.py`

# Enhanced PostgreSQL Atomic Delete and Merge Script - Technical Documentation

## 1. Overview

This Python script provides an enhanced mechanism for performing atomic delete and merge operations on PostgreSQL tables.  It addresses limitations of simpler approaches by:

* **Handling Generated/Computed Columns:** Automatically detects and excludes generated (computed) columns and identity columns in PostgreSQL from merge operations, preventing conflicts.
* **Configurable Exclusions:** Allows specifying custom sets of columns to exclude from the merge process via command-line parameters or a JSON configuration file.
* **Transaction Safety:**  Ensures atomicity using database transactions.  All delete and merge operations are executed within a single transaction; if any part fails, the entire operation is rolled back.
* **Flexible Merge Modes:** Offers a "simple" mode for faster merging without deduplication checks, and a "standard" mode with deduplication to prevent duplicate entries.
* **Dry Run Capability:**  Allows testing the script's behavior without making any actual changes to the database.
* **Merge-Only Mode:** A new feature that allows skipping the delete step entirely, performing a merge-only operation.
* **Connection Flexibility:** Supports various connection methods, including environment variables, connection strings, configuration dictionaries and connection pools.
* **Robust Error Handling:** Includes comprehensive error handling and informative logging to aid debugging.
* **CLI and API:** Provides both a command-line interface (CLI) and an application programming interface (API) for use in various contexts.


## 2.  Dependencies

The script relies on the following Python libraries:

* `psycopg2`:  For interacting with PostgreSQL databases.
* `typer`: For creating a user-friendly command-line interface.
* `json`: For handling JSON configuration data.
* `logging`: For logging messages and debugging information.
* `os`: For interacting with the operating system's environment variables and file system.
* `pathlib`: For easier file path manipulation.
* `datetime`: For timestamping log messages.


These libraries can be installed using pip:  `pip install psycopg2 typer`


## 3.  Functions and Classes

### 3.1 `get_default_excluded_columns()`

Returns a set containing the default column names to exclude (`id`, `created_at`, `updated_at`).  These are typically columns managed automatically by the database.


### 3.2 `is_excluded_column()`

Checks if a given column name should be excluded based on a provided set of excluded column names, performing a case-insensitive comparison.


### 3.3 `get_auto_excluded_columns()`

Automatically identifies and returns a set of column names that should be excluded from merge operations. This includes:

* **Generated Columns (computed columns):** Columns defined using `GENERATED ALWAYS AS` expressions in PostgreSQL 12+.
* **Identity Columns:** Columns defined as identity columns in PostgreSQL 10+.
* **Serial Columns:** Columns with `SERIAL` or `BIGSERIAL` data types (automatically assigned sequence values).


### 3.4 `load_environment_from_json()`

Loads environment variables from a JSON file named `environment-variables.json` located in the same directory as the script.  This provides a structured way to manage database connection credentials.  Handles `FileNotFoundError` and `json.JSONDecodeError`.


### 3.5 `get_db_connection()`

Establishes a connection to the PostgreSQL database. It retrieves the necessary connection parameters from environment variables (`PG_HOST`, `PG_DATABASE`, `PG_USERNAME`, `PG_PASSWORD`, `PG_PORT`).  Raises a `ValueError` if any required variable is missing and a `ConnectionError` if the connection attempt fails.  Crucially, it sets `conn.autocommit = False` to ensure that transactions are managed explicitly.


### 3.6 `validate_table_name()`

Performs basic validation of a table name to prevent potential SQL injection vulnerabilities.  Allows only alphanumeric characters, underscores, and dots, adhering to standard PostgreSQL naming conventions.


### 3.7 `validate_table_exists()`

Verifies that a specified table exists within a given database schema using a SQL query.


### 3.8 `get_table_columns()`

Retrieves and returns a list of column names for a given table, ordered by their ordinal position.


### 3.9 `get_table_count()`

Returns the number of rows in a specified table using `COUNT(*)`.  Uses parameterized queries to prevent SQL injection.


### 3.10 `get_filtered_columns()`

Combines the results of the `get_table_columns`, `get_default_excluded_columns`, and `get_auto_excluded_columns` to return a list of column names *after* excluding those specified (either explicitly or automatically).


### 3.11 `validate_table_compatibility()`

Checks whether multiple tables are structurally compatible for merge operations. It ensures that all source tables contain at least the necessary columns in the main table (allowing for some columns to be missing in the source, which are treated as `NULL` during the merge).  It also performs case-insensitive column name comparisons and provides detailed error messages if incompatibilities are found.


### 3.12 `AtomicDeleteAndMerge` Class

This class encapsulates the logic for atomic delete and merge operations.

*   `__init__`:  The constructor allows initializing the class with various methods to provide the database connection (either a connection string, a `psycopg2` connection pool, or a configuration dictionary). It also disables `autocommit` on the connection.
*   `_get_connection`: Retrieves a database connection using the configured method. Handles potential connection errors gracefully.
*   `_return_connection`: Returns the connection to the connection pool if one is used. Otherwise, it closes the connection.
*   `delete_and_merge_atomic`: The core method that performs the atomic delete and merge operation.  Includes detailed logging and dry-run support.  The `skip_delete` parameter enables a merge-only mode, allowing more flexibility in how the script is used. It manages the transaction, and in case of any error during the process, it rolls back the transaction to maintain data consistency.


### 3.13 `get_connection_params()`

Retrieves database connection parameters. Prioritizes a provided connection pool or config, then a connection string environment variable, falling back to individual environment variables or using default values.


### 3.14 `delete_and_merge_atomic_api()`

Serves as an API function, allowing external systems to call the delete and merge functionality. It wraps the core logic of `AtomicDeleteAndMerge.delete_and_merge_atomic` with additional error handling, input validation, and a standardized return format useful for API responses.  It also handles various connection options.


### 3.15 `run_cli()`

The Typer command-line interface function. It parses the command-line arguments, validates the input, displays the configuration, and calls the `delete_and_merge_atomic_api` function to perform the operation. It provides detailed output to the console, including success messages, error messages, dry-run previews, and execution statistics. The function first prioritizes config parameters over CLI parameters in case of conflicts.


### 3.16 `main_callback()`

The Typer callback function that handles the invocation of the script without any subcommand. It handles the `config` argument for compatibility with orchestrators.


## 4.  Usage

### 4.1 Command-Line Interface (CLI)

The script can be run from the command line using the `typer` framework:

```bash
python your_script_name.py --help  # Show help information
python your_script_name.py --main-table my_main_table --add-entries-from table1,table2 --delete-condition "id > 10"
python your_script_name.py --config '{"main_table": "my_main_table", "add_entries_from": ["table1", "table2"], "delete_condition": "id > 10", "simple_mode": true}'
python your_script_name.py --main-table my_main_table --add-entries-from table1,table2 --no-delete
```

### 4.2  Application Programming Interface (API)

The `delete_and_merge_atomic_api` function can be used directly from other Python scripts:


```python
from your_script_name import delete_and_merge_atomic_api

result = delete_and_merge_atomic_api(
    main_table="my_main_table",
    add_entries_from=["table1", "table2"],
    delete_condition="id > 10",
    connection_params={"host": "localhost", "database": "mydatabase", "user": "myuser", "password": "mypassword"},
    dry_run=True  #optional dry-run
)

if result["success"]:
    print(f"Operation successful: {result['message']}")
    print(f"Records deleted: {result['records_deleted']}")
    print(f"Records inserted: {result['records_inserted']}")
else:
    print(f"Operation failed: {result['message']}")
    print(f"Error: {result['error']}")

```

Remember to replace placeholders like table names, conditions, and connection parameters with your actual values.  The API function allows for a high level of control and customisation.  You can use it with or without a connection pool or a configuration dictionary for more flexibility.  The script handles various error conditions and returns informative error messages for debugging purposes.

## 5.  Logging

The script uses the `logging` module to provide detailed information about its progress and any encountered errors. The log level can be adjusted by modifying the `logging.basicConfig` call at the beginning of the script.


## 6. Error Handling

The script incorporates extensive error handling throughout its functions and classes.  It gracefully handles exceptions such as `FileNotFoundError`, `json.JSONDecodeError`, `psycopg2.Error`, `ValueError`, and others.  Detailed error messages are logged, and in the case of database operations, transactions are rolled back to maintain data consistency. The `delete_and_merge_atomic_api` function returns standardized JSON responses including both success and error indicators.  The CLI version provides informative messages to the console.


## 7.  Future Enhancements

*   More sophisticated table schema validation.
*   Support for different database systems (beyond PostgreSQL).
*   Improved performance optimization for very large datasets.
*   Integration with data profiling tools to provide more information before starting the process.


This comprehensive documentation provides a thorough understanding of the script's functionality, usage, and internal workings.  The inclusion of both API and CLI makes it versatile and suitable for a wide array of integration scenarios.

