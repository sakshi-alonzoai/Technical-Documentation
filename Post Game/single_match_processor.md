# Documentation for `single_match_processor.py`

# Single Match Data Processing Script - Technical Documentation

## 1. Overview

This Python script (`single_match_processor.py`) is designed to download, process, and store data for individual college football matches. It leverages a modular design, checksum-based change detection, and separate state files to enable efficient and parallel processing of multiple matches.  The script interacts with several API endpoints for data retrieval and processing, and integrates with a PostgreSQL database for persistent storage.  A key enhancement is the inclusion of derived statistics calculations (season and career data) for players and teams.

## 2. Dependencies

The script relies on the following Python libraries:

* `typer`: For creating command-line interfaces.
* `requests`: For making HTTP requests to API endpoints.
* `json`: For handling JSON data.
* `logging`: For logging events and errors.
* `os`, `sys`, `threading`, `datetime`, `pathlib`, `signal`: Standard Python libraries for various functionalities.


The script also requires a correctly configured JSON configuration file (`environment-variables.json` by default,  but configurable via `--config` command-line option) specifying API endpoints and other parameters.  A second configuration file (`migrator_config.json`, located in the same directory as `environment-variables.json`) is used for data migration to Postgres.  These files are crucial for the script's operation.

## 3. File Structure and State Management

The script maintains a `match_processor_state/` directory to store JSON state files for each processed match. Each state file (e.g., `match_12345_state.json`) contains information about the match's processing status, checksums of downloaded data, timestamps, and workflow completion flags. This allows the script to resume processing or skip already processed matches efficiently.  Log files are also created in a timestamped subdirectory under `logs/`.

## 4. Class Structure and Function Details

The core functionality is encapsulated within the `SingleMatchProcessor` class:

### 4.1 `__init__(self, config_file, state_folder, match_id, competition_id)`

* **Purpose:** Initializes the `SingleMatchProcessor` object.
* **Parameters:**
    * `config_file`: Path to the configuration JSON file.
    * `state_folder`: Path to the directory for storing match state files.
    * `match_id`: ID of the match to process.
    * `competition_id`: ID of the competition (required for derived statistics).
* **Actions:** Loads configuration, creates the state folder if needed, sets up logging, and loads or creates the match's state file.

### 4.2 `_load_config(self, config_file)`

* **Purpose:** Loads configuration settings from the specified JSON file.
* **Parameters:** `config_file`: Path to the configuration file.
* **Returns:** A dictionary containing configuration settings.  Exits with an error if the file is not found or is not valid JSON.

### 4.3 `_load_state(self)`

* **Purpose:** Loads the match state from the corresponding JSON file. Creates a default state if the file does not exist.
* **Parameters:** None.
* **Returns:** A dictionary containing the match state.  Handles JSON decoding errors gracefully.

### 4.4 `_create_default_state(self)`

* **Purpose:** Creates a default match state dictionary.
* **Parameters:** None.
* **Returns:** A dictionary representing the default state structure.

### 4.5 `_save_state(self)`

* **Purpose:** Saves the current match state to the JSON file.  Includes updating the `last_updated` and `last_run` timestamps.
* **Parameters:** None.
* **Returns:** None. Logs errors if saving fails.

### 4.6 `_setup_logging(self)`:

* **Purpose:** Configures logging to write to a timestamped log file and (optionally) to the console.
* **Parameters:** None.
* **Returns:** The configured logger object.


### 4.7 `_call_api_endpoint(self, endpoint, method, payload, params)`

* **Purpose:** A generic function for making API calls to various endpoints. Handles different HTTP methods (GET, POST), payloads, and parameters.  Includes error handling for HTTP request failures and JSON decoding errors.
* **Parameters:**
    * `endpoint`: The API endpoint path.
    * `method`: HTTP method ("GET" or "POST").
    * `payload`: Data payload for POST requests.
    * `params`: Query parameters for GET requests.
* **Returns:** The JSON response from the API, or `None` if the request fails.

### 4.8 `check_for_new_data(self, match_id)`

* **Purpose:** Checks for new data by downloading match statistics (`person_match` and `team_match`) and comparing checksums.
* **Parameters:** `match_id`: The ID of the match.
* **Returns:** A tuple `(has_any_new_data, has_new_person_data, has_new_team_data)` indicating whether new data is available for any, person, and team statistics, respectively.

### 4.9 `download_match_statistics(self, match_id, stat_type)`

* **Purpose:** Downloads person or team match statistics from the API.
* **Parameters:**
    * `match_id`: Match ID.
    * `stat_type`: Type of statistics ("person_match" or "team_match").
* **Returns:** A dictionary containing the downloaded statistics (s3 path, checksum, timestamp) or `None` if download fails.

### 4.10 `process_match_data_to_postgres(self, match_id, s3_path, table_type)`

* **Purpose:** Processes match data from S3 and loads it into the PostgreSQL database using match-specific tables.
* **Parameters:**
    * `match_id`: Match ID.
    * `s3_path`: S3 path to the data.
    * `table_type`: Type of table ("person_match" or "team_match").
* **Returns:** `True` if successful, `False` otherwise.

### 4.11 `run_match_workflow(self, match_id, workflow_name)`

* **Purpose:** Runs a specified workflow (e.g., "PlayerGameStatistics") via the orchestrator API.
* **Parameters:**
    * `match_id`: Match ID.
    * `workflow_name`: Name of the workflow.
* **Returns:** `True` if successful, `False` otherwise.

### 4.12 `_get_match_teams(self, match_id)`

* **Purpose:** Retrieves the team IDs involved in a specific match.
* **Parameters:** `match_id`: The ID of the match.
* **Returns:** A list of team IDs. Returns an empty list if retrieval fails.

### 4.13 `_get_affected_players(self, match_id)`

* **Purpose:** Retrieves the player IDs involved in a specific match.
* **Parameters:** `match_id`: The ID of the match.
* **Returns:** A list of player IDs (UUIDs). Returns an empty list if retrieval fails.

### 4.14 `_create_temp_view(self, view_type, team_ids, player_ids, season, competition_id)`

* **Purpose:** Creates temporary PostgreSQL views for use in derived statistics calculations.  The views are created dynamically based on the `view_type` and relevant parameters.
* **Parameters:**
    * `view_type`: Type of view ('player_season', 'team_season', 'player_career').
    * `team_ids`: List of team IDs (for season views).
    * `player_ids`: List of player IDs (for career views).
    * `season`: Season number (for season views).
    * `competition_id`: Competition ID (for season views).
* **Returns:** The name of the created view, or `None` if creation fails.

### 4.15 `_drop_temp_view(self, view_name)`

* **Purpose:** Drops a temporary PostgreSQL view.
* **Parameters:** `view_name`: The name of the view to drop.
* **Returns:** `True` if successful, `False` otherwise.  Handles cases where the view might not exist.

### 4.16 `_update_derived_statistics(self, match_id)`

* **Purpose:** Updates season and career derived statistics after processing match-level data.  This function manages the creation and dropping of temporary views, and calls the appropriate workflow APIs.
* **Parameters:** `match_id`: The ID of the match.
* **Returns:** `True` if successful, `False` otherwise.

### 4.17 `process_single_match(self, match_id, skip_derived)`

* **Purpose:** Orchestrates the end-to-end processing of a single match, including data download, database updates, and workflow execution.  Includes checksum-based change detection.
* **Parameters:**
    * `match_id`: Match ID.
    * `skip_derived`: Boolean flag to skip derived statistics processing.
* **Returns:** `True` if successful, `False` otherwise.

### 4.18 `get_match_status(self, match_id)`

* **Purpose:** Retrieves the processing status of a match.
* **Parameters:** `match_id`: Match ID.
* **Returns:** A dictionary containing match status information.

### 4.19 `list_all_processed_matches(self)`

* **Purpose:** Lists all matches processed so far by scanning the state directory.
* **Parameters:** None.
* **Returns:** A dictionary containing a summary of processed matches (total, completed, details).  Handles potential file reading errors.


## 5. `main` Function

The `main` function acts as the entry point for the script, parsing command-line arguments and coordinating the execution of the `SingleMatchProcessor`.  It supports various operational modes:

* Processing a single match (`--match-id`, `--competition-id` required).
* Checking for new data only (`--check`).
* Showing the current status of a match (`--status`).
* Listing all processed matches (`--list`).
* Forcing reprocessing even if no new data is detected (`--force`).
* Skipping derived statistics processing (`--skip-derived`).

## 6. Error Handling

The script incorporates robust error handling throughout its various functions, logging errors to files and (optionally) the console.  Specific error handling includes:

* JSON decoding errors during config file and state file loading.
* API request errors (HTTP errors, timeouts).
* Database interaction errors (PostgreSQL related).
* File I/O errors.

The script aims to provide informative error messages to facilitate debugging.  A signal handler (`debug_hanging_process`) is included to assist with debugging hanging processes.


## 7. Parallel Processing

The script is designed to be run independently for each match, enabling parallel processing of multiple matches. This is achieved by using separate state files for each match in the `match_processor_state` directory.  Multiple instances of the script can run concurrently without interfering with each other.


## 8.  Enhancements (Derived Statistics)

A significant enhancement is the inclusion of derived statistics calculations. The script now supports updating season and career statistics for players and teams after processing match-level data.  This involves creating temporary views in the database to facilitate efficient calculation and updating of aggregated statistics.  These views are automatically dropped after the calculations are complete.


This detailed documentation provides a comprehensive understanding of the `single_match_processor.py` script's functionality, dependencies, design, and error handling mechanisms.  The inclusion of derived statistics significantly enhances the script's capabilities and makes it a more powerful tool for data processing in the context of college football statistics.

