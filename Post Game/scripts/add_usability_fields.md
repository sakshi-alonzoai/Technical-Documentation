# Documentation for `add_usability_fields.py`

# PostgreSQL Usability Merge Script: Technical Documentation

## 1. Overview

The `usability_merge.py` script facilitates the merging of usability data from a dedicated usability table into a main PostgreSQL table.  This process involves intelligent handling of column creation, default value assignments, and data updates based on specified linking fields. The script is designed for flexibility and robustness, offering configuration options for data types and default values.


## 2. Dependencies

- Python 3.7+
- `psycopg2`: PostgreSQL adapter for Python (install using `pip install psycopg2-binary`)
- `typer`:  For creating command-line interfaces (install using `pip install typer`)
- `json`: Built-in Python library for JSON handling.

The script also requires a PostgreSQL database server running and accessible to the Python environment.  Database connection details are loaded either from an `environment-variables.json` file or from command line parameters.

## 3. Configuration

The script's behavior is controlled via a JSON configuration dictionary.  This dictionary is provided either via the `--config` command-line argument or implicitly through environment variables and CLI arguments. The configuration dictionary must contain the following keys:

- **`main_table` (str):** The name of the main PostgreSQL table where usability data will be merged.
- **`usability_table` (str):** The name of the PostgreSQL table containing the usability data.
- **`linking_fields` (list of str):** A list of column names common to both the `main_table` and `usability_table`. These fields are used to identify matching records during the merge operation.
- **`usability_fields` (list of str):** A list of column names in the `usability_table` representing the usability data to be merged.
- **`field_types` (dict):** A dictionary mapping each `usability_field` to its corresponding PostgreSQL data type (e.g., `"NUMERIC", "TEXT", "INTEGER", "BIGINT", "REAL", "DOUBLE PRECISION", "VARCHAR", "CHAR"`).
- **`defaults` (dict):** A dictionary mapping each `usability_field` to its default value.  This value is used to populate the `usability_field` in the `main_table` for records that do not have a match in the `usability_table`.


## 4. Functions

### 4.1 `load_environment_from_json(filename: str = "environment-variables.json") -> None`

Loads PostgreSQL connection parameters from a JSON file (`environment-variables.json` by default), setting them as environment variables.  Handles `FileNotFoundError` and `json.JSONDecodeError` exceptions.  The file should contain key-value pairs for `PG_HOST`, `PG_PORT`, `PG_DATABASE`, `PG_USERNAME`, and `PG_PASSWORD`.


### 4.2 `get_db_connection(connection_pool=None, config=None)`

Establishes a connection to the PostgreSQL database.  It prioritizes a provided `connection_pool` (for connection pooling). If no pool is provided it uses a configuration `config` dictionary; falling back to environment variables if neither is available.  Raises `ConnectionError` if the connection fails.


### 4.3 `return_db_connection(conn, connection_pool=None)`

Returns the database connection to the connection pool or closes the connection if a pool wasn't used.


### 4.4 `validate_table_exists(cursor, table_name: str, schema: str = "public") -> bool`

Checks if a table exists in the specified schema within the database using a SQL query.


### 4.5 `get_table_columns(cursor, table_name: str, schema: str = "public") -> List[str]`

Retrieves a list of column names from a specified table and schema.


### 4.6 `column_exists(cursor, table_name: str, column_name: str, schema: str = "public") -> bool`

Verifies the existence of a specific column within a table and schema.


### 4.7 `get_table_count(cursor, table_name: str, schema: str = "public") -> int`

Returns the number of rows in a specified table.


### 4.8 `validate_configuration(config: Dict) -> None`

Validates the provided configuration dictionary, ensuring all required keys are present and that data types and default values are correctly specified.  Raises `ValueError` for any validation errors.


### 4.9 `create_missing_columns(cursor, table_name: str, usability_fields: List[str], field_types: Dict[str, str], defaults: Dict, schema: str = "public") -> List[str]`

Creates any missing columns in the `main_table` based on the `usability_fields`, `field_types`, and `defaults` from the configuration. It handles different data types and sets default values appropriately.


### 4.10 `perform_usability_merge(cursor, config: Dict, schema: str = "public") -> Dict[str, int]`

Performs the core usability merge operation.  It first sets all relevant columns in the `main_table` to their default values and then updates those records that match based on the `linking_fields`. It returns a dictionary containing statistics about the merge process.


### 4.11 `usability_merge_main(config: Dict, connection_pool=None, db_config=None) -> None`

The main function that orchestrates the entire merge process. Handles database connection, configuration validation, column creation, and data merging.  Prints informative messages to the console during execution.


### 4.12 `merge_usability_data_api(config: Dict, connection_pool=None, db_config=None) -> Dict[str, Any]`

An API-friendly version of `usability_merge_main`. This function returns a dictionary containing success/failure status, statistics, and any error messages, suitable for use in API-driven workflows.


### 4.13 `run_cli` and `main_callback`

Typer functions that handle command line argument parsing and execution, including support for the `CONFIG=` argument commonly used in orchestration systems.

## 5. Usage

The script can be run from the command line using Typer:

```bash
python usability_merge.py --config '{"main_table": "your_main_table", "usability_table": "your_usability_table", "linking_fields": ["id1", "id2"], "usability_fields": ["fieldA", "fieldB"], "field_types": {"fieldA": "NUMERIC", "fieldB": "TEXT"}, "defaults": {"fieldA": 0, "fieldB": "N/A"}}'
```

or by setting environment variables and using command-line arguments (less preferred):

```bash
export PG_HOST=your_db_host
export PG_PORT=5432
export PG_DATABASE=your_db_name
export PG_USERNAME=your_db_user
export PG_PASSWORD=your_db_password
python usability_merge.py --main-table your_main_table --usability-table your_usability_table --linking-fields id1,id2 --usability-fields fieldA,fieldB #This will fail due to missing parameters.
```

Remember to replace the placeholder values with your actual database credentials and table/field names.  The `environment-variables.json` file is the recommended way to manage database credentials outside of the script itself.


## 6. Error Handling

The script includes comprehensive error handling for:

- Invalid JSON configuration
- Missing environment variables
- Database connection failures
- Table/column validation errors
- Data type validation errors

Error messages are printed to the console, and appropriate exceptions are raised where necessary, allowing for robust error management in various execution contexts.  The `merge_usability_data_api` function provides structured error reporting suitable for API applications.


## 7.  Logging

The script currently uses `typer.echo` for outputting messages to the console.  For more robust logging in production environments, consider integrating a logging library like Python's built-in `logging` module to record detailed information about the script's execution, including errors and warnings.  This would improve monitoring and debugging capabilities.

