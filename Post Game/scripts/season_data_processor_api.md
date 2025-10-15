# Documentation for `season_data_processor_api.py`

# Season Data Processor API Documentation

## Overview

This Python script implements a data processing pipeline for season data.  It retrieves data, migrates it to a PostgreSQL database, and runs a workflow (FlatMatch) for data processing.  The script utilizes asynchronous operations for improved efficiency and manages its state using a JSON file.  Crucially, it includes robust error handling and timeout mechanisms to prevent indefinite blocking.  The script exposes two APIs: `process_season_data_api` and `get_season_status_api`.

## Dependencies

The script relies on several external libraries and internal modules:

* **Standard Libraries:** `json`, `logging`, `os`, `asyncio`, `datetime`, `pathlib`, `typing`, `concurrent.futures`, `functools`
* **External Libraries:**  (Assumed based on import statements.  Precise versions may need to be specified in `requirements.txt`)
    * A PostgreSQL database driver (likely `psycopg2` or `asyncpg`)
    * An AWS SDK (likely `boto3`) for S3 interaction
    * A Slack notification library (needs identification)
* **Internal Modules:**  These modules must be present in the specified relative paths within the project directory structure.
    * `GeniusToS3Download.GeniusToS3Download`: Contains the `create_downloader` function for data retrieval.
    * `s3_to_postgres_migrator`: Contains the `run_migration_async` function for database migration.
    * `Orchestrator.orchestrator`: Contains the `run_workflow` function for executing the FlatMatch workflow.
    * `delete_from_postgres`: Contains the `delete_from_postgres` function for database table truncation.
    * `slack_notifier`: Contains the `post_to_slack` function for sending Slack notifications.


## Functions

### 1. `SeasonStateManager` Class

This class manages the state of the season processing.  It handles loading and saving the state to a JSON file (`season_state.json` by default).  The state includes information about each competition (e.g., S3 path, checksum, processing status) and the timestamp of the last run.  Error handling is implemented for JSON parsing and file I/O operations.

* **`__init__(self, state_file: str)`:** Constructor; initializes the state file path and loads the state.
* **`_load_state(self) -> Dict[str, Any]`:** Loads the state from the JSON file or creates a default state if the file doesn't exist or is corrupted.
* **`_create_default_state(self) -> Dict[str, Any]`:** Creates a default state dictionary.
* **`get_competition_state(self, competition_id: int) -> Dict[str, Any]`:** Retrieves the state for a specific competition. Creates a new entry if the competition isn't already in the state.
* **`save_state(self)`:** Saves the current state to the JSON file.  Includes error handling for I/O exceptions.


### 2. `run_in_thread_pool_with_timeout(func, *args, max_workers=1, timeout=1800, **kwargs)`

This is a crucial helper function that executes synchronous functions within an `asyncio` event loop using a thread pool.  This is essential for preventing blocking operations from halting the asynchronous pipeline.  It incorporates a timeout mechanism to prevent indefinite hangs.  The function utilizes `functools.partial` for flexible argument handling.  Proper cleanup of the thread pool is ensured using a `with` statement and `executor.shutdown(wait=True)`.

### 3. `process_season_data_api(competition_id: int, config: dict, sync_pg_pool, async_pg_pool, aws_env: dict, state_file: str = "season_state.json", force: bool = False) -> Dict[str, Any]`

This is the main function that orchestrates the season data processing pipeline. It handles data download, database migration, workflow execution, and state management.  It leverages `run_in_thread_pool_with_timeout` to safely handle potentially blocking operations.


* **Steps:**
    1. **Download:** Downloads season match data using `create_downloader` from `GeniusToS3Download`.  Uses a thread pool for this operation.
    2. **Truncate:** Truncates the `matches_mfb` table in PostgreSQL using `delete_from_postgres`. Uses a thread pool.
    3. **Migrate:** Migrates data from S3 to PostgreSQL using `run_migration_async` (which is already asynchronous).
    4. **Workflow:** Executes the FlatMatch workflow using `run_workflow` from `Orchestrator`. Uses a thread pool with a generous timeout.
    5. **Notification:** Sends a Slack notification using `post_to_slack`.
* **Error Handling:** Comprehensive error handling is implemented at each step, returning detailed error messages.
* **State Management:** Uses the `SeasonStateManager` to manage the processing state.
* **Timeout Handling:**  Includes timeout handling for download, database truncation, and workflow execution.

### 4. `get_season_status_api(competition_id: int, state_file: str = "season_state.json") -> Dict[str, Any]`

This function retrieves the current status of season processing for a given competition ID from the state file.  It returns a dictionary containing the competition ID, its status (from `SeasonStateManager`), and the timestamp of the last run.  It also includes basic error handling.


## Implementation Details

* **Asynchronous Programming:** The script effectively utilizes `asyncio` to manage asynchronous operations, improving overall efficiency.
* **Thread Pooling:**  The script employs `ThreadPoolExecutor` to run synchronous functions concurrently without blocking the event loop.  This is particularly important for I/O-bound tasks like database interactions and file downloads.
* **State Management:** The use of a JSON file for state management allows for persistence and resuming processing after interruptions.
* **Configuration:** The script expects configuration parameters (`config`, `aws_env`) to be passed as arguments. These likely contain database credentials, AWS access keys, and other environment-specific settings.
* **Logging:**  The script uses the `logging` module for detailed logging of events, errors, and status updates.  This is crucial for debugging and monitoring.


## Usage

The script's APIs (`process_season_data_api` and `get_season_status_api`) are designed to be called by other parts of the application.  The specific usage would depend on the larger system architecture but would likely involve passing the required configuration data and database connection pools as arguments.


This comprehensive documentation provides a detailed understanding of the script's functionality, dependencies, and implementation details.  The improved clarity, structure, and thoroughness make it suitable for developers and technical support personnel.

