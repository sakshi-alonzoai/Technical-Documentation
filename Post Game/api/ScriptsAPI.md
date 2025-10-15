# Documentation for `ScriptsAPI.py`

# Sports Data Processing API - Technical Documentation

## 1. Overview

This document provides technical details for the Sports Data Processing API, a FastAPI application built to handle sports data workflows. The API integrates three core functionalities:

* **Genius Data Download:** Downloads data from the Genius API and uploads it to Amazon S3.
* **Game Data Fetching:** Retrieves game data from a PostgreSQL database.
* **Workflow Orchestration:** Executes custom sports-specific data processing workflows.

The application dynamically loads Python modules from a configuration file (`script_paths.json`), enabling flexible extension and customization of its data processing capabilities.  Long-running tasks are offloaded to a Celery task queue for asynchronous processing.

## 2. Dependencies

The application relies on the following libraries:

* FastAPI: For building the API.
* Uvicorn: ASGI server for running the FastAPI application.
* Pydantic: For data validation and serialization.
* Celery: For asynchronous task management.
* asyncpg: Asynchronous PostgreSQL driver.
* psycopg2: Synchronous PostgreSQL driver.
* boto3: For interacting with Amazon S3.
* `importlib.util`: For dynamic module loading.


## 3. Configuration

The application's behavior is controlled through environment variables (loaded from `environment-variables.json` and several JSON configuration files. `script_paths.json` specifies the location of the dynamically loaded scripts. Each script may also include its own configuration files (e.g., `migrator_config.json`).  AWS credentials are also typically loaded from environment variables.


## 4. Dynamic Script Loading (ScriptLoader Class)

The `ScriptLoader` class manages the dynamic loading of Python modules.  It performs the following actions:

1. **`load_configuration()`:** Loads script paths and other configuration data from `script_paths.json`.  Handles file not found and JSON decoding errors.
2. **`load_module_from_path()`:** Loads a Python module from a specified file path.  Includes error handling for file not found and import errors.  Adds the script's directory to `sys.path`.
3. **`validate_module_imports()`:** Optionally validates the presence of required functions in the loaded module based on configuration.  Useful for preventing runtime errors due to missing dependencies.
4. **`import_objects_from_module()`:** Imports specific objects (functions, classes) from the loaded module, mapping local names to the attributes in the module.  Handles import errors if the object is not found.
5. **`load_all_scripts()`:** Loads and initializes all scripts defined in the `script_paths.json` configuration file. It validates imports and import specified objects from the loaded modules into a dictionary for later use.


## 5. FastAPI Endpoints

The application exposes several FastAPI endpoints, organized by functionality. Each endpoint includes detailed descriptions and request/response models using Pydantic.

### 5.1. Root Endpoint (`/`)

Provides an overview of the API, including version information, endpoint list, and information about the loaded scripts.

### 5.2. Script Listing Endpoint (`/scripts`)

Returns a list of all loaded scripts with their paths, module names, and descriptions.

### 5.3. Genius Data Download (`/genius_download`)

* **Request Model:** `GeniusDownloadRequest` (sport, api, params, s3_prefix, task, postgame, memory_threshold_mb, hard_limit_mb)
* **Response Model:** `GeniusDownloadResponse` (success, job_id, s3_path, folder_checksum, message, error, download_type, timestamp)
* **Functionality:** Queues a Celery task (`download_genius_data_task`) to download data from the Genius API based on the provided parameters, and uploads the data to S3. Returns the job ID immediately.  Provides a status endpoint `/genius_download/job/{job_id}` to check download status and results.

### 5.4. Game Data Fetching (`/fetch_games`, `/fetch_games/match/{match_id}`)

* **`/fetch_games` Endpoint:**
    * **Request Parameters:** `date_str`, `sport`, `num_batches`
    * **Response Model:**  JSON response with `batches` and `metadata`.
    * **Functionality:** Fetches game data from the database, grouping games into conflict-free batches for parallel processing.
* **`/fetch_games/match/{match_id}` Endpoint:**
    * **Request Parameters:** `match_id`, `sport`, `output_format`
    * **Response Model:** `FetchGameByIdResponse`
    * **Functionality:** Fetches game data for a specific match ID.

### 5.5. Workflow Orchestration (`/orchestrator/run`, `/orchestrator/workflows`, `/orchestrator/job/{job_id}`)

* **`/orchestrator/run` Endpoint:**
    * **Request Model:** `WorkflowRequest` (sport, workflow, inputs, start_from)
    * **Response Model:** JSON response with job ID and status.
    * **Functionality:** Queues a Celery task (`run_workflow_task`) to execute a specified workflow.
* **`/orchestrator/workflows` Endpoint:**
    * **Response Model:** `List[WorkflowInfo]`
    * **Functionality:** Lists all available workflows.
* **`/orchestrator/job/{job_id}` Endpoint:**
    * **Functionality:** Provides status and results for a workflow execution job.

### 5.6. Dump to S3 (`/dump_to_s3`, `/dump_to_s3/job/{job_id}`)

* **`/dump_to_s3` Endpoint:**
    * **Request Model:** `DumpToS3Request` (table_names, bucket, prefix, other database and AWS parameters)
    * **Response Model:** JSON response with job ID and status.
    * **Functionality:** Queues a Celery task (`dump_to_s3_task`) to dump PostgreSQL tables to S3.  Provides a status endpoint `/dump_to_s3/job/{job_id}` to check the status.
* **`/dump_to_s3/job/{job_id}` Endpoint:**
    * **Functionality:** Returns the status and result of the dump operation.

### 5.7. S3 to PostgreSQL Migration (`/s3_to_postgres`, `/s3_to_postgres/job/{job_id}`)

* **`/s3_to_postgres` Endpoint:**
    * **Request Model:** `S3ToPostgresRequest` (s3_path, config, config_file, sport, table, match_id, other parameters)
    * **Response Model:** JSON response with job ID and status.
    * **Functionality:** Queues a Celery task (`migrate_s3_to_postgres_task`) to migrate data from S3 to PostgreSQL. Provides a status endpoint `/s3_to_postgres/job/{job_id}` to check the status and results.
* **`/s3_to_postgres/job/{job_id}` Endpoint:**
    * **Functionality:** Returns status and results of the migration task.

### 5.8. Delete from PostgreSQL (`/delete_from_postgres`)

* **Request Model:** `DeleteFromPostgresRequest` (table_name, filter_condition, truncate, dry_run, connection parameters)
* **Response Model:** `DeleteFromPostgresResponse` (success, affected_rows, query_executed, message, error)
* **Functionality:** Deletes records from a PostgreSQL table based on the provided filter condition or truncates the table.  Uses a thread pool for synchronous execution.


### 5.9. Copy S3 Data (`/copy_s3_data`)

* **Request Model:** `CopyS3DataRequest` (s3_uris, dest_folder_name)
* **Response Model:** `CopyS3DataResponse` (success, new_s3_uri, message, error)
* **Functionality:** Copies S3 objects or folders to a new timestamped folder.

### 5.10. Get Player IDs (`/get_player_ids`)

* **Request Model:** `GetPlayerIdsRequest` (sport, g_match_id, g_match_ids, config_path)
* **Response Model:** `GetPlayerIdsResponse` (success, player_ids, count, message, error)
* **Functionality:** Retrieves unique player IDs from the database for a given match or list of matches.

### 5.11. Health Check (`/health`)

A simple endpoint to check the health of the API.

### 5.12. PostgreSQL View Management (`/postgres_views/create`, `/postgres_views/drop`)

* **`/postgres_views/create` Endpoint:**
    * **Request Model:** `CreateViewRequest` (view_name, table_name, where_clause, order_by, dry_run, connection parameters)
    * **Response Model:** `ViewOperationResponse` (success, view_name, operation, query_executed, message, error)
    * **Functionality:** Creates or replaces a PostgreSQL view.
* **`/postgres_views/drop` Endpoint:**
    * **Request Model:** `DropViewRequest` (view_name, dry_run, connection parameters)
    * **Response Model:** `ViewOperationResponse`
    * **Functionality:** Drops a PostgreSQL view.

### 5.13. Get Table List (`/get_table_list`)

* **Request Model:** `GetTableListRequest` (db_schema, name_pattern, match_id, sport, table_type, min_size_mb, max_size_mb, include_stats, output_format, list_all, config_file)
* **Response Model:** `GetTableListResponse` (success, table_count, total_size_mb, total_rows, output, format, message, error)
* **Functionality:** Retrieves a list of database tables with optional filtering and statistics.

### 5.14. Pivot Table (`/pivot_table`)

* **Request Model:** `PivotTableRequest` (source_table, target_table, unpivot_columns, config_file)
* **Response Model:** `PivotTableResponse` (success, stats_count, message, error)
* **Functionality:** Transforms a wide table into a unpivoted format.

### 5.15. Merge Tables (`/merge_tables`)

* **Request Model:** `MergeTablesRequest` (main_table, add_entries_from, simple_mode)
* **Response Model:** `MergeTablesResponse` (success, initial_count, final_count, rows_added, source_tables_processed, message, error)
* **Functionality:** Merges multiple PostgreSQL tables into a main table.

### 5.16. Match Tables (`/match_tables`)

* **Request Model:** `MatchTableRequest` (match_id, table_list, action, drop_if_exists, copy_functions, config_file)
* **Response Model:** `MatchTableResponse` (success, match_id, action, created_tables, dropped_tables, existing_tables, failed_tables, verification_results, message, error)
* **Functionality:** Creates, drops, lists, or verifies match-specific tables.

### 5.17. Usability Fields (`/usability_fields`)

* **Request Model:** `UsabilityFieldsRequest` (main_table, usability_table, linking_fields, usability_fields, field_types, defaults)
* **Response Model:** `UsabilityFieldsResponse` (success, main_table_records, usability_table_records, records_updated_with_usability_data, records_with_defaults_only, created_columns, message, error)
* **Functionality:** Merges usability data into a main table.

### 5.18. Atomic Delete and Merge (`/delete_and_merge_tables`)

* **Request Model:** `DeleteAndMergeRequest` (main_table, add_entries_from, delete_condition, simple_mode, dry_run, skip_delete, excluded_columns, auto_exclude, connection parameters)
* **Response Model:** `DeleteAndMergeResponse` (success, main_table, records_deleted, records_inserted, source_tables_processed, simple_mode, skip_delete, message, error)
* **Functionality:** Atomically deletes records and merges data from source tables.

### 5.19. Season Data Processor (`/season_data_processor`, `/season_data_processor/job/{job_id}`, `/season_data_processor/status`)
* **`/season_data_processor` Endpoint:**
    * **Request Model:** `SeasonProcessorRequest` (competition_id, state_file, force)
    * **Response Model:** JSON response with job ID and status.
    * **Functionality:** Queues a Celery task (`process_season_data_task`) to process season data.
* **`/season_data_processor/job/{job_id}` Endpoint:**
    * **Functionality:** Provides status and results for a season data processing job.
* **`/season_data_processor/status` Endpoint:**
    * **Request Model:** `SeasonStatusRequest` (competition_id, state_file)
    * **Response Model:** `ProcessorStatusResponse`
    * **Functionality:** Returns current status of season data processing.

### 5.20. Single Match Processor (`/single_match_processor`, `/single_match_processor/job/{job_id}`, `/single_match_processor/status`, `/single_match_processor/list`)
* **`/single_match_processor` Endpoint:**
    * **Request Model:** `SingleMatchProcessorRequest` (match_id, competition_id, state_folder, skip_derived, force)
    * **Response Model:** JSON response with job ID and status.
    * **Functionality:** Queues a Celery task (`process_single_match_task`) to process a single match.
* **`/single_match_processor/job/{job_id}` Endpoint:**
    * **Functionality:** Provides status and results for a single match processing job.
* **`/single_match_processor/status` Endpoint:**
    * **Request Model:** `MatchStatusRequest` (match_id, state_folder)
    * **Response Model:** `ProcessorStatusResponse`
    * **Functionality:** Returns current status of single match processing.
* **`/single_match_processor/list` Endpoint:**
    * **Request Parameter:** `state_folder`
    * **Response Model:** `MatchListResponse`
    * **Functionality:** Lists all processed matches and their status.


### 5.21. Roster Processor (`/roster_processor`, `/roster_processor/job/{job_id}`)

* **`/roster_processor` Endpoint:**
    * **Request Model:** `RosterProcessorRequest` (sport, force)
    * **Response Model:** `RosterProcessorResponse`
    * **Functionality:** Queues a celery task to process roster data.
* **`/roster_processor/job/{job_id}` Endpoint:**
    * **Functionality:** Returns the status and result of the roster processing job.

## 6. Celery Tasks

The application uses Celery to handle long-running asynchronous tasks.  The following Celery tasks are defined:

* `download_genius_data_task`: Downloads Genius API data.
* `run_workflow_task`: Executes a workflow.
* `migrate_s3_to_postgres_task`: Migrates data from S3 to PostgreSQL.
* `process_single_match_task`: Processes a single match.
* `process_season_data_task`: Processes season data.
* `dump_to_s3_task`: Dumps PostgreSQL tables to S3.
* `process_roster_task`: Processes roster data


Helper functions (`_run_single_match_async`, `_run_migration_async`, `_run_season_data_async`, `_run_roster_async`) handle the creation and cleanup of database connection pools for the Celery tasks.  They provide appropriate async contexts for those operations.


## 7. Database Connections

The application uses both synchronous (`psycopg2`) and asynchronous (`asyncpg`) PostgreSQL connection pools to handle database interactions efficiently. The connection details are loaded from environment variables.  The connection pools are created during startup (`_startup`) and closed during shutdown (`_shutdown`).

## 8. Error Handling

The `handle_exception` function provides centralized error handling, converting various exception types into appropriate HTTPException responses.  Unexpected exceptions are logged with full tracebacks.

## 9. Application Startup and Shutdown

The application uses FastAPI's `on_event` decorators to manage the creation and closure of database connection pools during startup and shutdown, ensuring resources are properly released.


## 10.  Concurrency and Uvicorn Configuration

The `uvicorn.run` configuration includes settings to handle a high volume of concurrent requests:

* `workers=50`:  Spawns multiple worker processes.
* `loop="asyncio"`: Uses the `asyncio` event loop for asynchronous operations.
* `limit_concurrency=1000`: Limits the number of concurrent requests.
* `backlog=16384`:  Increases the connection backlog.
* `timeout_keep_alive=600`: Adjusts the keep-alive timeout.


This configuration allows the API to handle numerous concurrent requests effectively while maintaining the responsiveness of the application.  However, care must be taken to adjust these settings appropriately to the hardware resources available.  Over-provisioning could lead to resource exhaustion.

