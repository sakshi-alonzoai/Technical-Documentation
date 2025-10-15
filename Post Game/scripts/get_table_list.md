# Documentation for `get_table_list.py`

# get_table_list.py Technical Documentation

## Overview

`get_table_list.py` is a Python script designed to retrieve a list of tables from a PostgreSQL database.  It offers robust filtering capabilities, allowing users to select tables based on various criteria such as name patterns, schema, size, creation date, and table type. The script outputs the results in either CSV or JSON format, making it suitable for both interactive use and integration into larger data processing pipelines.

## Dependencies

The script relies on the following Python libraries:

*   `psycopg2`: For connecting to and interacting with PostgreSQL databases.
*   `typer`:  A library for building interactive command-line applications.  Provides the command-line interface.
*   `csv`: For CSV output formatting.
*   `io`: For in-memory string manipulation when generating CSV output.
*   `json`: For JSON output formatting.
*   `sys`: For system-level operations, particularly handling exit codes.


These libraries can be installed using pip:  `pip install psycopg2 typer`


## Configuration

The script loads configuration settings from a JSON file (default: `environment-variables.json`). This file should contain the PostgreSQL connection details:

```json
{
  "PG_HOST": "localhost",
  "PG_PORT": 5432,
  "PG_DATABASE": "your_database_name",
  "PG_USERNAME": "your_username",
  "PG_PASSWORD": "your_password"
}
```

Alternatively, connection details can be provided directly via function arguments (primarily for API integration).


## Class: `DatabaseTableLister`

This class encapsulates the logic for interacting with the PostgreSQL database and retrieving table information.

### Methods:

*   `__init__(self, config_file: str = "environment-variables.json", connection_pool=None, config: Optional[Dict] = None)`:  Constructor. Initializes the object, loading configuration from either a JSON file or supplied arguments.  Handles potential errors during file loading. The `connection_pool` argument allows using connection pooling for better performance in multi-threaded or multi-process scenarios.

*   `_load_config(self, config_file: str) -> Dict[str, Any]`: Loads database connection parameters from the specified JSON configuration file.  Includes robust error handling for file not found and JSON parsing errors.

*   `_get_connection(self)`: Establishes a connection to the PostgreSQL database using the loaded configuration parameters. Uses either the connection pool if provided, the explicitly passed config dictionary, or the config file as a fallback.  Handles `psycopg2.Error` exceptions.

*   `get_all_tables(self, schema: str = "public", name_pattern: Optional[str] = None, ..., include_stats: bool = False) -> List[Dict[str, Any]]`: The core method for fetching table information. It constructs and executes a SQL query to retrieve table details from the `information_schema`. The query includes support for various filtering criteria. It optionally retrieves table statistics (row count, size, etc.) using `_get_table_stats`.  Includes comprehensive error handling for database query errors.

*   `_get_table_stats(self, cursor, schema: str, table_name: str) -> Dict[str, Any]`:  Retrieves detailed statistics for a single table, including row count, size, index size, and last vacuum/analyze times. Uses separate SQL queries for efficiency.  Includes warning messages if statistics retrieval fails for a table.

*   `get_match_specific_tables(self, match_id: str, sport: str = "MFB") -> List[Dict[str, Any]]`: A convenience method to retrieve tables related to a specific match ID.

*   `get_sport_tables(self, sport: str) -> List[Dict[str, Any]]`: A convenience method to get all tables for a specific sport.


*   `close(self)`: Closes the database connection, handling cases where a connection pool is used versus a direct connection.


## Function: `format_output(tables: List[Dict[str, Any]], format_type: str = "csv", include_stats: bool = False) -> str`

This function formats the retrieved table data into either CSV or JSON format for output.  It handles the case where no tables are found.


## Function: `main(...)`

This function serves as the entry point for the command-line interface, using `typer` to parse command-line arguments and execute the core logic.  It includes a detailed help message with usage examples.  It handles `KeyboardInterrupt` and other exceptions gracefully, ensuring clean exit.

## Function: `get_table_list(...) -> Dict[str, Any]`

This function provides an API-friendly interface to the core functionality. It returns a dictionary containing the results, metadata (success flag, table count, total size), the formatted output, and potentially error messages.  This is designed for programmatic use in other applications.

## Error Handling

The script incorporates extensive error handling throughout, including checking for configuration file errors, database connection errors, and SQL query errors.  Error messages are displayed using `typer.echo` to the console (or returned in the dictionary for API use), and appropriate exit codes are used.

## Usage

The script can be run from the command line with various options:

```bash
python get_table_list.py [OPTIONS]
```

See the `main` function's docstring for detailed usage examples.


##  API Integration

The `get_table_list` function is designed for easy integration into other Python applications or services.  It accepts connection parameters and filtering criteria as arguments and returns a structured JSON response.  This allows for dynamic querying of database tables from external programs.

