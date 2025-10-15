# Documentation for `merge_postgres_tables.py`

# PostgreSQL Table Merger Script - Technical Documentation

## 1. Overview

The `merge_postgres_tables.py` script merges data from multiple PostgreSQL source tables into a designated main table using SQL `UNION` operations.  It's designed to be robust and flexible, handling potential inconsistencies between source tables and providing options for different merge strategies. The script is intended to be invoked either directly via the command line or indirectly by a workflow orchestrator.

## 2. Dependencies

- Python 3.7+
- `psycopg2`:  For PostgreSQL database interaction.  Install using `pip install psycopg2-binary`
- `typer`: For creating a command-line interface. Install using `pip install typer`

The script also requires an `environment-variables.json` file (see section 4) in the same directory to specify PostgreSQL connection details.


## 3. Functions

### 3.1 `is_excluded_column(column_name: str) -> bool`

Checks if a given column name should be excluded from the merge operation. By default, columns named "id", "created_at", and "updated_at" (case-insensitive) are excluded.

### 3.2 `load_environment_from_json(filename: str = "environment-variables.json") -> None`

Loads PostgreSQL connection credentials from a JSON file (`environment-variables.json` by default). This file should contain key-value pairs for `PG_HOST`, `PG_PORT`, `PG_DATABASE`, `PG_USERNAME`, and `PG_PASSWORD`.  The function raises a `FileNotFoundError` if the file is not found and a `ValueError` if the JSON is invalid.

### 3.3 `get_db_connection(connection_pool=None, config=None) -> psycopg2.connection`

Establishes a connection to the PostgreSQL database using credentials loaded from the environment variables.  It handles potential errors during connection establishment and raises a `ValueError` if required environment variables are missing.  It also accepts optional `connection_pool` and `config` arguments for integration with connection pooling and external configuration.

### 3.4 `get_default_excluded_columns() -> Set[str]`

Returns a set containing the default names of columns to be excluded from the merge process ("id", "created_at", "updated_at").

### 3.5 `get_auto_excluded_columns(cursor, table_name: str, schema: str = "public") -> Set[str]`

This function uses database introspection to automatically identify and exclude columns that are likely to be auto-managed by the database system.  Specifically, it identifies and excludes:
    - Generated columns (computed columns)
    - Identity columns
    - SERIAL/BIGSERIAL columns (auto-incrementing columns)
The function dynamically checks the table schema in PostgreSQL and returns a set of automatically detected excluded column names.


### 3.6 `get_filtered_columns(cursor, table_name: str, schema: str = "public", excluded_columns: Optional[Set[str]] = None, auto_exclude: bool = True) -> List[str]`

Retrieves the list of columns for a given table, applying exclusions specified by the `excluded_columns` parameter (or the defaults if none are provided) and optionally enabling automatic exclusion via `get_auto_excluded_columns` (controlled by `auto_exclude` flag).  It returns a list of column names after applying all exclusion rules.

### 3.7 `is_excluded_column_enhanced(column_name: str, excluded_columns: Set[str]) -> bool`

Enhanced version of `is_excluded_column` which uses a provided set `excluded_columns` for flexibility.


### 3.8 `validate_table_exists(cursor, table_name: str, schema: str = "public") -> bool`

Checks if a table exists in the specified schema within the database.

### 3.9 `get_table_columns(cursor, table_name: str, schema: str = "public") -> list`

Retrieves a list of column names for a given table.

### 3.10 `validate_table_compatibility(cursor, main_table: str, add_entries_from_tables: list, schema: str = "public", excluded_columns: Optional[Set[str]] = None, auto_exclude: bool = True) -> bool`

Validates the compatibility of the main table and source tables for merging. It checks for extra columns in source tables (which are disallowed) and reports missing columns in source tables (which are allowed, filled with NULLs). The function leverages the enhanced exclusion logic.

### 3.11 `get_table_count(cursor, table_name: str, schema: str = "public") -> int`

Retrieves the number of rows in a given table.

### 3.12 `merge_tables_append(cursor, main_table: str, add_entries_from_tables: list, schema: str = "public", excluded_columns: Optional[Set[str]] = None, auto_exclude: bool = True) -> Dict[str, int]`

Performs the main merge operation, appending rows from source tables to the main table. It handles missing columns by setting them to NULL in the target table, and it ensures that there are no extra columns in source tables. This function incorporates deduplication using `NOT EXISTS` clause.

### 3.13 `merge_tables_main(main_table: str, add_entries_from: list, excluded_columns: Optional[List[str]] = None, auto_exclude: bool = True) -> None`

The main function that orchestrates the merge process when called from the command line, using the `merge_tables_append` function for the actual database operations.  It includes error handling and progress reporting.

### 3.14 `merge_tables_simple(cursor, main_table: str, add_entries_from_tables: list, schema: str = "public", excluded_columns: Optional[Set[str]] = None, auto_exclude: bool = True) -> Dict[str, int]`

Similar to `merge_tables_append` but performs a simple append without deduplication.  This is significantly faster but should only be used when duplicates are not a concern.

### 3.15 `merge_tables_main_simple(main_table: str, add_entries_from: list, excluded_columns: Optional[List[str]] = None, auto_exclude: bool = True) -> None`

Orchestrates the simple merge process (no deduplication) using `merge_tables_simple`.

### 3.16 `merge_tables_api(...) -> Dict[str, Any]`

Provides an API interface for the table merging functionality. It accepts a connection pool or configuration dictionary as input, allowing for flexible integration with other applications or services. It returns a dictionary containing the operation status, statistics, and any errors encountered.


## 4. Environment Variables

The script reads PostgreSQL connection parameters from a JSON file named `environment-variables.json` located in the same directory as the script.  This file should have the following structure:

```json
{
  "PG_HOST": "your_db_host",
  "PG_PORT": "5432",
  "PG_DATABASE": "your_db_name",
  "PG_USERNAME": "your_db_user",
  "PG_PASSWORD": "your_db_password"
}
```

## 5. Command-Line Interface (CLI)

The script uses `typer` to provide a command-line interface.  The main command is `run_cli`.  Refer to the script's docstrings for detailed usage instructions and examples.

## 6. Usage

The script can be run from the command line:

```bash
python merge_postgres_tables.py --main-table main_table_name --add-entries-from "table1,table2,table3" [--simple] [--excluded-columns column1,column2] [--auto-exclude/--no-auto-exclude] [--config <json_config>]

#or using short options:
python merge_postgres_tables.py -m main_table_name -a "table1,table2,table3" -s -e id,created_at --no-auto-exclude -c '{"main_table": "my_main_table", "add_entries_from": ["source_table_1", "source_table_2"]}' 
```

The `--simple` flag indicates a simple append (without deduplication).  The `--excluded-columns` allows specifying additional columns to exclude besides the defaults. The `--auto-exclude` (or `--no-auto-exclude`) flag controls automatic detection and exclusion of generated/identity columns. The `--config` option provides a JSON configuration string for external orchestration.

## 7. API Usage

The `merge_tables_api` function provides an API for programmatic usage.  It allows passing a `connection_pool` and other parameters for greater integration flexibility.  See the function docstring for details.

## 8. Error Handling

The script includes comprehensive error handling.  It checks for missing environment variables, invalid JSON, database connection errors, table existence, schema compatibility, and other potential issues.  Error messages are informative and provide guidance on resolving issues.


## 9.  Enhancements

The script incorporates several improvements compared to a basic table merger:

- **Enhanced Exclusion Logic:**  The addition of `get_auto_excluded_columns` and the refined exclusion handling significantly improves the robustness and safety of the merge process, especially for tables with complex schemas and auto-generated columns.
- **Flexibility:** The API and configuration options enable easier integration into larger workflows and systems.
- **Detailed Logging and Reporting:**  Extensive use of `typer.echo` provides clear feedback to the user throughout the process.
- **Clearer Error Messages:**  Improved error handling provides more context to the user.
- **Separation of Concerns:** Functions are well-separated, improving code organization and maintainability.
- **Testability:** The modular design makes it easier to write unit tests for individual functions.
- **Simple Mode:** The addition of a `simple` mode offers a faster option for scenarios where data deduplication is unnecessary.


This comprehensive documentation provides all the information necessary for using, extending, and maintaining the `merge_postgres_tables.py` script.

