# Documentation for `s3_to_postgres_migrator.py`

# s3_to_postgres_migrator.py Technical Documentation

## Overview

`s3_to_postgres_migrator.py` is a Python script designed to efficiently migrate JSON data from Amazon S3 to a PostgreSQL database.  It leverages asynchronous programming with `asyncpg` and `aioboto3` for parallel processing, significantly improving performance compared to synchronous approaches. The script supports optional complex data transformations using a user-defined helper script and offers robust error handling and logging.  It also includes features for creating and managing match-specific tables in PostgreSQL.

## Dependencies

The script relies on several key Python libraries:

* **`typer`**: For creating a command-line interface.
* **`asyncpg`**: Asynchronous PostgreSQL driver.
* **`aioboto3`**: Asynchronous AWS SDK for Boto3 (optional, used for the asynchronous version).
* **`boto3`**:  AWS SDK for Boto3 (used for the synchronous version).
* **`psycopg2`**:  PostgreSQL driver (used for the synchronous version).
* **`psycopg2.pool`**: Connection pooling for `psycopg2`.
* **`yaml`**: For parsing YAML configuration files.
* **`json`**: For handling JSON data.
* **`asyncio`**:  For asynchronous operations.
* **`multiprocessing`**: For parallel processing in the synchronous version.
* **`tqdm`**: For displaying a progress bar.
* **`dataclasses`**: For creating data classes.
* **`pathlib`**: For working with file paths.
* **`uuid`**: For generating unique identifiers.
* **`logging`**: For detailed logging of the migration process.
* **`datetime`**: For timestamping log files.
* **`importlib`**: For dynamically loading helper scripts.
* **`re`**: For regular expression operations.


## Configuration

The script uses a configuration system that can be provided via several mechanisms: a YAML file, a dictionary, or command-line parameters.  The configuration specifies the PostgreSQL connection details, the S3 path to the data, field mappings, data types, and optional settings for table creation and custom transformations. The core configuration is handled by the `MigrationConfig` dataclass.  It ensures that all necessary parameters are present and that the data types are correct.  Legacy methods for YAML and dictionary parsing are provided for backward compatibility.


### `MigrationConfig` Dataclass

This dataclass encapsulates all configuration parameters:

* `pg_host`, `pg_port`, `pg_db`, `pg_user`, `pg_password`: PostgreSQL connection parameters.
* `pg_table`: The name of the PostgreSQL table to insert data into.  Can be templated for match-specific tables.
* `field_mappings`: A dictionary mapping column names in the PostgreSQL table to fields in the JSON data (can handle nested fields with dot notation).  **Required**.
* `schema_types`: A dictionary specifying the data type of each column in the PostgreSQL table. **Required**.
* `pg_table_template`: An optional template string for creating match-specific tables (uses `match_id` as a placeholder).
* `table_creation_enabled`, `drop_if_exists`: Boolean flags controlling the automatic creation and dropping of tables.
* `helper_script`: An optional string specifying the path to a custom Python script for data transformation.  Format: "module.function".
* `batch_size`, `num_processes`: Parameters controlling the batch processing and the number of concurrent processes/workers.


### Configuration Loading Methods

Several static methods are available for creating `MigrationConfig` instances from different sources:

* `from_yaml_with_env(path, env_config)`: Loads configuration from a YAML file, using environment variables for PostgreSQL settings.
* `from_dict_with_env(data, env_config)`: Loads configuration from a dictionary, using environment variables for PostgreSQL settings.
* `from_yaml(path)`: (Legacy) Loads from YAML file.
* `from_dict(data)`: (Legacy) Loads from dictionary.


### Configuration Validation

The `validate_for_migration()` method ensures the configuration is complete and ready for the migration.


## Function Details

### `setup_logging()`

Configures logging to a file and console, creating a log file with a timestamp in the "logs" directory.  Ensures only one logger is set up.

### `PostgreSQLTableManager` Class

This class manages the creation and management of PostgreSQL tables, particularly for match-specific tables. It employs `CREATE TABLE ... LIKE INCLUDING ALL` for efficient table creation and copying all constraints and indexes.  It also creates match-specific insert functions. The class uses an external pool for acquiring connections when available, enhancing performance and robustness.

* `ensure_match_table_exists(match_id)`: Creates a match-specific table if it doesn't exist or drops and recreates if `drop_if_exists` is True.
* `_table_exists(table_name)`: Checks if a table exists.
* `_drop_table(table_name)`: Drops a table.
* `_create_match_table(target_table)`: Creates a new table using `LIKE INCLUDING ALL` copying structure from the base table.
* `_create_insert_function(conn, target_table)`: Creates a new insert function for the new match table.
* `verify_table_structure(table_name)`: Verifies the table structure and returns table metadata for debugging purposes.
* `get_table_stats(table_name)`: Gets basic table statistics (row count, size).


### `GracefulKiller` Class

Handles graceful shutdown of the script upon receiving SIGINT or SIGTERM signals.

### `ErrorLogger` Class

Logs errors to JSON files in the "error_logs" directory, managing a buffer to prevent excessive disk I/O.  Provides both synchronous and asynchronous logging methods.

### `AsyncConnectionManager` Class

Manages a pool of asynchronous PostgreSQL connections using `asyncpg`.  It uses a singleton pattern to ensure only one pool is created per connection string.

### `simple_transform(rec, field_mappings, schema_types)`

A basic data transformation function that maps JSON fields to PostgreSQL columns, handling nested fields using dot notation and converting empty strings to `None` for numeric types.

### `process_batch_async(batch, cfg)`

Processes a batch of records asynchronously. It dynamically loads a custom transformation function if specified by `cfg.helper_script` or uses the `simple_transform` function otherwise.  It then performs a bulk insert into the PostgreSQL table using a prepared statement with `asyncpg`.

### `process_batch(args)`

Sync version for backward compatibility with multiprocessing.


### `ConnectionManager` Class

Manages a pool of synchronous PostgreSQL connections using `psycopg2`.  It uses a singleton pattern to ensure only one pool is created per connection string and process ID.

### `AsyncS3Migrator` Class

The main class responsible for the asynchronous S3 to PostgreSQL migration.

* `migrate()`:  The core migration function. It retrieves data from S3, processes it in batches using asynchronous functions, and performs bulk inserts into PostgreSQL.
* `_parse_s3_path(s3_path)`: Parses an S3 URI into bucket name and prefix.


### `S3Migrator` Class

Sync version of the S3Migrator for backward compatibility.

### `load_migrator_config(config_file)`

Loads the migrator configuration from a JSON file.

### `get_config_path(sport, table, migrator_config_file)`

Looks up the configuration file path based on the sport and table name from the `migrator_config.json` file.

### `load_aws_env(env_file)`

Loads AWS environment variables from a JSON file.

### `run_migration(...)`, `run_migration_async(...)`

These functions encapsulate the core migration logic for both sync and async versions, handling configuration loading and calling the appropriate migrator class.

### `verify_database_state(cfg)`

Verifies the database table and function state, providing detailed information for debugging purposes.

## CLI Interface

The script provides a command-line interface using `typer`.  The `migrate` command performs the migration, while other commands (`validate_config`, `list_configs`, `validate_aws_env`) handle configuration validation and listing available configurations.


## Error Handling and Logging

The script incorporates comprehensive error handling and logging mechanisms.  Errors are logged to both the console and a log file, and a separate file ("error_logs") is used to store details of failed records or batches.  This facilitates debugging and troubleshooting.  A graceful shutdown mechanism is implemented to handle interruptions gracefully.



##  Improvements and Enhancements

The code is well-structured and well-commented, making it easy to understand and maintain.  The use of asynchronous programming and connection pooling significantly improves performance. The introduction of match-specific tables and their associated functions is a valuable addition, improving data organization and flexibility. The error handling and logging mechanisms are robust.  The use of dataclasses for configuration provides better type safety and organization.  The legacy methods are included to support backward compatibility, but the new methods using environment variables for database connection details are more robust.  The `_parse_s3_path` function correctly handles both `s3://bucket/prefix` and `bucket/prefix` formats.  The progress bar (`tqdm`) provides user feedback during the migration. The use of semaphores to control concurrency prevents overwhelming the database. The script includes comprehensive checks to ensure database state validity before and after the migration.



This documentation provides a comprehensive overview of the `s3_to_postgres_migrator.py` script's functionality, design, and usage.  It should be sufficient for developers to understand, modify, and maintain the script effectively.

