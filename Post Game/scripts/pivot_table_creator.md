# Documentation for `pivot_table_creator.py`

# Technical Documentation: `pivot_table_creator.py`

## Overview

`pivot_table_creator.py` is a Python script designed to populate a pre-existing "tall" (unpivoted) table in a PostgreSQL database from a "wide" source table.  The script efficiently handles the unpivoting process, allowing for filtering by `period_number` and `period_type`. It supports configuration via a JSON configuration file or command-line arguments, offering flexibility in deployment.

## Dependencies

The script relies on several external libraries:

* **`psycopg2`:**  For interacting with PostgreSQL databases.
* **`typer`:** A modern, Pythonic library for building command-line interfaces.
* **`json`:** Python's built-in JSON library for handling configuration files.
* **`logging`:** Python's built-in logging library for recording script activity.
* **`re`:** Python's regular expression library (used for pattern matching in column selection).
* **`pathlib`:** For path manipulation.
* **`typing`:** For type hinting.

These dependencies should be installed using `pip`:  `pip install psycopg2 typer`

## Classes

### `PivotTableCreator`

This class encapsulates the core logic of the script.

**Constructor (`__init__`)**:

* Takes a `config_file` (path to JSON config), an optional `connection_pool` (for connection pooling), and an optional `config` dictionary (for direct configuration).
* Loads the configuration from the specified file or uses the provided `config` dictionary. If both `connection_pool` and `config` are provided, it initializes an empty config dictionary.
* Establishes database connection parameters (`db_config`) from the configuration, including host, port, database name, username, and password.
* Initializes `conn` and `cur` (database connection and cursor) to `None`.  `_borrowed_connection` flags whether the connection was obtained from a pool.

**`_load_config(config_file: str) -> dict`**:

* Loads the JSON configuration from the given `config_file`.
* Handles file existence and JSON decoding errors, exiting the script with an error message if problems occur.

**`_connect()`**:

* Establishes a connection to the PostgreSQL database using the parameters in `db_config`. It uses the provided connection pool if available.
* Logs connection status.

**`_disconnect()`**:

* Closes the database cursor and connection.  Returns the connection to the pool if it was borrowed.
* Logs disconnection status.

**`_get_columns(table: str) -> List[str]`**:

* Retrieves all column names from the specified PostgreSQL table.

**`_get_numeric_columns(table: str, prefix: str = "s_") -> List[str]`**:

* Retrieves only numeric columns from the specified table that match the given prefix (defaulting to "s_").  Uses a regular expression for efficient matching.


**`populate_unpivot(source_table: str, target_table: str, unpivot_columns: Optional[List[str]] = None, period_number: int = 0, period_type: str = "REGULAR")`**:

* This is the main function for unpivoting data.
* Connects to the database (`_connect()`).
* Reads the schema of the target table to identify identifier columns (`id_cols`).
* Determines the columns to unpivot (`stats`). If `unpivot_columns` is provided, it uses those columns; otherwise, it retrieves numeric columns with the "s_" prefix.
* Truncates the target table.
* Executes an `INSERT` statement to populate the unpivoted data. This statement uses a `LATERAL JOIN` for efficient unpivoting, and it filters out rows where the `value` is `NULL` or `0`.
* Commits changes, logs the number of rows inserted, and disconnects from the database.

## Functions

### `populate_unpivot_table(...) -> Dict[str, Any]`

* Acts as an API wrapper for the `PivotTableCreator` class.
* Handles potential exceptions, returning a dictionary indicating success or failure, along with relevant information.
* It also handles the case where `unpivot_columns` is not provided, automatically discovering them before the main operation.


### `run_cli(...)`

* Core command-line interface logic.  Instantiates `PivotTableCreator` and calls `populate_unpivot()`.
* Performs basic validation to ensure that source and target table names are provided.


### `main_command(...)`

* Typer command-line interface function.  Parses command-line arguments, handles JSON configuration (`--config`, `--unpivot-columns`), and calls `run_cli`.

### `main_callback(...)`

* Typer callback function.  Handles the case where the script is called with a single `CONFIG=` argument (a JSON string containing all configuration parameters).  Parses the JSON and dispatches the command to `main_command`.


## Command-Line Interface

The script provides a command-line interface using `typer`.  The primary command is `main_command`, which can be invoked with various options:

* `--source-table (-s)`: The name of the wide source table.
* `--target-table (-t)`: The name of the existing unpivot table.
* `--unpivot-columns (-u)`: A JSON array specifying the columns to unpivot (optional; if omitted, the script discovers numeric columns with an "s_" prefix).
* `--period-number (-p)`:  A filter for the `period_number` column (default: 0).
* `--period-type`: A filter for the `period_type` column (default: "REGULAR").
* `--config-file (-c)`: Path to the JSON configuration file (default: "environment-variables.json").
* `--config`:  An orchestrator-style JSON blob that overrides individual flags.


The script can also be invoked with a single `CONFIG=` argument, which is a JSON string containing all configuration parameters.  This mode simplifies automation.


## Error Handling

The script includes error handling for:

* Missing configuration files.
* Invalid JSON in configuration files or command-line arguments.
* Database connection errors.
* Issues reading table schemas.
*  General exceptions during the unpivoting process.


Error messages are logged using the `logging` module and are also displayed to the user via the command-line interface.  The `populate_unpivot_table` function returns a dictionary containing success/failure status and error messages.


## Logging

The script uses the Python `logging` module to record events such as configuration loading, database connection status, and the number of rows inserted.  The logging level is set to `INFO` by default.


This detailed documentation provides a comprehensive understanding of the `pivot_table_creator.py` script's functionality, implementation, and usage.

