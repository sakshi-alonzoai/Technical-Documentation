# Documentation for `match_table_creator.py`

# Match-Specific Table Creator Documentation

## Overview

This Python script automates the creation and management of match-specific tables in a PostgreSQL database.  It leverages PostgreSQL's `CREATE TABLE ... LIKE INCLUDING ALL` command for efficient and reliable table creation, significantly reducing the complexity and potential for errors compared to manual schema copying. The script supports creating, dropping, listing, and verifying the structure of these tables.  It's designed to be robust, handling potential concurrency issues and logging all operations for auditing and debugging.

## Dependencies

- Python 3.7+
- `psycopg2`: PostgreSQL adapter for Python (install with `pip install psycopg2-binary`)
- `typer`: CLI framework (install with `pip install typer`)
- `python-json-logger`:  (While not explicitly listed as a dependency it's used for improved log formatting.  Install with `pip install python-json-logger`)

## Configuration

The script uses a JSON configuration file (default: `environment-variables.json`) to store database connection details. This file should contain the following keys:

- `PG_HOST`: PostgreSQL server hostname or IP address.
- `PG_PORT`: PostgreSQL server port (default: 5432).
- `PG_DATABASE`: Name of the PostgreSQL database.
- `PG_USERNAME`: PostgreSQL username.
- `PG_PASSWORD`: PostgreSQL password.

## Classes

### `MatchTableCreator`

This class encapsulates the core functionality of the script.

#### Methods:

- `__init__(self, config_file: str = "environment-variables.json", match_id: str = None, connection_pool=None, config: Optional[Dict] = None)`: Constructor. Loads configuration, initializes logging, and optionally accepts a connection pool and a configuration dictionary.

- `_load_config(self, config_file: str) -> Dict[str, Any]`: Loads the database configuration from the specified JSON file.  Handles file not found and JSON parsing errors gracefully.

- `_setup_logging(self)`: Configures logging to write to both a timestamped log file and the console.  Creates the necessary log directory structure.

- `_connect_to_database(self)`: Establishes a connection to the PostgreSQL database using the loaded configuration. Uses a connection pool if provided.  Handles connection errors with detailed logging and exception handling.

- `_disconnect_from_database(self)`: Closes the database connection, returning the connection to the pool if it was used.

- `_table_exists(self, table_name: str) -> bool`: Checks if a table exists in the database.

- `_get_table_stats(self, table_name: str) -> Dict[str, int]`: Retrieves basic table statistics (column count, index count, constraint count).

- `_log_table_info(self, table_name: str)`: Logs detailed information about a table after creation.

- `_create_insert_function(self, source_table: str, target_table: str)`: Creates a match-specific insert function by copying and modifying a base function from the source table. It uses regex to safely modify the function definition, replacing the source table name with the target table name.  Handles cases where the source function doesn't exist.

- `create_match_table(self, source_table: str, drop_if_exists: bool = False, copy_functions: bool = True) -> str`: Creates a match-specific table by copying the schema and indexes from a source table using `CREATE TABLE ... LIKE INCLUDING ALL`.  It uses advisory locks to prevent concurrent creation conflicts.  Handles errors, including existing tables and lock failures, with detailed logging and exception handling. The function also recreates per-column indexes, improving performance.

- `create_match_tables(self, table_list: List[str], drop_if_exists: bool = False, copy_functions: bool = True) -> List[str]`: Creates multiple match-specific tables.  It iterates through the list of tables, calling `create_match_table` for each one. It reports success and failures separately, making the function more resilient and providing better feedback to the user.

- `drop_match_tables(self, table_list: List[str]) -> List[str]`: Drops match-specific tables.

- `list_match_tables(self, table_pattern: str = None) -> List[str]`: Lists existing match-specific tables matching an optional pattern.

- `verify_table_structure(self, source_table: str, target_table: str) -> Dict[str, Any]`: Verifies the structure of a created match-specific table by comparing its statistics with the source table.


## Functions

### `run_cli(...)`:

This function is the main entry point for the command-line interface (CLI) using `typer`. It parses CLI arguments, handles orchestrator-specific input formats, and calls the appropriate methods of the `MatchTableCreator` class.

### API wrapper functions (`create_match_tables_api`, `drop_match_tables_api`, `list_match_tables_api`, `verify_table_structure_api`):

These functions provide a structured API for interacting with the table creator, returning dictionaries containing success status, results, and error messages.  This makes it suitable for integration into other applications or workflows.

## Command-Line Interface (CLI)

The script provides a CLI using `typer`. The following commands are available:

- `create`: Creates match-specific tables.
- `drop`: Drops match-specific tables.
- `list`: Lists existing match-specific tables.
- `verify`: Verifies the structure of created tables.

The CLI supports various options, including specifying the match ID, a comma-separated list of tables, configuration file, whether to drop existing tables, and whether to copy insert functions. It also accepts a JSON configuration string for compatibility with orchestrators.  The help message provides clear instructions on usage.

## Logging

The script uses comprehensive logging to record all actions, including successes, warnings, and errors.  Log files are timestamped and stored in a dedicated directory (`logs`), making them easy to find and review.  The log level can be adjusted as needed.


## Error Handling

The script includes detailed error handling, catching exceptions, logging errors, and providing informative messages to the user.  It gracefully handles various scenarios, including database connection issues, table creation failures, and file not found errors.  Transactions are used to ensure atomicity of operations and prevent partial success in the `create_match_table` method.  Advisory locks are used to protect against concurrent write issues during table creation.

##  Concurrency Control

The script uses PostgreSQL advisory locks to prevent race conditions when multiple processes try to create tables from the same source table concurrently. This ensures data consistency and avoids potential schema conflicts.

## Best Practices

-  Use of parameterized queries (`psycopg2.extras.RealDictCursor`) helps prevent SQL injection vulnerabilities.
-  Advisory locks (`pg_advisory_lock`) for better concurrency control.
-  Robust error handling with detailed logging for easier debugging.
-  Clear separation of concerns using classes and functions for better code organization and maintainability.
-  Comprehensive documentation for improved understanding and usability.


This comprehensive documentation provides a complete guide to understanding, using, and maintaining the `match_table_creator` script.  The detailed explanation of the code structure, functionality, and error handling makes it easy for developers to integrate it into their workflows and debug any issues that might arise.

