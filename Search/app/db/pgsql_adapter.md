# Documentation for `pgsql_adapter.py`

This document provides a comprehensive technical overview of the `PGSQLAdapter` Python script, detailing its purpose, architecture, and implementation. It covers all functions, classes, and main execution flow, along with important implementation details and dependencies.

---

## Technical Documentation: PGSQLAdapter

### 1. High-Level Overview

The `PGSQLAdapter` script serves as a robust and flexible interface for interacting with a PostgreSQL database, primarily for executing analytical queries and transforming their results. It acts as a crucial bridge between the application's data querying logic and the underlying PostgreSQL database.

Its core responsibilities include:
*   **Database Connection Management**: Utilizing a connection pool (`psycopg2.pool`) for efficient and reliable database interactions.
*   **Query Execution**: Sending SQL queries to the database and retrieving results.
*   **Result Transformation**: Mapping raw database column names to user-friendly labels for both statistical and metadata fields using configurable JSON mappings.
*   **Error Handling and Logging**: Providing robust error management and detailed logging for operations, especially in case of connection issues or query failures.
*   **Query Caching and Export Logging**: Supporting features like retrieving recent queries and logging data export actions.

The adapter is designed to be configurable for different sports and entities (e.g., "Player", "Team"), allowing for dynamic mapping of statistical and metadata fields.

### 2. Dependencies and Configuration

The script relies on several external libraries and internal modules:

*   **`psycopg2`**: The Python adapter for PostgreSQL. Used for database connectivity and operations.
*   **`decouple`**: Used for managing environment variables (e.g., database credentials) to keep sensitive information out of the codebase.
*   **`json`**: For parsing and loading JSON configuration files (e.g., stat mappings).
*   **`decimal.Decimal`**: Used for handling precise decimal values, typically returned by `psycopg2` for numeric database types.
*   **`os`**: For operating system interactions, primarily path manipulation (`os.path`).
*   **`datetime.date`**: For handling date objects.
*   **`collections.defaultdict`**: A dictionary subclass that calls a factory function to supply missing values, not explicitly used in the current version but imported.
*   **`logging`**: Python's standard logging library for outputting diagnostic messages.
*   **`time`**: For timing operations, specifically query execution duration.
*   **`contextlib.contextmanager`**: A decorator to create context managers for `with` statements, used for managing database cursors.

**Internal Module Dependencies:**
*   **`app.data_config.mappings.mappings_handler.MappingsHandler`**: Manages the loading of various mapping configurations.
*   **`app.constants.QueryTypes`, `Qualifiers`, `SortingFields`, `SortOrders`, `PostgreSQL`**: Contains application-wide constants, including database configuration keys.
*   **`app.sql_templates.sql_template_loader.SQLTemplateLoader`**: For loading SQL query templates (not directly used in the current `PGSQLAdapter` methods but implied by imports).
*   **`app.db.connection.get_db_pool`**: A centralized function to retrieve the `psycopg2` connection pool instance.

**Configuration Variables**:
Database connection parameters (database name, user, password, host, port) are expected to be configured via environment variables, accessed through `decouple.config()`. These variables are typically defined in a `.env` file in the project root.

### 3. Global Configuration and Variables

```python
import psycopg2
from decouple import config
import json
from decimal import Decimal
import os
from datetime import date
from collections import defaultdict
import logging
from app.data_config.mappings.mappings_handler import MappingsHandler
from app.constants import QueryTypes, Qualifiers, SortingFields, SortOrders, PostgreSQL
from app.sql_templates.sql_template_loader import SQLTemplateLoader
from app.db.connection import get_db_pool
import time

from contextlib import contextmanager
```
**Line 1-17**: These lines import all necessary modules and classes from Python's standard library and the application's custom modules. Each import is essential for the functionality of the `PGSQLAdapter` class, ranging from database interaction (`psycopg2`), configuration management (`decouple`), data serialization (`json`), path handling (`os`), logging (`logging`), to custom mapping and constant definitions.

```python
# Configure logging with standard format including timestamp and log level
logging.basicConfig(level=logging.INFO, format='%(asctime)s - %(levelname)s - %(message)s')
logger = logging.getLogger()
```
**Line 20-21**:
*   `logging.basicConfig(...)`: Configures the root logger.
    *   `level=logging.INFO`: Sets the minimum logging level to `INFO`, meaning `INFO`, `WARNING`, `ERROR`, and `CRITICAL` messages will be processed.
    *   `format='%(asctime)s - %(levelname)s - %(message)s'`: Defines the format for log messages, including a timestamp, the log level, and the actual message.
*   `logger = logging.getLogger()`: Retrieves the root logger instance, which will be used throughout the script for logging messages.

### 4. Function Documentation

#### `print_conns(pg_pool)`

```python
def print_conns(pg_pool):
    used_connections = len(pg_pool._used)  # connections currently checked out
    free_connections = len(pg_pool._pool)  # idle connections available
    total_connections = used_connections + free_connections

    print(f"Used connections: {used_connections}")
    print(f"Free connections: {free_connections}")
    print(f"Total connections in pool: {total_connections}")
```
*   **Purpose**: This utility function is designed to provide a snapshot of the current state of the PostgreSQL connection pool. It prints the number of connections currently in use, the number of idle connections available, and the total connections managed by the pool.
*   **Parameters**:
    *   `pg_pool` (psycopg2.pool.SimpleConnectionPool): An instance of the `psycopg2` connection pool.
*   **Return Value**: None. This function only prints information to the console.
*   **Implementation Logic**:
    1.  It accesses the internal `_used` attribute of the `pg_pool` object to count connections that have been checked out.
    2.  It accesses the internal `_pool` attribute to count connections that are currently idle and available.
    3.  It calculates the `total_connections` by summing `used_connections` and `free_connections`.
    4.  It then prints these three counts using f-strings for clear output.
*   **Important Notes**: Accessing internal attributes (like `_used` and `_pool`) is generally discouraged as they might change in future library versions, but it's a common pattern for inspecting `psycopg2` pool state in diagnostic scenarios.

### 5. Class Documentation: `PGSQLAdapter`

```python
class PGSQLAdapter:
    """
    PostgreSQL Adapter for executing analytical queries and transforming results.
    
    This adapter serves as a bridge between the application's query interface and PostgreSQL database.
    It handles database connections, query execution, and result transformation using configurable
    mappings for statistics and metadata fields.
    
    Key responsibilities:
    - Manages PostgreSQL database connection
    - Executes SQL queries and retrieves results
    - Transforms database column names to user-friendly labels
    - Handles error cases and provides detailed logging
    - Maintains separation between statistical and metadata fields
    """
```
*   **Purpose**: The `PGSQLAdapter` class encapsulates all logic related to interacting with a PostgreSQL database. Its primary goal is to simplify database operations for the rest of the application, providing a clean interface for executing queries, handling connections, and transforming raw database results into a more consumable format using predefined mappings.
*   **Key Responsibilities (as per docstring)**:
    *   **Database Connection Management**: Ensures connections are properly acquired from and returned to a connection pool.
    *   **Query Execution**: Executes SQL queries, including parameterized queries, against the database.
    *   **Result Transformation**: Converts database column names (often technical) into user-friendly labels based on application-specific mapping configurations.
    *   **Error Handling**: Catches and logs database-related errors, preventing application crashes and providing diagnostic information.
    *   **Data Separation**: Organizes query results into distinct 'metadata' and 'stats' sections, improving data structure and clarity.

#### `PGSQLAdapter.__init__(self, pg_pool=None)`

```python
    def __init__(self, pg_pool=None):
        """
        Initializes database connection and loads required mapping configurations.
        
        Establishes PostgreSQL connection using the centralized connection pool.
        Loads metadata mappings used for transforming column names in query results.
        
        Args:
            pg_pool: Optional connection pool. If not provided, uses the centralized pool.
        """
        self.conn = None
        self.cursor = None
        self.pg_pool = pg_pool
        try:
            # Use the provided pool or get the centralized one
            # self.pg_pool = pg_pool if pg_pool is not None else get_db_pool()
            # print("PostgreSQL connection pool initialized @ PGSQLAdapter::INIT")
            # self.conn = self.pg_pool.getconn()
            # self.conn.autocommit = True
            # self.cursor = self.conn.cursor()
            # logger.info("PostgreSQL Connection Established.")

            # Load Mappings 
            self.mappings_handler = MappingsHandler()
            self.metadata_mapping = self.mappings_handler.get_metadata_mapping()
        except Exception as e:
            logger.error(f"Failed to initialize database connection: {e}")
            self.pg_pool = None
            self.conn = None
            self.cursor = None
            # Don't raise here - let methods handle connection issues as needed
```
*   **Purpose**: Initializes an instance of the `PGSQLAdapter`. This involves setting up initial connection-related attributes and loading necessary data mapping configurations.
*   **Parameters**:
    *   `pg_pool` (psycopg2.pool.SimpleConnectionPool, optional): An existing connection pool instance. If `None`, the adapter *would normally* retrieve a centralized pool, but in the *current implementation*, the connection acquisition is deferred to specific methods.
*   **Return Value**: None.
*   **Implementation Logic**:
    1.  `self.conn = None`, `self.cursor = None`: Initializes `conn` (database connection) and `cursor` (database cursor) to `None`. These will be acquired from the pool later by specific methods.
    2.  `self.pg_pool = pg_pool`: Stores the provided connection pool.
    3.  **Commented-out Connection Logic**: The commented-out block (`# self.pg_pool = ...`, etc.) indicates an earlier design choice where a connection was acquired directly during initialization. This has been refactored to defer connection acquisition, likely to improve resource management and handle connections more dynamically using context managers or within individual methods.
    4.  **Mapping Loading**:
        *   `self.mappings_handler = MappingsHandler()`: Creates an instance of `MappingsHandler`, which is responsible for managing various data mapping configurations.
        *   `self.metadata_mapping = self.mappings_handler.get_metadata_mapping()`: Loads the metadata mapping. This mapping is used to transform technical database column names into more readable metadata labels.
    5.  **Error Handling**: A `try-except` block catches any `Exception` during the initialization of the `MappingsHandler` or loading of mappings. If an error occurs, it logs the error, sets `pg_pool`, `conn`, and `cursor` to `None`, and does *not* re-raise the exception, allowing the adapter to be instantiated even if initial mapping loading fails. Subsequent database operations would then need to handle the lack of a valid connection or mappings.

#### `PGSQLAdapter.close(self)`

```python
    def close(self):
        """Safely closes the PostgreSQL database connection."""
        try:
            if self.conn is not None and self.pg_pool is not None:
                self.pg_pool.putconn(self.conn)
                self.conn = None
                self.cursor = None
                print(f"Closed Adapter: PGAdapter")

        except Exception as e:
            logger.warning(f"Error closing database connection: {e}")
            # If we can't return connection normally, try to close it directly
            try:
                if self.conn is not None:
                    self.conn.close()
            except:
                pass
            self.conn = None
            self.cursor = None
```
*   **Purpose**: Safely returns the database connection (if it exists and was acquired) to the connection pool and cleans up the adapter's connection-related attributes.
*   **Parameters**: None.
*   **Return Value**: None.
*   **Implementation Logic**:
    1.  **Primary `try` block**: Attempts to return the connection to the pool.
        *   `if self.conn is not None and self.pg_pool is not None`: Checks if a connection (`self.conn`) is currently held by the adapter and if a connection pool (`self.pg_pool`) is available.
        *   `self.pg_pool.putconn(self.conn)`: Returns the connection to the `psycopg2` connection pool, making it available for reuse.
        *   `self.conn = None`, `self.cursor = None`: Resets the adapter's `conn` and `cursor` attributes to `None` to prevent accidental reuse of a returned connection.
        *   `print(f"Closed Adapter: PGAdapter")`: Prints a confirmation message.
    2.  **`except Exception as e`**: Catches any error that occurs during the process of returning the connection to the pool.
        *   `logger.warning(...)`: Logs a warning about the error.
        *   **Secondary `try-except` block**: As a fallback, if `putconn` fails, it attempts to directly close the connection using `self.conn.close()`. This ensures that even if the connection can't be returned to the pool, it's at least released at the database level to prevent resource leaks. This secondary `except` block is a bare `except` which is generally bad practice but here implies catching *any* error during `conn.close()` and silently continuing.
        *   Finally, `self.conn = None`, `self.cursor = None`: Resets the attributes regardless of success or failure in closing/returning.

#### `PGSQLAdapter.get_cursor(self)`

```python
    @contextmanager
    def get_cursor(self):
        """
        Context manager that yields a connection and cursor from the pool,
        and ensures they are returned/closed safely.
        """
        self.pg_pool = get_db_pool()
        conn = self.pg_pool.getconn()
        try:
            conn.autocommit = True
            with conn.cursor() as cursor:
                yield cursor  # <-- Use this cursor in queries
        finally:
            self.pg_pool.putconn(conn)
```
*   **Purpose**: This method acts as a context manager, simplifying the acquisition and release of a database connection and cursor from the pool. It ensures that the connection is always returned to the pool, even if errors occur during query execution, promoting resource efficiency and preventing leaks.
*   **Decorator**: `@contextmanager` from `contextlib` transforms this generator function into a context manager.
*   **Parameters**: None.
*   **Return Value**: A cursor object (implicitly, via `yield`).
*   **Implementation Logic**:
    1.  `self.pg_pool = get_db_pool()`: Retrieves the centralized PostgreSQL connection pool. This ensures the adapter always uses the active pool.
    2.  `conn = self.pg_pool.getconn()`: Acquires a database connection from the pool.
    3.  **`try` block**: Encapsulates the operations that use the connection.
        *   `conn.autocommit = True`: Sets the connection to autocommit mode. This means every statement executed will be committed immediately without requiring an explicit `conn.commit()`.
        *   `with conn.cursor() as cursor:`: Uses a nested context manager for the cursor itself. This ensures the cursor is properly closed when exiting the `with` block.
        *   `yield cursor`: The core of the context manager. It yields the `cursor` object to the caller. The code within the `with` statement (where `get_cursor()` is used) will then execute, using this `cursor`.
    4.  **`finally` block**: This block guarantees that `self.pg_pool.putconn(conn)` is executed regardless of whether the `try` block completed successfully or an exception occurred.
        *   `self.pg_pool.putconn(conn)`: Returns the acquired database connection (`conn`) back to the connection pool. This is critical for preventing connection leaks and ensuring resource availability.

#### `PGSQLAdapter.get_stat_mapping_by_position(self, sport_code, entity, player_position)`

```python
    def get_stat_mapping_by_position(self, sport_code, entity, player_position):
        stat_mapping = self.fetch_stat_mapping_R2(sport_code, entity)
        return stat_mapping.get(player_position, stat_mapping.get("ALL", {}))
```
*   **Purpose**: To retrieve statistical field mappings tailored to a specific player position for a given sport and entity. If a position-specific mapping isn't found, it falls back to a general "ALL" mapping.
*   **Parameters**:
    *   `sport_code` (str): The code identifying the sport (e.g., "MFB").
    *   `entity` (str): The type of entity (e.g., "Player" or "Team").
    *   `player_position` (str): The specific player position (e.g., "QB", "RB").
*   **Return Value**: `dict`. A dictionary representing the statistical mapping for the `player_position`, or the "ALL" mapping if the specific position is not found, or an empty dictionary if neither is found or an error occurred during `fetch_stat_mapping_R2`.
*   **Implementation Logic**:
    1.  `stat_mapping = self.fetch_stat_mapping_R2(sport_code, entity)`: Calls `fetch_stat_mapping_R2` to get the base statistical mapping dictionary, which likely includes nested mappings for different positions or categories.
    2.  `return stat_mapping.get(player_position, stat_mapping.get("ALL", {}))`: This line uses dictionary's `get` method for safe access:
        *   It first tries to retrieve the mapping for the `player_position`.
        *   If `player_position` is not found, it tries to retrieve the mapping for the key `"ALL"`.
        *   If neither `player_position` nor `"ALL"` is found, it returns an empty dictionary `{}` as a default fallback.

#### `PGSQLAdapter.fetch_stat_mapping(self, sport_code, entity, return_full_name=False)`

```python
    def fetch_stat_mapping(self, sport_code, entity,return_full_name=False):
        """
        Retrieves statistical field mappings from JSON configuration files.
        
        Loads sport and entity specific mapping configurations that define how database column names
        should be transformed into user-friendly labels in the query results.
        
        Args:
            sport_code (str): Sport identifier code (e.g. "MFB" for football)
            entity (str): Type of entity ("Player" or "Team") for which stats are being fetched
            return_full_name (bool): If True, returns full names; otherwise, returns short labels.
            
        Returns:
            dict: Mapping of database column names to their display labels
                 Empty dict if mapping file is not found or contains errors
                 
        Example mapping:
            {"rushing_yards": "RUSH_YDS", "passing_yards": "PASS_YDS"}
        """
        base_dir = os.path.abspath(os.path.dirname(__file__))
        stat_mapping_path = os.path.join(base_dir, "..", "data_config", "mappings", "stat_mapping", sport_code,
                                         f"{entity}.json")
        stat_mapping_path = os.path.normpath(stat_mapping_path)

        if not os.path.exists(stat_mapping_path):
            logger.error(f"Stat mapping file not found: {stat_mapping_path}")
            return {}

        try:
            if not os.path.exists(stat_mapping_path): # Redundant check, already done above
                logger.error(f"Stat mapping file not found: {stat_mapping_path}")
                raise FileNotFoundError(f"File not found: {stat_mapping_path}")

            with open(stat_mapping_path, "r", encoding="utf-8") as file:
                stat_data = json.load(file)
            if return_full_name:
                return {entry["stat"].lower(): entry["name"] for entry in stat_data if "name" in entry}
            return {entry["stat"].lower(): entry["short_label"] for entry in stat_data if "short_label" in entry}
        except Exception as e:
            logger.error(f"Error loading stat mapping from file {stat_mapping_path}: {e}")
            return {}
```
*   **Purpose**: This method is responsible for loading and parsing statistical field mappings from specific JSON configuration files. These files define how raw database column names should be translated into more user-friendly labels in the application's output.
*   **Parameters**:
    *   `sport_code` (str): A string representing the sport (e.g., "MFB" for Men's Football). This is used to navigate to the correct sport-specific mapping directory.
    *   `entity` (str): A string indicating the entity type (e.g., "Player" or "Team"). This specifies which entity's mapping file to load.
    *   `return_full_name` (bool, optional): If `True`, the returned mapping will use the full `name` from the JSON entries as values. If `False` (default), it will use the `short_label`.
*   **Return Value**: `dict`. A dictionary where keys are lowercase database column names and values are their corresponding display labels (either `name` or `short_label` based on `return_full_name`). Returns an empty dictionary `{}` if the mapping file is not found or if an error occurs during loading or parsing.
*   **Implementation Logic**:
    1.  **Path Construction**:
        *   `base_dir = os.path.abspath(os.path.dirname(__file__))`: Gets the absolute path of the directory containing the current script.
        *   `stat_mapping_path = os.path.join(base_dir, "..", "data_config", "mappings", "stat_mapping", sport_code, f"{entity}.json")`: Constructs the absolute path to the specific stat mapping JSON file. The path traverses up one directory (`..`) from the current script to find the `data_config` directory.
        *   `stat_mapping_path = os.path.normpath(stat_mapping_path)`: Normalizes the path, removing redundant separators and resolving `..`.
    2.  **File Existence Check**:
        *   `if not os.path.exists(stat_mapping_path):`: Checks if the constructed file path actually points to an existing file. If not, it logs an error and returns an empty dictionary.
    3.  **Loading and Parsing**:
        *   **Redundant Check**: The `try` block contains another `if not os.path.exists(...)` check which is redundant as it's already performed just before the `try` block. If the file doesn't exist, it would raise `FileNotFoundError`.
        *   `with open(stat_mapping_path, "r", encoding="utf-8") as file:`: Opens the JSON file for reading with UTF-8 encoding.
        *   `stat_data = json.load(file)`: Parses the JSON file content into a Python list of dictionaries.
        *   **Mapping Creation**:
            *   `if return_full_name:`: If `return_full_name` is `True`, it generates a dictionary mapping `entry["stat"].lower()` (database column name) to `entry["name"]` (full label) for each entry that contains a "name" key.
            *   `else:`: Otherwise, it maps `entry["stat"].lower()` to `entry["short_label"]` (short label) for each entry containing a "short_label" key.
    4.  **Error Handling**: Catches any `Exception` that might occur during file operations (e.g., `FileNotFoundError` if the redundant check is reached, `json.JSONDecodeError` if the file is malformed, etc.), logs the error, and returns an empty dictionary.

#### `PGSQLAdapter.fetch_stat_to_short_name(self, sport_code, entity)`

```python
    def fetch_stat_to_short_name(self, sport_code, entity):
        """
        Retrieves statistical field mappings from JSON configuration files.

        Loads sport and entity specific mapping configurations that define how database column names
        should be transformed into user-friendly labels in the query results.

        Args:
            sport_code (str): Sport identifier code (e.g. "MFB" for football)
            entity (str): Type of entity ("Player" or "Team") for which stats are being fetched

        Returns:
            dict: Mapping of database column names to their display labels
                 Empty dict if mapping file is not found or contains errors

        Example mapping:
               {"sFieldGoalAttemptLongest": "FGA Long","sFieldGoalsMade": "FGM",}
        """
        base_dir = os.path.abspath(os.path.dirname(__file__))
        stat_mapping_path = os.path.join(base_dir, "..", "data_config", "mappings", "stat_mapping", sport_code,
                                         "stats_to_shortname", f"{entity}.json")
        stat_mapping_path = os.path.normpath(stat_mapping_path)
        # print("mapping path", stat_mapping_path)


        if not os.path.exists(stat_mapping_path):
            logger.error(f"Stat mapping file not found: {stat_mapping_path}")
            return {}

        try:
            if not os.path.exists(stat_mapping_path): # Redundant check
                logger.error(f"Stat mapping file not found: {stat_mapping_path}")
                raise FileNotFoundError(f"File not found: {stat_mapping_path}")

            with open(stat_mapping_path, "r", encoding="utf-8") as file:
                stat_data = json.load(file)
                # print("stat data", stat_data)
            return stat_data
        except Exception as e:
            logger.error(f"Error loading stat mapping from file {stat_mapping_path}: {e}")
            return {}
```
*   **Purpose**: Similar to `fetch_stat_mapping`, but specifically designed to fetch a mapping from database column names to short display names from a dedicated `stats_to_shortname` directory.
*   **Parameters**:
    *   `sport_code` (str): Sport identifier code.
    *   `entity` (str): Type of entity ("Player" or "Team").
*   **Return Value**: `dict`. A dictionary representing the mapping from stat names to their short display names. Returns an empty dictionary `{}` if the file is not found or an error occurs.
*   **Implementation Logic**:
    1.  **Path Construction**: Constructs the `stat_mapping_path` to point to a JSON file within the `stat_mapping/<sport_code>/stats_to_shortname/` directory, named after the `entity`.
    2.  **File Existence Check**: Checks if the file exists. Logs an error and returns an empty dict if not found.
    3.  **Loading and Parsing**: Opens and loads the JSON file.
        *   Again, there's a redundant `os.path.exists` check within the `try` block.
        *   Unlike `fetch_stat_mapping`, this method directly returns the `json.load(file)` result, implying the JSON file itself is already structured as the desired dictionary mapping.
    4.  **Error Handling**: Catches any `Exception`, logs it, and returns an empty dictionary.

#### `PGSQLAdapter.fetch_stat_mapping_R2(self, sport_code, entity)`

```python
    def fetch_stat_mapping_R2(self, sport_code, entity):
        """
        Retrieves statistical field mappings from JSON configuration files.
        
        Loads sport and entity specific mapping configurations that define how database column names
        should be transformed into user-friendly labels in the query results.
        
        Args:
            sport_code (str): Sport identifier code (e.g. "MFB" for football)
            entity (str): Type of entity ("Player" or "Team") for which stats are being fetched
            
        Returns:
            dict: Mapping of database column names to their display labels
                 Empty dict if mapping file is not found or contains errors
                 
        Example mapping:
            {"rushing_yards": "RUSH_YDS", "passing_yards": "PASS_YDS"}
        """
        base_dir = os.path.abspath(os.path.dirname(__file__))
        stat_mapping_path = os.path.join(base_dir, "..", "data_config", "mappings", "stat_mapping", sport_code, "category_to_stats", f"{entity}.json")
        stat_mapping_path = os.path.normpath(stat_mapping_path)

        if not os.path.exists(stat_mapping_path):
            logger.error(f"Stat mapping file not found: {stat_mapping_path}")
            return {}

        try:
            if not os.path.exists(stat_mapping_path): # Redundant check
                logger.error(f"Stat mapping file not found: {stat_mapping_path}")
                raise FileNotFoundError(f"File not found: {stat_mapping_path}")

            with open(stat_mapping_path, "r", encoding="utf-8") as file:
                stat_data = json.load(file)
            return stat_data
        except Exception as e:
            logger.error(f"Error loading stat mapping from file {stat_mapping_path}: {e}")
            return {}
```
*   **Purpose**: This method is another variant for fetching statistical mappings, specifically from the `category_to_stats` subdirectory. It is structurally very similar to `fetch_stat_to_short_name` and `fetch_stat_mapping`, but targets a different file location. It's likely intended for a mapping that groups stats by category or position before providing the final stat-to-label mapping.
*   **Parameters**:
    *   `sport_code` (str): Sport identifier code.
    *   `entity` (str): Type of entity.
*   **Return Value**: `dict`. The loaded JSON data directly. Returns an empty dictionary `{}` on error or file not found.
*   **Implementation Logic**:
    1.  **Path Construction**: Constructs the `stat_mapping_path` to target the `stat_mapping/<sport_code>/category_to_stats/` directory.
    2.  **File Existence Check**: Performs an initial check for file existence.
    3.  **Loading and Parsing**: Inside a `try-except` block, it re-checks file existence (redundant), opens, and loads the JSON file, then returns the entire JSON content.
    4.  **Error Handling**: Catches exceptions, logs them, and returns an empty dictionary.

#### `PGSQLAdapter.fetch_game_categories(self, sport_code)`

```python
    def fetch_game_categories(self, sport_code):
        base_dir = os.path.abspath(os.path.dirname(__file__))
        stat_mapping_path = os.path.join(base_dir, "..", "data_config", "mappings", "stat_mapping", sport_code, "game_categories.json")
        stat_mapping_path = os.path.normpath(stat_mapping_path)

        if not os.path.exists(stat_mapping_path):
            logger.error(f"Stat mapping file not found: {stat_mapping_path}")
            return {}

        try:
            if not os.path.exists(stat_mapping_path): # Redundant check
                logger.error(f"Stat mapping file not found: {stat_mapping_path}")
                raise FileNotFoundError(f"File not found: {stat_mapping_path}")

            with open(stat_mapping_path, "r", encoding="utf-8") as file:
                stat_data = json.load(file)
            return stat_data
        except Exception as e:
            logger.error(f"Error loading stat mapping from file {stat_mapping_path}: {e}")
            return {}
```
*   **Purpose**: To retrieve game category definitions from a JSON file specific to a given sport.
*   **Parameters**:
    *   `sport_code` (str): The code identifying the sport.
*   **Return Value**: `dict`. The loaded JSON data (game categories). Returns an empty dictionary `{}` on error or file not found.
*   **Implementation Logic**:
    1.  **Path Construction**: Constructs the `stat_mapping_path` to `stat_mapping/<sport_code>/game_categories.json`.
    2.  **File Existence Check**: Performs an initial check.
    3.  **Loading and Parsing**: Inside a `try-except` block, it re-checks (redundantly), opens, and loads the JSON file, then returns the entire JSON content.
    4.  **Error Handling**: Catches exceptions, logs them, and returns an empty dictionary.

#### `PGSQLAdapter.fetch_streaks_mapping(self, sport_code, entity)`

```python
    def fetch_streaks_mapping(self, sport_code, entity):
        """
        Retrieves statistical field mappings from JSON configuration files.

        Loads sport and entity specific mapping configurations that define how database column names
        should be transformed into user-friendly labels in the query results.

        Args:
            sport_code (str): Sport identifier code (e.g. "MFB" for football)
            entity (str): Type of entity ("Player" or "Team") for which stats are being fetched

        """
        base_dir = os.path.abspath(os.path.dirname(__file__))
        stat_mapping_path = os.path.join(base_dir, "..", "data_config", "mappings", "stat_mapping", sport_code, "streaks_mapping", f"{entity}.json")
        stat_mapping_path = os.path.normpath(stat_mapping_path)

        if not os.path.exists(stat_mapping_path):
            logger.error(f"Stat mapping file not found: {stat_mapping_path}")
            return {}

        try:
            if not os.path.exists(stat_mapping_path): # Redundant check
                logger.error(f"Stat mapping file not found: {stat_mapping_path}")
                raise FileNotFoundError(f"File not found: {stat_mapping_path}")

            with open(stat_mapping_path, "r", encoding="utf-8") as file:
                stat_data = json.load(file)
            return stat_data
        except Exception as e:
            logger.error(f"Error loading stat mapping from file {stat_mapping_path}: {e}")
            return {}
```
*   **Purpose**: To retrieve specific mappings related to "streaks" from JSON configuration files, similar to other `fetch_` methods but targeting a different subdirectory.
*   **Parameters**:
    *   `sport_code` (str): Sport identifier code.
    *   `entity` (str): Type of entity ("Player" or "Team").
*   **Return Value**: `dict`. The loaded JSON data, which contains streak-related mappings. Returns an empty dictionary `{}` on error or file not found.
*   **Implementation Logic**:
    1.  **Path Construction**: Constructs the `stat_mapping_path` to target `stat_mapping/<sport_code>/streaks_mapping/` directory.
    2.  **File Existence Check**: Performs an initial check.
    3.  **Loading and Parsing**: Inside a `try-except` block, it re-checks (redundantly), opens, and loads the JSON file, then returns the entire JSON content.
    4.  **Error Handling**: Catches exceptions, logs them, and returns an empty dictionary.

#### `PGSQLAdapter.execute_query(self, sql_query, selected_columns, sport_code, entity, stat_columns, aql_only=False)`

```python
    def execute_query(self, sql_query, selected_columns, sport_code, entity, stat_columns, aql_only=False):
        """
        Executes SQL query and formats results using configured mappings.
        
        This method handles the complete query execution workflow:
        1. Executes the main SQL query to fetch results
        2. Maps database columns to appropriate display labels using sport-specific mappings
        3. Separates results into metadata and statistical components
        4. Executes count query to get total number of matching records (this functionality seems merged with main query now)
        
        Args:
            sql_query (str): Main SQL query to execute
            selected_columns (list): List of columns to include in metadata section (Note: logic seems to use all selected columns as potential metadata, or uses mapping directly)
            sport_code (str): Sport identifier code for loading correct mappings
            entity (str): Entity type ('Player' or 'Team') for stat mapping
            stat_columns (list): List of columns to be treated as statistics
            aql_only (bool, optional): If True, returns empty results without executing query. Defaults to False.
            
        Returns:
            tuple: (results, total_count) where:
                  - results: List of dicts with 'metadata' and 'stats' sections
                  - total_count: Total number of records matching query criteria
        """
        if aql_only:
            logger.info("Skipping SQL execution, returning only AQL output.")
            return [], 0

        # Check if connection is available
        if self.conn is None or self.cursor is None:
            try:
                # Try to get a new connection
                self.pg_pool = get_db_pool()
                print("PostgreSQL connection pool initialized @ PGSQLAdapter::EQ")
                print_conns(self.pg_pool)
                self.conn = self.pg_pool.getconn()
                self.conn.autocommit = True
                self.cursor = self.conn.cursor()
                logger.info("Re-established PostgreSQL connection.")
            except Exception as e:
                logger.error(f"Failed to re-establish database connection: {e}")
                return [], 0

        max_retries = 2
        retry_count = 0
        
        while retry_count <= max_retries:
            try:
                logger.info("Executing SQL Query:\n" + sql_query)
                s = time.time()
                self.cursor.execute(sql_query)
                e = time.time()

                rows = self.cursor.fetchall()

                logger.info(f"Query execution took {e-s:.2f} seconds")

                column_names = [desc[0].lower() for desc in self.cursor.description]
                stat_mapping = self.fetch_stat_mapping(sport_code, entity)

                results = []
                total_count = 0

                for row_num, row in enumerate(rows):
                    row_dict = {column_names[i]: value for i, value in enumerate(row)}

                    if row_num == 0:
                        total_count = row_dict.get("total_count", 0)
                        

                    metadata = {"ID": row_dict.get("id", None)}
                    stats = {}
                    for key, value in row_dict.items():
                        key_lower = key.lower()

                        if key_lower == "total_count":
                            continue

                        if key_lower in stat_mapping:
                            mapped_key = stat_mapping[key_lower]
                            stats[mapped_key] = value
                        else:
                            mapped_key = self.metadata_mapping.get(key_lower, key_lower).upper()
                            metadata[mapped_key.upper()] = value
                            
                    # This block re-initializes and re-populates stats, overriding the previous loop's stats.
                    # This suggests the previous stats population logic (lines 145-152) might be partially or fully
                    # redundant if stat_columns is the definitive source for stats.
                    stats = {} 
                    for key in stat_columns:
                        key_lower = key.lower()
                        if key_lower in row_dict:
                            mapped_key = stat_mapping.get(key_lower, None)
                            if mapped_key:
                                stats[mapped_key.upper()] = row_dict[key_lower]

                    results.append({"metadata": metadata, "stats": stats})

                # Successfully completed query
                return results, total_count

            # except psycopg2.OperationalError as e:
            #     # Connection issues - try to recover
            #     retry_count += 1
            #     logger.warning(f"Database connection error (attempt {retry_count}/{max_retries}): {e}")
            #
            #     if retry_count <= max_retries:
            #         # Try to clean up and get a new connection
            #         try:
            #             self.close()  # Will handle None conn/cursor
            #         except:
            #             pass
            #
            #         try:
            #             # Get a fresh connection
            #             self.pg_pool = get_db_pool()
            #             print("PostgreSQL connection pool initialized @ PGSQLAdapter::EQ2")
            #
            #             self.conn = self.pg_pool.getconn()
            #             self.conn.autocommit = True
            #             self.cursor = self.conn.cursor()
            #             logger.info(f"Re-established PostgreSQL connection on retry {retry_count}")
            #         except Exception as retry_error:
            #             logger.error(f"Failed to re-establish connection on retry: {retry_error}")
            #     else:
            #         logger.error(f"Exceeded maximum retry attempts ({max_retries}) for database operation")
            #         return [], 0
                    
            except Exception as e:
                # Other errors - log and return empty results
                logger.error(f"Database Query Error: {e}")
                return [], 0
            finally:
                print("PostgreSQL connection pool closed @ PGSQLAdapter::EQ")

                self.pg_pool.putconn(self.conn)
                print_conns(self.pg_pool)

                self.conn = None
                self.cursor = None
```
*   **Purpose**: This is the core method for executing arbitrary SQL queries, fetching the results, and transforming them into a structured format (metadata and statistics) using predefined mappings. It also handles connection re-establishment and error logging.
*   **Parameters**:
    *   `sql_query` (str): The main SQL query string to be executed.
    *   `selected_columns` (list): A list of column names that are expected in the query results. This parameter's usage is somewhat inconsistent with the actual parsing logic.
    *   `sport_code` (str): The sport code for fetching relevant stat mappings.
    *   `entity` (str): The entity type ("Player" or "Team") for fetching relevant stat mappings.
    *   `stat_columns` (list): A list of column names that should be specifically treated as statistical values.
    *   `aql_only` (bool, optional): If `True`, the method bypasses actual database query execution and immediately returns an empty result set and a total count of 0. This is typically used when the application logic needs to process AQL (Analytical Query Language) input without needing to hit the database for results. Defaults to `False`.
*   **Return Value**: `tuple`. A tuple containing:
    *   `results` (list): A list of dictionaries, where each dictionary represents a row and contains two keys: `'metadata'` (a dictionary of general information) and `'stats'` (a dictionary of statistical values).
    *   `total_count` (int): The total count of records matching the query criteria, usually extracted from a `total_count` column present in the first row of the query result.
*   **Implementation Logic**:
    1.  **AQL Only Short-Circuit**: If `aql_only` is `True`, it logs this and immediately returns `([], 0)`.
    2.  **Connection Re-establishment**:
        *   `if self.conn is None or self.cursor is None:`: Checks if the adapter's connection or cursor is not active.
        *   **`try-except` block**: Attempts to get a new connection from `get_db_pool()`, sets `autocommit=True`, and creates a new cursor. Logs success or failure. If re-establishment fails, it returns `([], 0)`. This ensures that the method can attempt to recover from lost connections or lazy-initialize if not done in `__init__`.
    3.  **Retry Loop**:
        *   `max_retries = 2`, `retry_count = 0`: Initializes variables for a retry mechanism.
        *   `while retry_count <= max_retries:`: The main loop for executing the query, allowing retries in case of transient database errors.
    4.  **Query Execution (`try` block)**:
        *   `logger.info(...)`: Logs the SQL query being executed.
        *   `s = time.time()`: Starts a timer.
        *   `self.cursor.execute(sql_query)`: Executes the main SQL query against the database.
        *   `e = time.time()`: Ends the timer.
        *   `rows = self.cursor.fetchall()`: Fetches all results from the executed query.
        *   `logger.info(...)`: Logs the query execution time.
        *   `column_names = [desc[0].lower() for desc in self.cursor.description]`: Extracts column names from the cursor's description and converts them to lowercase.
        *   `stat_mapping = self.fetch_stat_mapping(sport_code, entity)`: Loads the sport and entity-specific statistical mapping.
        *   `results = []`, `total_count = 0`: Initializes lists and counters.
    5.  **Result Processing Loop**:
        *   `for row_num, row in enumerate(rows):`: Iterates through each fetched row.
        *   `row_dict = {column_names[i]: value for i, value in enumerate(row)}`: Converts the current row (tuple) into a dictionary mapping column names to their values.
        *   **Total Count Extraction**: `if row_num == 0: total_count = row_dict.get("total_count", 0)`: Assumes the `total_count` is present in a column named "total_count" within the *first* row of the result set.
        *   **Metadata Initialization**: `metadata = {"ID": row_dict.get("id", None)}`: Initializes the metadata dictionary, always including an "ID" field if available.
        *   **Initial Stat/Metadata Population (Lines 145-152)**: This loop iterates through all items in `row_dict`.
            *   If `key_lower == "total_count"`, it skips.
            *   If `key_lower` is found in `stat_mapping`, it's considered a statistic and added to `stats` with its mapped key.
            *   Otherwise, it's considered metadata and added to `metadata` with its mapped key (from `self.metadata_mapping`) or raw key (if not found in `metadata_mapping`).
        *   **Explicit Stat Population (Lines 156-162)**: This block **re-initializes** `stats = {}` and then populates `stats` *only* with columns explicitly listed in `stat_columns`. This means the previous `for key, value in row_dict.items():` loop's `stats` population logic (lines 148-150) is effectively overwritten and might be redundant or indicative of evolving requirements.
            *   For each `key` in `stat_columns`:
                *   It checks if `key_lower` exists in `row_dict`.
                *   If a `mapped_key` exists in `stat_mapping`, it's added to `stats`.
        *   `results.append({"metadata": metadata, "stats": stats})`: Appends the structured row to the `results` list.
    6.  **Return Value**: If the query executes successfully, it returns `(results, total_count)`.
    7.  **Error Handling (`except Exception as e`)**:
        *   **Commented-out `psycopg2.OperationalError` handling**: There was a detailed retry mechanism specifically for `psycopg2.OperationalError` (connection issues), which included closing the current connection and attempting to re-establish a new one from the pool. This block is currently commented out, meaning `OperationalError` will now fall into the general `Exception` block.
        *   `except Exception as e:`: Catches any other exception during query execution. It logs the error and returns `([], 0)`.
    8.  **`finally` block**: This block ensures that the connection is always returned to the pool, regardless of success or failure.
        *   `self.pg_pool.putconn(self.conn)`: Returns the connection to the pool.
        *   `print_conns(self.pg_pool)`: Prints the connection pool status.
        *   `self.conn = None`, `self.cursor = None`: Resets the adapter's connection attributes.

#### `PGSQLAdapter.execute_export_log(self, username, sport_code, entity, query_input, row_count)`

```python
    def execute_export_log(self, username, sport_code, entity, query_input, row_count):
        """
        Logs an export operation to the export_logs table.
        
        Args:
            username (str): Username of the user performing the export
            sport_code (str): Sport code for the exported data
            entity (str): Entity type (Player or Team)
            query (str): The query that was exported (parameter named `query_input` in implementation)
            row_count (int): Number of rows exported
            
        Returns:
            dict: Status of the operation
        """
        try:
            # Ensure we have a valid connection
            if not self.conn or not self.cursor:
                self.pg_pool = get_db_pool()
                print("PostgreSQL connection pool initialized @ PGSQLAdapter::EXPORT")

                self.conn = self.pg_pool.getconn()
                self.conn.autocommit = True
                self.cursor = self.conn.cursor()
            
            # Insert the export log record
            query = "INSERT INTO export_logs (username, sport_code, entity, query, row_count) VALUES (%s, %s, %s, %s, %s) RETURNING id"
            self.cursor.execute(query, (username, sport_code, entity,query_input, row_count))
            log_id = self.cursor.fetchone()[0]
            
            logger.info(f"Export log recorded with ID {log_id} for user {username}")
            return {"success": True, "log_id": log_id}
            
        except Exception as e:
            logger.error(f"Error logging export operation: {e}")
            return {"success": False, "error": str(e)}
        finally:
            print("PostgreSQL connection pool closed @ PGSQLAdapter::EXPORT")

            self.pg_pool.putconn(self.conn)
            self.conn = None
            self.cursor = None
```
*   **Purpose**: To record details of a data export operation into a `export_logs` table in the database.
*   **Parameters**:
    *   `username` (str): The user who initiated the export.
    *   `sport_code` (str): The sport code associated with the exported data.
    *   `entity` (str): The entity type (e.g., "Player", "Team") of the exported data.
    *   `query_input` (str): The actual query string or description of the query that generated the exported data.
    *   `row_count` (int): The number of rows included in the export.
*   **Return Value**: `dict`. A dictionary indicating the success (`{"success": True, "log_id": log_id}`) or failure (`{"success": False, "error": str(e)}`) of the logging operation.
*   **Implementation Logic**:
    1.  **`try` block**: Attempts the logging operation.
        *   **Connection Check**: `if not self.conn or not self.cursor:`: Similar to `execute_query`, it ensures an active connection exists. If not, it acquires one from the pool.
        *   **SQL Insertion**:
            *   `query = "INSERT INTO export_logs (username, sport_code, entity, query, row_count) VALUES (%s, %s, %s, %s, %s) RETURNING id"`: Defines an SQL `INSERT` statement to add a new record to the `export_logs` table. `RETURNING id` is used to get the ID of the newly inserted row.
            *   `self.cursor.execute(query, (username, sport_code, entity,query_input, row_count))`: Executes the insert query with the provided parameters.
            *   `log_id = self.cursor.fetchone()[0]`: Retrieves the returned `id` of the newly created log entry.
        *   `logger.info(...)`: Logs the successful recording of the export.
        *   `return {"success": True, "log_id": log_id}`: Returns success status with the log ID.
    2.  **`except Exception as e`**: Catches any exception during the process, logs it as an error, and returns a failure status dictionary.
    3.  **`finally` block**: Ensures the connection is returned to the pool and local connection attributes are reset, similar to `execute_query`.

#### `PGSQLAdapter.get_recent_queries(self, username, limit, sport_code, entity)`

```python
    def get_recent_queries(self, username, limit, sport_code, entity):
        """
        Get recent queries with proper connection pool handling and retry logic.
        """
        max_retries = 2
        retry_count = 0
        
        while retry_count <= max_retries:
            try:
                # Ensure we have a valid connection
                if self.conn is None or self.cursor is None:
                    self.pg_pool = get_db_pool()
                    print("PostgreSQL connection pool initialized @ PGSQLAdapter::RQ")

                    self.conn = self.pg_pool.getconn()
                    self.conn.autocommit = True
                    self.cursor = self.conn.cursor()
                
                query = """
                    SELECT * FROM cache_queries 
                    WHERE user_id = %s AND sport_code = %s AND entity = %s 
                    ORDER BY createdon DESC 
                    LIMIT %s
                """
                self.cursor.execute(query, (username, sport_code, entity, limit))
                rows = self.cursor.fetchall()

                if not rows:
                    return []
                
                column_names = [desc[0].lower() for desc in self.cursor.description]
                result = [dict(zip(column_names, row)) for row in rows]
                return result
                
            except psycopg2.OperationalError as e:
                # Connection issues - try to recover
                retry_count += 1
                logger.warning(f"Database connection error in get_recent_queries (attempt {retry_count}/{max_retries}): {e}")
                
                if retry_count <= max_retries:
                    # Clean up and get new connection
                    try:
                        self.close()
                    except:
                        pass
                        
                    try:
                        self.pg_pool = get_db_pool()
                        print("PostgreSQL connection pool initialized @ PGSQLAdapter::RQ")

                        self.conn = self.pg_pool.getconn()
                        self.conn.autocommit = True
                        self.cursor = self.conn.cursor()
                        logger.info(f"Re-established connection on retry {retry_count}")
                    except Exception as retry_error:
                        logger.error(f"Failed to re-establish connection: {retry_error}")
                else:
                    logger.error(f"Exceeded maximum retry attempts for get_recent_queries")
                    return []
                    
            except Exception as e:
                logger.error(f"Error in get_recent_queries: {str(e)}", exc_info=True)
                return []
            # This method lacks a `finally` block to `putconn`. This is a critical omission.
            # The connection obtained via `self.pg_pool.getconn()` is never returned to the pool
            # unless an OperationalError occurs, which then attempts to call self.close().
            # If no OperationalError and no other exception, the connection is leaked.
```
*   **Purpose**: To retrieve a limited number of the most recent cached queries for a specific user, sport, and entity from the `cache_queries` table. It includes retry logic for database connection issues.
*   **Parameters**:
    *   `username` (str): The ID of the user whose queries are to be fetched.
    *   `limit` (int): The maximum number of recent queries to retrieve.
    *   `sport_code` (str): The sport code associated with the cached queries.
    *   `entity` (str): The entity type associated with the cached queries.
*   **Return Value**: `list`. A list of dictionaries, where each dictionary represents a cached query record. Returns an empty list `[]` if no queries are found, or if an error occurs.
*   **Implementation Logic**:
    1.  **Retry Loop**:
        *   `max_retries = 2`, `retry_count = 0`: Initializes variables for retry logic.
        *   `while retry_count <= max_retries:`: Loops to attempt query execution with retries.
    2.  **`try` block (Query Execution)**:
        *   **Connection Check**: `if self.conn is None or self.cursor is None:`: Acquires a new connection from the pool if the current one is not valid, setting `autocommit=True`.
        *   **SQL Query**:
            *   `query = """ SELECT * FROM cache_queries WHERE user_id = %s AND sport_code = %s AND entity = %s ORDER BY createdon DESC LIMIT %s """`: Defines the SQL `SELECT` query to fetch recent queries, ordered by creation date in descending order and limited by `limit`.
            *   `self.cursor.execute(query, (username, sport_code, entity, limit))`: Executes the query with parameters.
            *   `rows = self.cursor.fetchall()`: Fetches all results.
        *   **Result Formatting**:
            *   `if not rows: return []`: If no rows are returned, immediately returns an empty list.
            *   `column_names = [desc[0].lower() for desc in self.cursor.description]`: Extracts column names from the cursor description.
            *   `result = [dict(zip(column_names, row)) for row in rows]`: Converts each row into a dictionary using `zip` to pair column names with values.
        *   `return result`: Returns the formatted list of queries.
    3.  **`except psycopg2.OperationalError as e` (Connection Recovery)**:
        *   This specific `except` block handles `psycopg2.OperationalError`, indicating connection issues.
        *   It increments `retry_count` and logs a warning.
        *   If `retry_count` is within `max_retries`:
            *   It attempts to `self.close()` the current connection (which puts it back to the pool).
            *   It then attempts to acquire a fresh connection from the pool, sets `autocommit=True`, and creates a new cursor. Logs the re-establishment.
        *   If `max_retries` are exceeded, it logs a final error and returns an empty list.
    4.  **`except Exception as e` (General Error)**:
        *   Catches any other general exception, logs it with traceback information (`exc_info=True`), and returns an empty list.
*   **Critical Omission**: This method **lacks a `finally` block** to ensure `self.pg_pool.putconn(self.conn)` is called when the `try` block completes successfully (i.e., no exceptions or `psycopg2.OperationalError` leading to connection re-establishment). If a query succeeds without connection issues, the connection obtained via `self.pg_pool.getconn()` is *not* returned to the pool, leading to connection leaks over time.

### 6. Main Execution Flow (`if __name__ == "__main__":`)

```python
#
# # === Testing PGSQLAdapter for Basic Queries ===
# from psycopg2 import pool
# from decouple import config
# from app.constants import PostgreSQL
#
# # === Testing PGSQLAdapter for Basic Queries ===
# if __name__ == "__main__":
#     # Set up the PostgreSQL connection pool
#     pg_pool = pool.SimpleConnectionPool(
#         minconn=1,
#         maxconn=5,
#         database=config(PostgreSQL.PG_DB.value),
#         user=config(PostgreSQL.PG_USER.value),
#         password=config(PostgreSQL.PG_PASSWORD.value),
#         host=config(PostgreSQL.PG_HOST.value),
#         port=config(PostgreSQL.PG_PORT.value),
#     )
#
#     adapter = PGSQLAdapter(pg_pool)
#
#     # Test Query for "basic"
#     test_sql = """
#     SELECT playername, teamname, position, gamedate, opponentteamname, gameresult, rushing_yards, touchdowns
#     FROM player_game_statistics
#     WHERE rushing_yards > 50 AND teamname = 'Alabama' AND season = '2022'
#     ORDER BY playername DESC
#     LIMIT 10 OFFSET 0;
#     """
#
#     test_count_sql = """
#     SELECT COUNT(*)
#     FROM player_game_statistics
#     WHERE rushing_yards > 50 AND teamname = 'Alabama' AND season = '2022';
#     """
#
#     # Example selected columns (should match actual columns in query)
#     test_columns = ["playername", "teamname", "position", "gamedate", "opponentteamname", "gameresult", "rushing_yards", "touchdowns"]
#
#     # Run the test
#     results, count = adapter.execute_query(test_sql, test_count_sql, test_columns, sport_code="MFB", entity="Player", stat_columns=["rushing_yards", "touchdowns"])
#
#     print("\n=== Query Results ===")
#     print(results)
#     print(f"\nTotal Count: {count}")
#
#     # Close the DB connection
#     adapter.close()
```
*   **Purpose**: This block contains commented-out code that demonstrates how to initialize and use the `PGSQLAdapter` for basic query testing. It's a self-contained example for local development and verification.
*   **Implementation Logic**:
    1.  **Connection Pool Setup**: It imports necessary components (`pool`, `config`, `PostgreSQL`) and then initializes a `psycopg2.pool.SimpleConnectionPool`.
        *   `minconn=1`, `maxconn=5`: Configures the pool to maintain between 1 and 5 active connections.
        *   Database credentials are loaded from environment variables using `decouple.config()` and `app.constants.PostgreSQL` enums.
    2.  **Adapter Instantiation**: An instance of `PGSQLAdapter` is created, passing the configured `pg_pool`.
    3.  **Test Queries**:
        *   `test_sql`: A sample SQL `SELECT` query targeting `player_game_statistics` table, fetching specific columns for players with rushing yards > 50 from 'Alabama' in '2022', ordered and limited.
        *   `test_count_sql`: A separate `COUNT(*)` query for the same criteria, intended to get the total number of matching records. *Note: The `execute_query` method's current implementation expects `total_count` to be part of the main `sql_query` results, not a separate `count_query` parameter, indicating this test setup might be slightly outdated relative to the method's signature.*
    4.  **Test Columns**: `test_columns` is a list defining the expected columns.
    5.  **Query Execution**: `adapter.execute_query()` is called with the test SQL, selected columns, sport code ("MFB"), entity ("Player"), and specific stat columns.
    6.  **Result Output**: The fetched `results` (structured metadata and stats) and `count` are printed to the console.
    7.  **Connection Closure**: `adapter.close()` is called to return the acquired connection to the pool.

This commented-out section provides a clear usage example of the `PGSQLAdapter` and its core `execute_query` method, demonstrating how to set up the environment, perform a query, and process its results.
