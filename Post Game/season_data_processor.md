# Documentation for `season_data_processor.py`

# College Football Season Data Processing Script: Technical Documentation

## Overview

This Python script, `season_data_processor.py`, is designed to download and process college football season match data. Unlike a daily game processor, this script focuses on comprehensive season-level data management. It runs periodically to ensure that the stored season data remains current and accurate.  The script interacts with an external API to retrieve data and a PostgreSQL database to store it.  It also uses S3 for temporary data storage during the processing pipeline.


## Dependencies

* Python 3.7+
* `typer`: For command-line interface.  Install with `pip install typer`
* `requests`: For making HTTP requests to the API. Install with `pip install requests`
* `json`: Built-in Python library for JSON handling.
* `logging`: Built-in Python library for logging.
* `os`: Built-in Python library for operating system interaction.
* `sys`: Built-in Python library for system-specific parameters and functions.
* `datetime`: Built-in Python library for date and time manipulation.
* `pathlib`: Built-in Python library for working with file paths.
* A PostgreSQL database accessible via the specified API endpoints.
* An S3 bucket accessible via the specified API endpoints.

## Configuration

The script relies on a configuration file (default: `config/environment-variables.json`) which should contain:

* `"api"`: A dictionary with:
    * `"base_url"`: The base URL of the API endpoint (default: `"http://localhost:8000"`).
    * `"timeout"`: Timeout for API requests in seconds (default: 1000).

This file should be created in the `config` directory before running the script.

## State Management

The script uses a state file (default: `season_state.json`) to track its progress and avoid redundant processing. This file stores:

* `"competitions"`: A dictionary of competition IDs, each with:
    * `"competition_id"`: The ID of the competition.
    * `"s3_path"`: The S3 path where the downloaded data is stored.
    * `"checksum"`: A checksum of the downloaded data.
    * `"last_updated"`: The timestamp of the last data update.
    * `"last_processed"`: The timestamp of the last data processing.
    * `"flat_match_completed"`: Boolean indicating if the `FlatMatch` workflow has finished.

* `"last_run"`: The timestamp of the script's last execution.


## Classes and Functions

### `SeasonDataProcessor` Class

This class encapsulates the main logic of the script.

* **`__init__(self, config_file: str = "environment-variables.json", state_file: str = "season_state.json", competition_id: int = None)`:** Initializes the processor with configuration and state data. Sets up logging and loads config and state from JSON files.  If the state file does not exist it creates a default state.

* **`_load_config(self, config_file: str) -> Dict[str, Any]`:** Loads the configuration from the specified JSON file, handling potential errors.

* **`_load_state(self) -> Dict[str, Any]`:** Loads the state from the JSON file, creating a default state if the file doesn't exist or is corrupt.

* **`_create_default_state(self) -> Dict[str, Any]`:** Creates an empty state dictionary with a default structure.

* **`_get_or_create_competition_state(self, competition_id: int) -> Dict[str, Any]`:** Retrieves the state for a specific competition, creating it if it doesn't exist.

* **`_save_state(self)`:** Saves the current state to the state file.

* **`_setup_logging(self)`:** Configures logging to write to a timestamped log file in a `logs` directory.

* **`_call_api_endpoint(self, endpoint: str, method: str = "POST", payload: Dict = None, params: Dict = None) -> Optional[Dict]`:** A generic function to make API calls, handling various error conditions.

* **`_truncate_table(self, table_name: str) -> bool`:** Truncates a PostgreSQL table using an API endpoint. Returns `True` on success, `False` otherwise.

* **`download_season_match_data(self) -> bool`:** Downloads season match data, checks for changes, and processes the data if changes are detected.  It uses checksums to determine if the data has changed.

* **`_process_season_data_to_postgres(self, s3_path: str) -> bool`:** Processes data downloaded from S3 to Postgres, truncating the table before inserting new data. Returns `True` on success, `False` otherwise.

* **`_run_flat_match_workflow(self) -> bool`:** Runs the "FlatMatch" workflow via an API call. Returns `True` on success, `False` otherwise.

* **`process_season_data(self)`:** The main method to orchestrate the data processing workflow.

* **`get_competition_status(self) -> Dict[str, Any]`:** Retrieves the current status of the specified competition.


### `main` Function

This function serves as the entry point for the script. It parses command-line arguments and instantiates the `SeasonDataProcessor` class.  It handles command-line flags for specifying config file paths, a state file path, and the competition ID. It also handles the `--status` flag to only display the current status without processing any data.


## Usage

The script can be run from the command line using `typer`:

```bash
python season_data_processor.py --competition-id <competition_id> [--config <config_file>] [--state <state_file>] [--status]
```

* `<competition_id>`: The ID of the college football competition.  This is required.
* `<config_file>`: (Optional) Path to the configuration file. Defaults to `"config/environment-variables.json"`.
* `<state_file>`: (Optional) Path to the state file. Defaults to `"season_state.json"`.
* `--status`: (Optional) If specified, only the current status will be printed; data processing will not be performed.


## Error Handling

The script includes comprehensive error handling using `try-except` blocks and logging to capture and report errors during configuration loading, API calls, data processing, and state saving.  Error messages are logged to a timestamped log file and (optionally) to the console.  The script exits with a non-zero exit code in case of errors.


## Logging

The script uses the Python `logging` module to log events, including informational messages, warnings, and errors, to a timestamped log file within a `logs` directory.  This aids in debugging and monitoring the script's execution.  The log file contains timestamps, log levels, and detailed messages to improve troubleshooting.

## Atomic Operations

The script uses a combination of techniques to ensure data consistency and atomicity (all-or-nothing operations):

* **Table Truncation:** Before loading new season data, the `matches_mfb` table is truncated, ensuring a clean slate.
* **Checksums:**  The script checks checksums of downloaded data to prevent redundant processing of unchanged data.
* **`FlatMatch` Workflow:**  This workflow, invoked via the API, handles atomic replacement of data within its internal pipeline.

These measures help prevent partial updates or data corruption.


This documentation provides a complete overview of the script's functionality, dependencies, usage, and error handling mechanisms. The detailed explanations of classes, functions, and data structures facilitate understanding and maintainability.

