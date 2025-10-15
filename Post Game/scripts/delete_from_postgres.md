# Documentation for `delete_from_postgres.py`

# PostgreSQL Delete Script: Technical Documentation

## Overview

This Python script facilitates the deletion of records from a PostgreSQL database table.  It offers two primary interfaces: a command-line interface (CLI) via the Typer library and a programmatic API for integration into other applications. The script uses the `psycopg2` library for PostgreSQL interaction and handles configuration through an `environment-variables.json` file or environment variables.  Robust error handling and logging are included.

## Dependencies

- `psycopg2`: PostgreSQL adapter for Python.  Install using `pip install psycopg2-binary`.
- `typer`: CLI building library. Install using `pip install typer`.

## Configuration

The script retrieves database connection details from the `environment-variables.json` file (located in the same directory as the script). This file should be in JSON format and contain either a `DATABASE_URL` connection string or individual connection parameters:


**Using a `DATABASE_URL` connection string (recommended):**

```json
{
    "DATABASE_URL": "postgresql://user:password@localhost:5432/database"
}
```

**Using individual connection parameters:**

```json
{
    "PG_HOST": "localhost",
    "PG_PORT": "5432",
    "PG_DATABASE": "mydatabase",
    "PG_USERNAME": "postgres",
    "PG_PASSWORD": "mypassword"
}
```

Alternatively, environment variables  `DATABASE_URL`, `PG_HOST`, `PG_PORT`, `PG_DATABASE`, `PG_USERNAME`, and `PG_PASSWORD` can be used if the `environment-variables.json` file is absent or incorrectly formatted.  CLI arguments override values from both the JSON file and environment variables.

## Classes

### `PostgreSQLDeleter`

This class encapsulates the database interaction logic.

**Methods:**

- `__init__(self, connection_string=None, connection_pool=None, config=None, **kwargs)`: Constructor.  Accepts either a connection string, a connection pool, or individual database parameters (`host`, `port`, `database`, `user`, `password`).  The connection string takes precedence over the other methods. A `config` dictionary can also be provided directly.

- `_get_connection(self) -> psycopg2.extensions.connection`: Establishes a connection to the PostgreSQL database using the provided parameters. It uses a connection pool if one is provided, otherwise it creates a new connection.  Handles `psycopg2.Error` exceptions and logs connection failures.  Sets the isolation level to `ISOLATION_LEVEL_AUTOCOMMIT` to ensure changes are immediately committed.

- `_return_connection(self, conn)`: Returns the connection to the pool (if used), otherwise closes the connection.

- `validate_table_name(self, table_name: str) -> bool`: Performs basic validation of the table name to mitigate SQL injection risks.  Currently, it only allows alphanumeric characters, underscores, and periods.  More robust validation might be needed in a production environment.

- `build_delete_query(self, table_name: str, filter_condition: str) -> str`: Constructs the SQL `DELETE` query, including the `WHERE` clause. Performs basic validation to ensure the `filter_condition` is not empty.  Raises a `ValueError` if the table name or filter condition is invalid.

- `truncate_table(self, table_name: str, dry_run: bool = False) -> Tuple[int, str]`: Truncates the specified table (deletes all records).  If `dry_run` is True, it only returns the query without execution.  It returns a tuple containing the number of affected rows and the executed query.

- `delete_records(self, table_name: str, filter_condition: str, dry_run: bool = False) -> Tuple[int, str]`: Deletes records from the table based on the provided `filter_condition`. If `dry_run` is True, it only returns the query without execution. It returns a tuple containing the number of affected rows and the executed query.



## Functions

### `load_environment_variables() -> Dict[str, Any]`

Loads the database connection parameters from the `environment-variables.json` file.  Handles file I/O errors and JSON parsing errors gracefully, logging warnings and errors accordingly. Returns an empty dictionary if the file is not found or cannot be parsed.


### `get_connection_params() -> Dict[str, Any]`

Retrieves database connection parameters, prioritizing the `DATABASE_URL` connection string (from the environment file or environment variables). If not present, it falls back to individual parameters loaded from the environment file or environment variables.  Returns a dictionary containing the connection parameters.

### `delete_from_postgres(...) -> Tuple[int, str]`

This function provides the API interface for deleting records from PostgreSQL.  It accepts similar parameters to the CLI command but uses keyword arguments for flexibility.


## CLI Commands

### `delete`

The `delete` command provides the CLI interface.  It uses Typer for argument parsing and help generation.


**Arguments:**

- `--table (-t)`: Name of the PostgreSQL table (required).
- `--filter (-f)`: WHERE clause condition for filtering records (optional, required if not using `--truncate`).
- `--truncate`: Truncate the entire table (optional, mutually exclusive with `--filter`).
- `--dry-run`: Perform a dry run (shows the query without execution) (optional).
- `--host`, `--port`, `--database (-db)`, `--user`, `--password`: Individual connection parameters (optional, override values from configuration files).
- `--connection-string`: Full PostgreSQL connection string (optional, overrides other parameters).
- `--verbose (-v)`: Enable verbose logging (optional).

## Error Handling and Logging

The script incorporates comprehensive error handling for database connections, SQL query execution, file I/O, and JSON parsing.  It uses the Python `logging` module to provide informative logs at different severity levels (INFO, DEBUG, WARNING, ERROR).  The logging level can be adjusted using the `--verbose` flag in the CLI.

## Example Usage (CLI)

```bash
# Delete users older than 30
python delete_postgres.py --table users --filter "age > 30"

# Truncate the 'temp_table' table
python delete_postgres.py --table temp_table --truncate

# Dry run to preview the delete query
python delete_postgres.py --table users --filter "status = 'inactive'" --dry-run
```

## Example Usage (API)

```python
from delete_postgres import delete_from_postgres

affected_rows, query = delete_from_postgres(
    table_name="products", 
    filter_condition="price > 100", 
    host="dbhost", 
    database="mydatabase", 
    user="dbuser", 
    password="dbpassword"
)

print(f"Deleted {affected_rows} rows. Query: {query}")

```

