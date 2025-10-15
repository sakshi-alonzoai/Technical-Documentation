# Documentation for `single_match_processor_api.py`

# Single Match Processor API Documentation

## Overview

This Python script implements a Single Match Processor API designed for processing individual match data.  Instead of relying on HTTP API calls, it directly imports functions for data fetching, processing, and state management.  This revised version incorporates robust asynchronous execution and timeout handling to prevent delays and ensure reliable operation.

The API provides three core functionalities:

1. **`process_single_match_api`**: Processes a single match, handling data download, ingestion into a Postgres database, execution of various data workflows (PlayerGameStatistics, TeamGameStatistics, PlayerSeasonStatistics, TeamSeasonStatistics, PlayerCareerStatistics), and sending Slack notifications.

2. **`get_match_status_api`**: Retrieves the current processing status of a specified match.

3. **`list_processed_matches_api`**: Provides a summary of all matches processed, including their completion status and timestamps.


## Dependencies

The script relies on several external libraries:

* `asyncio`: For asynchronous operations.
* `json`: For JSON data handling.
* `logging`: For logging events and errors.
* `os`: For interacting with the operating system.
* `sys`: For accessing command-line arguments and system-specific parameters.
* `time`: For time-related functions.
* `concurrent.futures`: For using thread pools.
* `contextlib`: For context managers.
* `datetime`: For working with dates and times.
* `functools`: For higher-order functions.
* `pathlib`: For working with file paths.
* `typing`: For type hinting.
* `pytz`: For handling time zones.
* `asyncpg`: For asynchronous PostgreSQL database interactions.
* Custom modules (located in the same directory or parent directories):
    * `fetch_games`: Contains functions to retrieve game data by match ID.  Specifically uses `fetch_game_by_match_id`
    * `GeniusToS3Download`: Contains the `create_downloader` function to download data from Genius and upload to S3.  Specifically uses `create_downloader` and the `download_data` method within the downloaded class.
    * `get_player_ids`: Contains `get_player_ids` for retrieving player IDs associated with a match.
    * `Orchestrator`: Contains `run_workflow` for executing various data processing workflows.
    * `postgres_views`: Contains `create_view` and `drop_view` for managing temporary Postgres views.
    * `s3_to_postgres_migrator`: Contains `run_migration_async` for migrating data from S3 to Postgres.
    * `slack_notifier`: Contains `post_to_slack` for sending Slack notifications.


## Functions

### 1. `process_single_match_api`

This is the core function of the script. It orchestrates the entire match processing pipeline:

* **Input**:  `match_id`, `competition_id`, `config`, `sync_pg_pool`, `async_pg_pool`, `aws_env`, `state_folder`, `skip_derived`, `force`.
* **Output**: A dictionary indicating success/failure, along with relevant metadata, processing details, and timing information.
* **Steps**:
    1. **Loads match state**: Retrieves existing processing state from a Postgres database, creating a default state if none exists.  The state is cached locally for efficiency.
    2. **Checks for new data**: Downloads person and team match statistics from S3, and compares checksums to determine if new data is available, or if processing is forced.
    3. **Processes match data**: Migrates person and team match data from S3 to Postgres using `run_migration_async`.
    4. **Runs workflows**: Executes the specified data workflows (PlayerGameStatistics, TeamGameStatistics, etc.) using `run_workflow`.  Workflows are run in a thread pool to prevent blocking.  Temporary views are created and dropped using `create_view` and `drop_view` respectively.  These calls are also handled in a thread pool.
    5. **Sends Slack notification**: Sends a Slack notification summarizing the match processing status using `post_to_slack`.
    6. **Persists state**: Updates the match's processing state in the database using `_persist_state_to_db_async`.
    7. **Returns results**: Returns a comprehensive dictionary containing success/failure status, processing details, and timing information.


### 2. `get_match_status_api`

This function retrieves the current processing status of a single match.

* **Input**: `match_id`, `state_folder`
* **Output**: A dictionary containing the success status, match ID, and the current match state.

### 3. `list_processed_matches_api`

This function provides a summary of all matches that have been processed.

* **Input**: `state_folder`
* **Output**: A dictionary containing a summary of the processed matches, including the total number of processed matches, the number of completed matches, and details for each match.


### Helper Functions

The script uses several helper functions to improve code organization and readability. These include asynchronous context managers for logging (`_match_file_logger`), functions for running synchronous operations in a thread pool with timeouts (`run_in_thread_pool_with_timeout`), asynchronous database operations (`_persist_state_to_db_async`, `_fetch_state_from_db_async`),  and functions for handling Slack notifications (`_send_slack_notification_async`), team information retrieval (`_get_match_teams`), match statistics downloads (`_download_match_statistics`), temporary view management (`_create_temp_view`, `_drop_temp_view`), and state management (`MatchStateManager`).  Each of these functions is thoroughly documented within the script itself.


## Implementation Details

* **Asynchronous Programming**: The script leverages `asyncio` to handle multiple tasks concurrently, improving efficiency and responsiveness.  Synchronous operations are offloaded to a thread pool (`ThreadPoolExecutor`) to prevent blocking the event loop.

* **Error Handling**: Comprehensive error handling is implemented throughout the script to ensure robustness and graceful handling of failures.  Exceptions are caught, logged, and appropriate error messages are returned.

* **Logging**: Extensive logging is used to track the progress of the processing pipeline and to record errors.  Each match gets its own log file.

* **State Management**:  Match processing state is persisted to a Postgres database and cached locally, providing a robust and efficient mechanism for tracking progress and resuming interrupted processes.


##  How to Use

1.  **Install Dependencies**:  Ensure that all required Python libraries are installed (`asyncio`, `json`, `logging`, `os`, `sys`, `time`, `concurrent.futures`, `contextlib`, `datetime`, `functools`, `pathlib`, `typing`, `pytz`, `asyncpg`).  You will also need to install the custom modules listed in the "Dependencies" section.

2.  **Configure**:  Update the configuration files (environment variables, database connection details, AWS credentials, Slack webhook URL, etc.) as needed.

3.  **Run**:  Execute the script, providing the necessary command-line arguments or using the API functions directly.


This detailed documentation provides a complete understanding of the script's functionality, dependencies, implementation details, and usage instructions.  The comprehensive error handling, logging, and asynchronous capabilities ensure efficient and reliable processing of single match data.

