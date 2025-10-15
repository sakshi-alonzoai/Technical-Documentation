# Documentation for `roster_processor_api.py`

# Roster Processor API Documentation

## Overview

The `roster_processor` Python script automates the process of downloading roster data, migrating it to a PostgreSQL database, executing the ActiveRoster workflow, and updating the last download timestamp in the database.  It's designed to handle different sports and leagues, providing a robust and efficient solution for roster management. The script leverages asynchronous programming and utilizes several external libraries for data fetching, migration, workflow execution, and Slack notifications.


## Dependencies

The script relies on the following Python packages:

* `asyncio`: For asynchronous programming.
* `logging`: For logging events and errors.
* `time`: For time-related operations.
* `concurrent.futures`: For managing thread pools.
* `contextlib`: For context managers.
* `datetime`: For date and time manipulation.
* `functools`: For higher-order functions.
* `pathlib`: For path manipulation.
* `typing`: For type hints.
* `psycopg2`: For PostgreSQL database interaction.
* `fetch_games`:  (Assumed to be a custom library) Provides `SportType` enum.
* `GeniusToS3Download`: (Assumed to be a custom library) Contains `create_downloader` function for downloading data from Genius.
* `s3_to_postgres_migrator`: (Assumed to be a custom library) Contains `run_migration_async` for data migration.
* `Orchestrator`: (Assumed to be a custom library) Contains `run_workflow` for executing the ActiveRoster workflow.
* `slack_notifier`: (Assumed to be a custom library) Contains `post_to_slack` for sending Slack notifications.


## Functions

### 1. `_roster_file_logger(sport: str, league_id: int)`

* **Purpose:** An asynchronous context manager that creates a log file for each roster processing job.  The log file is named according to sport, league ID, and timestamp.  It ensures that the file handler is properly closed even if exceptions occur.
* **Implementation Details:** Creates a log directory if it doesn't exist. Creates a file handler with a specific format and adds it to the logger.  The handler is removed and closed in the `finally` block.
* **Returns:** An asynchronous context manager.

### 2. `run_in_thread_pool_with_timeout(func, *args, max_workers=1, timeout=1800, **kwargs)`

* **Purpose:** Executes a given function in a thread pool with a timeout. This allows for long-running operations to be performed concurrently without blocking the main thread and with a mechanism to prevent indefinite hanging.
* **Implementation Details:** Uses `ThreadPoolExecutor` to run the function in a separate thread.  `asyncio.wait_for` ensures that the function completes within the specified timeout.  Handles exceptions gracefully and shuts down the executor.
* **Returns:** The result of the executed function.

### 3. `_fetch_roster_state_from_db_async(async_pg_pool, sport: str, league_id: int, table_name: str = "roster_processor_state") -> Optional[Dict[str, Any]]`

* **Purpose:** Fetches the last downloaded roster timestamp from the PostgreSQL database.
* **Implementation Details:** Uses an asynchronous PostgreSQL connection pool to execute a SQL query.  Handles the case where no previous state is found.
* **Returns:** A dictionary containing the sport, league ID, and last download timestamp (if available), or `None` if no record is found.

### 4. `_persist_roster_state_to_db_async(async_pg_pool, sport: str, league_id: int, last_download: date, table_name: str = "roster_processor_state")`

* **Purpose:** Updates the last downloaded roster timestamp in the PostgreSQL database.
* **Implementation Details:** Uses an asynchronous PostgreSQL connection pool to execute an INSERT/UPDATE SQL statement using `ON CONFLICT` for upsert functionality.
* **Returns:**  `None`.

### 5. `_prepare_temp_persons_table(sport: str, sync_pg_pool)`

* **Purpose:** Creates or truncates a temporary table (`temp_persons_<sport>`) that mirrors the structure of the main persons table (`persons_<sport>`).
* **Implementation Details:** Executes synchronous PostgreSQL commands. The function handles potential exceptions and ensures that the connection is returned to the pool.
* **Returns:** `None`.

### 6. `process_roster_api(sport: str = "MFB", league_id: int = None, *, config: Dict[str, Any], sync_pg_pool, async_pg_pool, aws_env: Dict[str, str], state_table: str = "roster_processor_state", force: bool = False) -> Dict[str, Any]`

* **Purpose:** This is the main function that orchestrates the entire roster processing pipeline.
* **Implementation Details:**  It sequentially performs the following steps: prepares temporary table, loads the previous state, downloads roster data, migrates data to Postgres, runs the ActiveRoster workflow, persists the new state, and sends a Slack notification. Each step is carefully handled with error checking and logging. Uses thread pools for long-running operations.
* **Returns:** A dictionary indicating success/failure, along with relevant information like the number of records processed, workflow status, last download timestamp, and execution timings of individual stages.


## Error Handling and Logging

The script incorporates robust error handling and logging mechanisms using the `logging` module and `try...except` blocks throughout the code.  Log files are created for each run, providing detailed information about the process.  Error messages are logged and also returned in the output dictionary for easy analysis.


## Configuration

The script requires a configuration dictionary (`config`) containing database credentials, AWS credentials, and other environment-specific settings.  These settings should be managed securely, outside of the codebase.


## Future Improvements

* **More sophisticated error handling:** Implement more granular error handling for specific scenarios.
* **Improved logging:**  Consider using a structured logging format (e.g., JSON) for easier parsing and analysis.
* **Health checks:**  Add health checks to verify database connectivity and other dependencies before starting processing.
* **Metrics:**  Add metrics tracking for key performance indicators (e.g., processing time, number of records).



This documentation provides a comprehensive overview and detailed explanation of the `roster_processor` script.  The clarity and structure make it easy for developers and system administrators to understand its functionality, dependencies, and usage.

