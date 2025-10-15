# Documentation for `get_player_ids.py`

# Technical Documentation: `get_player_ids.py`

## Overview

The Python script `get_player_ids.py` retrieves distinct player IDs from a PostgreSQL database for specified sports and match IDs.  It offers flexibility in querying: single match IDs, or batches of match IDs. The script utilizes the `typer` library for creating a command-line interface (CLI) and `psycopg2` for database interaction.  Configuration details for the PostgreSQL connection are loaded from a JSON file.


## Dependencies

- Python 3.7+
- `psycopg2` (PostgreSQL adapter):  `pip install psycopg2-binary`
- `typer` (CLI framework): `pip install typer`


## Configuration

The script expects a JSON configuration file (default: `environment-variables.json`) located in the same directory as the script. This file should contain the following PostgreSQL connection parameters:

- `PG_HOST`:  PostgreSQL server hostname or IP address.
- `PG_PORT`: PostgreSQL server port (default: 5432 if not specified).
- `PG_DATABASE`: Name of the PostgreSQL database.
- `PG_USERNAME`: PostgreSQL username.
- `PG_PASSWORD`: PostgreSQL password.

**Example `environment-variables.json`:**

```json
{
  "PG_HOST": "localhost",
  "PG_PORT": 5432,
  "PG_DATABASE": "mydatabase",
  "PG_USERNAME": "myuser",
  "PG_PASSWORD": "mypassword"
}
```

The config file path can be overridden using the `--config-path` or `-c` command-line option.


## Functions

### `load_db_config(config_path: str = "environment-variables.json", required: bool = True) -> dict`

This function loads PostgreSQL connection details from the specified JSON configuration file.

- **Arguments:**
    - `config_path`: Path to the JSON configuration file (default: "environment-variables.json").
    - `required`: Boolean indicating whether the config file is required (default: True). If False, an empty dictionary is returned.

- **Returns:** A dictionary containing the PostgreSQL connection parameters.  Raises `FileNotFoundError` if the file is not found, `ValueError` if the JSON is invalid, or `RuntimeError` if any required parameters are missing.

- **Implementation Details:** The function first checks if the config file is required.  It then determines the script's directory to locate the config file.  Error handling is implemented to catch file not found and JSON decoding errors, ensuring robustness.


### `get_player_ids(sport: str, g_match_id: Optional[int] = None, g_match_ids: Optional[List[int]] = None, config_path: str = "environment-variables.json", connection_pool=None, config: Optional[Dict] = None, **connection_params) -> List[str]`

This is the core function that retrieves player IDs from the database.

- **Arguments:**
    - `sport`: The sport type (MFB, MBB, WBB). Case-insensitive.
    - `g_match_id`: A single match ID (for backwards compatibility).
    - `g_match_ids`: A list of match IDs (for batch processing).  Use this instead of `g_match_id` for better performance.
    - `config_path`: Path to the database configuration file.
    - `connection_pool`: An optional connection pool object (for improved performance in high-traffic scenarios).
    - `config`: Optional database config dictionary.
    - `**connection_params`: Additional connection parameters to be passed to `psycopg2.connect()`.

- **Returns:** A list of distinct player IDs as strings.

- **Raises:** `ValueError` for invalid input (e.g., both `g_match_id` and `g_match_ids` provided, empty match ID list, invalid match ID, or invalid sport).

- **Implementation Details:**  The function validates inputs, normalizes the match IDs to a list, builds a parameterized SQL query using an `IN` clause for efficient retrieval of multiple match IDs, executes the query, and handles database connection and error conditions.  Supports the use of an optional connection pool for better performance and resource management.


### `get_player_ids_single(sport: str, g_match_id: int, config_path: str = "environment-variables.json") -> List[str]`

This function is a wrapper around `get_player_ids` for backward compatibility, accepting only a single `g_match_id`.


### CLI Commands (`single`, `batch`, `main`)

The script provides three Typer commands:

- `single`: Retrieves player IDs for a single match ID.
- `batch`: Retrieves player IDs for multiple match IDs provided as space-separated arguments.  Includes a `--verbose` option to display the number of matches and players found.
- `main`: A legacy command for backward compatibility; functionally equivalent to `single`.  Use `single` or `batch` instead.

All CLI commands handle potential errors and print appropriate messages using the `typer` library.


## Usage

The script can be run from the command line using Typer's commands:

```bash
# Single match ID
python get_player_ids.py single MFB 12345

# Multiple match IDs
python get_player_ids.py batch MBB 67890 11111 22222 --verbose

# Using a custom config file path
python get_player_ids.py batch WBB 33333 44444 -c my_db_config.json 
```


## Error Handling

The script includes comprehensive error handling for file I/O, JSON parsing, database connection issues, and invalid input parameters. Error messages are user-friendly and informative.


## Potential Improvements

- **Logging:** Implement proper logging for better debugging and monitoring.
- **Connection Pooling:** Integrate a robust connection pooling mechanism for improved performance in production environments.
- **Input Validation:** Add more sophisticated input validation to handle edge cases.
- **Asynchronous Operations:** Consider using asynchronous operations for database interactions to improve performance, especially when handling many match IDs.
- **Configuration Management:** Explore more robust configuration management solutions beyond simple JSON files (e.g., environment variables, dedicated config libraries).

This detailed documentation provides a comprehensive overview of the `get_player_ids.py` script, including its functionality, dependencies, usage instructions, and potential areas for improvement.  The clear structure and explanations make it suitable for both developers and users.

