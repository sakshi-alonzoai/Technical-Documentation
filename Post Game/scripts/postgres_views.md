# Documentation for `postgres_views.py`

# PostgreSQL View Management Script: Technical Documentation

## 1. Overview

This Python script, `postgres_views.py`, facilitates the creation and deletion of PostgreSQL views.  It offers a command-line interface (CLI) for direct execution and also exposes its functionality as an API for integration into other Python applications. Configuration is primarily managed through an `environment-variables.json` file, mirroring the structure used in `delete_from_postgres.py` (presumably a related script).  The script utilizes the `psycopg2` library for PostgreSQL interaction and `typer` for CLI construction.

## 2.  Module Components

### 2.1 `load_environment_variables()`

This function attempts to load PostgreSQL connection parameters from a JSON file named `environment-variables.json`.  The file should contain key-value pairs specifying connection details (e.g., `PG_HOST`, `PG_PORT`, `PG_DATABASE`, `PG_USERNAME`, `PG_PASSWORD`). If the file is missing or unreadable, a warning is logged, and an empty dictionary is returned.  The function prioritizes a `DATABASE_URL` environment variable, if present, for connection details.  Error handling ensures graceful degradation if the file is malformed or inaccessible.

### 2.2 `get_connection_params()`

This function retrieves database connection parameters. It prioritizes `DATABASE_URL` from the environment variables file. If not found it falls back to individual parameters (`PG_HOST`, `PG_PORT`, `PG_DATABASE`, `PG_USERNAME`, `PG_PASSWORD`) read from the same file or, as a last resort, from environment variables directly. The function returns a dictionary containing these connection parameters.

### 2.3 `PostgreSQLViewManager` Class

This class encapsulates the core logic for interacting with PostgreSQL views.

* **`__init__(self, connection_string=None, connection_pool=None, config=None, **kwargs)`:** The constructor initializes the view manager. It accepts a connection string, a connection pool (for connection pooling), a configuration dictionary (presumably loaded from `environment-variables.json`), or individual connection parameters (`**kwargs`).

* **`_get_connection(self)`:** Establishes a connection to the PostgreSQL database. It prioritizes a connection pool; if none is provided, it uses the configuration dictionary or the connection string.  It sets the isolation level to `ISOLATION_LEVEL_AUTOCOMMIT` to ensure each query is executed in its own transaction.  Robust error handling is included for database connection failures.

* **`_return_connection(self, conn)`:** Returns the connection to the pool if a pool is used; otherwise, closes the connection.

* **`validate_identifier(self, identifier: str)`:** This crucial function validates SQL identifiers (view names, table names) to mitigate SQL injection vulnerabilities. It uses a regular expression to allow only alphanumeric characters, underscores, and dots (for schema-qualified names).  Any invalid identifier raises a `ValueError`.

* **`create_view(self, view_name, table_name, where_clause=None, order_by=None, dry_run=False)`:** Creates or replaces a PostgreSQL view.  It constructs the `CREATE OR REPLACE VIEW` statement based on provided parameters, including optional `WHERE` and `ORDER BY` clauses.  It performs input validation using `validate_identifier`.  If `dry_run` is True, it only logs the generated SQL query without executing it.  The function returns a tuple indicating success and the executed SQL query.

* **`drop_view(self, view_name, dry_run=False)`:** Drops a PostgreSQL view if it exists.  It constructs a `DROP VIEW IF EXISTS` statement and handles the execution, similar to `create_view`.  Input validation is also performed.


### 2.4 CLI Commands (`@app.command` Decorators)

The script defines two CLI commands using the `typer` library:

* **`create`:** This command provides a CLI interface for the `create_view` functionality.  It accepts view and table names, optional `WHERE` and `ORDER BY` clauses, a `dry-run` flag, and individual connection parameters as CLI arguments.

* **`drop`:** This command provides a CLI interface for the `drop_view` functionality.  It accepts the view name, a `dry-run` flag, and connection parameters as CLI arguments.

Both CLI commands include verbose logging functionality and handle exceptions gracefully.


### 2.5 API Functions (`create_view`, `drop_view`)

The script exports two functions, `create_view` and `drop_view`,  that provide the core view management functionality as a Python API.  These functions accept similar parameters to their class method counterparts, allowing for programmatic creation and deletion of views.  They support either direct connection parameters, a connection pool, or a configuration dictionary as input.


## 3. Dependencies

* `psycopg2`:  For PostgreSQL database interaction.
* `typer`: For creating the command-line interface.
* `json`: For parsing the JSON configuration file.
* `logging`: For logging messages.
* `os`: For accessing environment variables.
* `sys`: For exiting the script with error codes.
* `pathlib`: For path manipulation.
* `typing`: For type hinting.
* `re`: for regular expression matching in `validate_identifier`.

## 4. Usage

The script can be run from the command line using `typer`:

```bash
python postgres_views.py create --view my_view --table my_table --where "column1 > 10"
python postgres_views.py drop --view my_view
```

The API functions can be imported and used within other Python scripts:

```python
from postgres_views import create_view

success, query = create_view(view_name="my_view", table_name="my_table", where_clause="column1 > 10", host="localhost", database="mydb", user="myuser", password="mypassword")
```

## 5. Error Handling

The script includes comprehensive error handling for database connection issues, invalid input, and other potential exceptions. Error messages are logged, and appropriate exit codes are used for CLI execution.


## 6. Security Considerations

The `validate_identifier` function is crucial for preventing SQL injection.  However, relying solely on input validation is not sufficient for complete security.  Further security measures, such as parameterized queries (which `psycopg2` supports), are recommended for production environments.  Additionally, storing database credentials securely (e.g., using environment variables or a secrets management system) is essential.

