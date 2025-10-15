# Documentation for `connection.py`

This document provides comprehensive technical documentation for the `connection_manager.py` module, designed for software engineers. It covers the module's architecture, functionality, dependencies, and implementation details.

---

## Database Connection Management Module Documentation

### 1. High-Level Overview

The `connection_manager.py` module is a core component of the R2 API, dedicated to centralizing and robustly managing PostgreSQL database connections. Its primary purpose is to provide a reliable, shared connection pool across the entire application, mitigating common database-related issues such as connection resource exhaustion and temporary connection failures.

Key features include:
*   **Singleton Connection Pool**: Ensures only one instance of the connection pool exists, promoting efficient resource usage.
*   **Retry Logic**: Implements automatic retries with exponential backoff for database connection attempts, improving resilience against transient network or database issues.
*   **Connection Validation**: Tests returned connections before handing them out to ensure they are active.
*   **Safe Connection Handling**: Provides mechanisms to safely acquire and release connections back to the pool, including handling of potentially bad connections.
*   **Centralized Configuration**: Utilizes `decouple` for externalizing database credentials, enhancing security and configurability.

### 2. Dependencies

The module relies on the following Python libraries and internal modules:

*   **`time`**: Standard library module used for introducing delays (e.g., in retry logic).
*   **`logging`**: Standard library module for generating application logs, crucial for monitoring connection pool activity and debugging errors.
*   **`psycopg2`**: The most popular PostgreSQL adapter for Python.
    *   **`psycopg2.pool`**: Specifically used for managing a pool of database connections, enhancing performance by reusing connections.
*   **`decouple`**: A library that helps to organize settings by separating parameters that vary by deployment environment (e.g., database credentials) from the codebase. It reads values from `.env` files or environment variables.
*   **`app.constants`**: An internal module (assumed to be `app/constants.py`) that likely defines an `Enum` or similar structure (`PostgreSQL`) for database-related configuration keys, making configuration more robust and type-safe.

### 3. Configuration Variables

The module uses several configuration parameters, some of which are hardcoded and others are loaded from environment variables via `decouple`.

*   **`MAX_RETRIES` (int)**:
    *   **Value**: `3`
    *   **Purpose**: Defines the maximum number of attempts the system will make to acquire a database connection from the pool before giving up and raising an error.
*   **`RETRY_DELAY` (int)**:
    *   **Value**: `1` (seconds)
    *   **Purpose**: The base delay in seconds between connection retry attempts. This value is multiplied by the retry count for exponential backoff.
*   **PostgreSQL Connection Parameters**:
    These parameters are loaded dynamically using `decouple.config()` and are essential for establishing the database connection. Their keys are sourced from `app.constants.PostgreSQL`.
    *   `database` (PostgreSQL.PG_DB): The name of the database to connect to.
    *   `user` (PostgreSQL.PG_USER): The username for database authentication.
    *   `password` (PostgreSQL.PG_PASSWORD): The password for database authentication.
    *   `host` (PostgreSQL.PG_HOST): The hostname or IP address of the PostgreSQL server.
    *   `port` (PostgreSQL.PG_PORT): The port number on which the PostgreSQL server is listening.

### 4. Logging Configuration

The module sets up basic logging using Python's standard `logging` library:

```python
logging.basicConfig(level=logging.INFO, format='%(asctime)s - %(levelname)s - %(message)s')
logger = logging.getLogger(__name__)
```
*   `logging.basicConfig(...)`: Configures the root logger.
    *   `level=logging.INFO`: Sets the minimum logging level to INFO. Messages with severity INFO, WARNING, ERROR, and CRITICAL will be processed.
    *   `format='%(asctime)s - %(levelname)s - %(message)s'`: Defines the format for log messages, including timestamp, log level, and the message content.
*   `logger = logging.getLogger(__name__)`: Creates a logger instance specific to this module. Using `__name__` ensures that log messages from this module are clearly identifiable.

### 5. `DatabasePool` Class

The `DatabasePool` class implements the Singleton design pattern to manage a single instance of a `psycopg2.pool.ThreadedConnectionPool`. This ensures that all parts of the application share the same connection pool, preventing resource duplication and managing connections efficiently.

#### Class Attributes:

*   **`_instance = None`**: A class-level variable used to store the single instance of the `DatabasePool` class, fundamental for the Singleton pattern.
*   **`_pool = None`**: A class-level variable that will hold the actual `psycopg2.pool.ThreadedConnectionPool` instance after initialization.

#### `__new__(cls)` Method

```python
def __new__(cls):
    if cls._instance is None:
        cls._instance = super(DatabasePool, cls).__new__(cls)
        cls._initialize_pool()
    return cls._instance
```
*   **Purpose**: This special method is invoked before `__init__` and controls instance creation. It enforces the Singleton pattern.
*   **Parameters**:
    *   `cls`: The class itself.
*   **Return**: The single instance of `DatabasePool`.
*   **Logic**:
    1.  It checks if `cls._instance` is `None`. If it is, this signifies that no instance of `DatabasePool` has been created yet.
    2.  If no instance exists, it calls `super(DatabasePool, cls).__new__(cls)` to create a new instance and assigns it to `cls._instance`.
    3.  Crucially, it then calls `cls._initialize_pool()` to set up the actual PostgreSQL connection pool *only once* when the first `DatabasePool` instance is created.
    4.  Finally, it returns the (newly created or existing) single instance.

#### `@classmethod` `_initialize_pool(cls)` Method

```python
@classmethod
def _initialize_pool(cls):
    """Initialize the PostgreSQL connection pool."""
    try:
        cls._pool = pool.ThreadedConnectionPool(
            minconn=5,
            maxconn=20,
            database=config(PostgreSQL.PG_DB.value),
            user=config(PostgreSQL.PG_USER.value),
            password=config(PostgreSQL.PG_PASSWORD.value),
            host=config(PostgreSQL.PG_HOST.value),
            port=config(PostgreSQL.PG_PORT.value),
        )
        logger.info("PostgreSQL Connection Pool initialized successfully")
    except Exception as e:
        logger.critical(f"Failed to initialize database connection pool: {e}")
        raise
```
*   **Purpose**: This private class method is responsible for instantiating and configuring the `psycopg2.pool.ThreadedConnectionPool`. It's called only once during the first `DatabasePool` instantiation.
*   **Parameters**:
    *   `cls`: The class itself.
*   **Return**: None.
*   **Logic**:
    1.  It attempts to create a `ThreadedConnectionPool` instance.
    2.  **`minconn=5`**: Specifies the minimum number of connections to maintain in the pool. These connections are opened eagerly upon pool initialization. Increased from a default of 1 for better initial availability.
    3.  **`maxconn=20`**: Specifies the maximum number of connections the pool can open. This prevents overwhelming the database server.
    4.  The remaining parameters (`database`, `user`, `password`, `host`, `port`) are retrieved from environment variables using `decouple.config()` with keys provided by `app.constants.PostgreSQL`.
    5.  If successful, an informational log message is recorded.
*   **Error Handling**:
    *   If any `Exception` occurs during pool initialization (e.g., incorrect credentials, unreachable database), a critical error message is logged, and the exception is re-raised. This ensures that the application fails fast if the essential database connection cannot be established.

#### `@classmethod` `get_pool(cls)` Method

```python
@classmethod
def get_pool(cls):
    """Get the connection pool instance."""
    if cls._pool is None:
        cls._initialize_pool()
    return cls._pool
```
*   **Purpose**: Provides a public interface to retrieve the initialized connection pool instance.
*   **Parameters**:
    *   `cls`: The class itself.
*   **Return**: The `psycopg2.pool.ThreadedConnectionPool` instance.
*   **Logic**:
    1.  It first checks if `cls._pool` is `None`. While `__new__` ensures initialization, this check acts as a safeguard in case `get_pool` is called before the Singleton fully initializes or if there's an unusual state.
    2.  If `_pool` is `None`, it calls `_initialize_pool()` to create it.
    3.  Finally, it returns the `_pool` instance.

#### `@classmethod` `get_connection(cls)` Method

```python
@classmethod
def get_connection(cls):
    """
    Get a connection from the pool with retry logic.

    Returns:
        A database connection from the pool

    Raises:
        Exception: If unable to get connection after retries
    """
    retry_count = 0
    last_error = None

    while retry_count < MAX_RETRIES:
        try:
            conn = cls._pool.getconn()
            # Test if connection is still valid
            with conn.cursor() as cursor:
                cursor.execute("SELECT 1")
            return conn
        except Exception as e:
            last_error = e
            logger.warning(f"Connection attempt {retry_count + 1} failed: {e}")
            retry_count += 1

            # If connection is bad, try to return it to the pool
            try:
                cls._pool.putconn(conn, close=True)
            except:
                pass  # Ignore errors when trying to return a potentially bad connection

            if retry_count < MAX_RETRIES:
                time.sleep(RETRY_DELAY * retry_count)  # Exponential backoff

    logger.error(f"Failed to get database connection after {MAX_RETRIES} attempts")
    raise last_error or Exception("Failed to get database connection")
```
*   **Purpose**: Acquires a database connection from the pool, incorporating robust retry logic and connection validation.
*   **Parameters**:
    *   `cls`: The class itself.
*   **Return**: A `psycopg2` database connection object.
*   **Raises**: An `Exception` if a connection cannot be obtained after `MAX_RETRIES` attempts.
*   **Logic**:
    1.  **Initialization**: `retry_count` is set to 0, and `last_error` is `None`.
    2.  **Retry Loop**: A `while` loop continues as long as `retry_count` is less than `MAX_RETRIES`.
    3.  **Connection Attempt**:
        *   `conn = cls._pool.getconn()`: Attempts to get a connection from the pool.
        *   **Connection Validation**: `with conn.cursor() as cursor: cursor.execute("SELECT 1")`: Executes a simple query to test if the obtained connection is still active and valid. If the query succeeds, the connection is good and returned.
    4.  **Error Handling & Retries**:
        *   If `getconn()` or the `SELECT 1` query fails, an `Exception` is caught.
        *   `last_error` stores the caught exception for later re-raising.
        *   A warning message is logged, indicating a failed attempt.
        *   `retry_count` is incremented.
        *   **Bad Connection Handling**: `cls._pool.putconn(conn, close=True)`: If a connection was obtained but failed validation (e.g., during `SELECT 1`), it's put back into the pool with `close=True`. This instructs `psycopg2` to physically close the connection, effectively removing a potentially bad connection from the pool. Errors during this step are silently ignored.
        *   **Exponential Backoff**: If more retries are remaining (`retry_count < MAX_RETRIES`), `time.sleep(RETRY_DELAY * retry_count)` introduces an increasing delay before the next attempt (e.g., 1s, then 2s, then 3s for `RETRY_DELAY=1`).
    5.  **Failure**: If the loop completes without successfully acquiring a connection, an error message is logged, and the `last_error` (or a generic `Exception`) is re-raised.

#### `@classmethod` `return_connection(cls, conn)` Method

```python
@classmethod
def return_connection(cls, conn):
    """
    Return a connection to the pool safely.

    Args:
        conn: The database connection to return
    """
    try:
        cls._pool.putconn(conn)
    except Exception as e:
        logger.warning(f"Error returning connection to pool: {e}")
        # If we can't return it normally, try to close it
        try:
            conn.close()
        except:
            pass  # Ignore errors on close
```
*   **Purpose**: Returns a database connection back to the pool, ensuring proper resource management. It includes safeguards for handling connections that cannot be returned normally.
*   **Parameters**:
    *   `cls`: The class itself.
    *   `conn`: The `psycopg2` database connection object to be returned.
*   **Return**: None.
*   **Logic**:
    1.  **Normal Return**: `cls._pool.putconn(conn)`: Attempts to return the connection to the pool. This is the standard way to release a connection for reuse.
    2.  **Error Handling**:
        *   If `putconn()` fails (e.g., the connection is already closed, or the pool is in an unexpected state), a warning message is logged.
        *   **Fallback Close**: As a last resort, `conn.close()` is attempted. This ensures that even if the connection cannot be re-pooled, its underlying resources are released, preventing resource leaks. Errors during the `conn.close()` operation are silently ignored, as the primary goal is to release the resource, and further error handling on a failing close is often not productive.

### 6. Global Instance and Function

#### `db_pool = DatabasePool()`

```python
db_pool = DatabasePool()
```
*   **Purpose**: This line creates the single global instance of the `DatabasePool` class. Due to the Singleton implementation in `DatabasePool.__new__`, this will:
    1.  Create the `DatabasePool` object.
    2.  Trigger the `_initialize_pool()` method to set up the `psycopg2.pool.ThreadedConnectionPool`.
*   **Execution Flow**: This code executes immediately when the `connection_manager.py` module is imported into another part of the application. This ensures that the database connection pool is initialized early in the application's lifecycle.

#### `get_db_pool()` Function

```python
def get_db_pool():
    """
    Get the database connection pool.

    Returns:
        The database connection pool singleton instance
    """
    return db_pool.get_pool()
```
*   **Purpose**: Provides a convenient, module-level function to access the globally available database connection pool instance. This abstracts away the `DatabasePool` class details for consumers of this module.
*   **Parameters**: None.
*   **Return**: The `psycopg2.pool.ThreadedConnectionPool` instance, obtained from the global `db_pool` object.
*   **Logic**: Simply calls `db_pool.get_pool()` to retrieve the underlying connection pool.

### 7. Main Execution Flow (`__main__` block)

While there isn't an explicit `if __name__ == "__main__":` block in this script, the "main execution flow" effectively happens upon module import.

When `connection_manager.py` is imported (e.g., `from app import connection_manager`), the following sequence of operations occurs:

1.  **Import statements** are processed.
2.  **Logging configuration** is set up.
3.  **Global variables** `MAX_RETRIES` and `RETRY_DELAY` are defined.
4.  The `DatabasePool` class is defined.
5.  **`db_pool = DatabasePool()`** is executed:
    *   The `DatabasePool.__new__` method is called.
    *   A single `DatabasePool` instance is created (if it doesn't already exist).
    *   The `DatabasePool._initialize_pool()` class method is invoked, which:
        *   Retrieves PostgreSQL connection parameters from environment variables via `decouple`.
        *   Instantiates the `psycopg2.pool.ThreadedConnectionPool` with `minconn=5` and `maxconn=20`.
        *   Logs an informational message or a critical error if initialization fails.
6.  The `get_db_pool()` function is defined.

This setup ensures that the connection pool is ready for use as soon as the module is imported, making it immediately available to other parts of the application.

### 8. Implementation Details and Design Patterns

*   **Singleton Pattern**: The `DatabasePool` class is implemented as a Singleton using the `__new__` method. This guarantees that only one instance of the connection pool exists throughout the application, centralizing connection management and preventing resource conflicts.
*   **Connection Pooling**: `psycopg2.pool.ThreadedConnectionPool` is used to manage a pool of database connections. This allows connections to be reused instead of being opened and closed for each request, significantly reducing overhead and improving application performance.
*   **Retry Logic with Exponential Backoff**: The `get_connection` method incorporates a robust retry mechanism. If an initial connection attempt fails, subsequent attempts are made with increasing delays (`RETRY_DELAY * retry_count`). This strategy makes the application more resilient to temporary database unavailability or network glitches.
*   **Defensive Programming**:
    *   **Connection Validation (`SELECT 1`)**: Before returning a connection from the pool, `get_connection` executes a `SELECT 1` query to verify that the connection is still active and functional. This prevents handing out "stale" or broken connections.
    *   **Safe Return of Connections**: `return_connection` includes a fallback mechanism to close a connection directly if it cannot be returned to the pool, preventing resource leaks.
    *   **Error Logging**: Extensive logging (INFO, WARNING, ERROR, CRITICAL) provides visibility into the connection management process, aiding in monitoring and debugging.
*   **Configuration Management**: The use of `decouple` for database credentials promotes best practices by separating sensitive configuration from the codebase, making the application easier to configure and deploy across different environments.
*   **Constants for Configuration Keys**: Leveraging `app.constants.PostgreSQL` provides type-safe and consistent access to configuration keys, reducing the chance of typos or hardcoding issues.
