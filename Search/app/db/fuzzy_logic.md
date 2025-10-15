# Documentation for `fuzzy_logic.py`

This document provides comprehensive technical documentation for the given Python script, detailing its purpose, structure, functions, and key implementation aspects.

---

## 1. Overview

This Python script defines a `FuzzyMatch` class designed to perform fuzzy matching operations against a PostgreSQL database for various sports entities like teams, conferences, and players. It leverages SQL templates to construct dynamic queries based on `sport_code` and the entity name. The script uses `psycopg2` for database interaction, managed through a connection pool, and `decouple` for secure configuration loading from environment variables or `.env` files. It also incorporates structured logging to track execution flow and performance.

The primary goal of this module is to provide a standardized way to find canonical names for sports entities when given potentially ambiguous or partially matching input names, using database-driven fuzzy logic.

## 2. Dependencies and Imports

The script relies on several external libraries and internal modules:

*   **`from decouple import config`**:
    *   **Purpose**: Used to manage configuration variables, allowing them to be loaded from environment variables (e.g., `os.environ`) or `.env` files. This promotes Twelve-Factor App principles by keeping configuration separate from code.
    *   **Usage**: The `config()` function is called with a key (string) to retrieve its corresponding value.
*   **`import logging`**:
    *   **Purpose**: Python's standard library for flexible event logging. It provides a robust framework for emitting log messages from the application.
    *   **Usage**: Used to set up a logger, define its level, and format, and then record informational messages, warnings, errors, etc.
*   **`from app.constants import PostgreSQL`**:
    *   **Purpose**: Imports an enumeration or a class containing constant values related to PostgreSQL database configuration. These constants likely define the keys used with `decouple.config` (e.g., `PG_DB`, `PG_USER`).
    *   **Usage**: Provides a structured way to reference configuration keys, improving readability and reducing magic strings.
*   **`from app.sql_templates.sql_template_loader import SQLTemplateLoader`**:
    *   **Purpose**: Imports a custom class responsible for loading SQL queries from predefined template files. This allows for cleaner separation of SQL logic from Python code and promotes reusability.
    *   **Usage**: An instance is created with a base path, and then its `load_template` method is used to retrieve specific SQL query strings.
*   **`import time`**:
    *   **Purpose**: Python's standard library for time-related functions.
    *   **Usage**: Used here to measure the execution duration of the fuzzy matching operations by recording start and end timestamps.
*   **`from app.db.pgsql_adapter import PGSQLAdapter`**:
    *   **Purpose**: Imports a custom database adapter class, likely providing a wrapper around `psycopg2` to simplify common database operations (e.g., managing connections, cursors, executing queries).
    *   **Usage**: The `FuzzyMatch` class inherits from `PGSQLAdapter`, suggesting it relies on its parent for fundamental database interaction capabilities.
*   **`from psycopg2 import pool`**:
    *   **Purpose**: Imports the `pool` module from `psycopg2`, the most popular PostgreSQL adapter for Python. This module provides classes for managing a pool of database connections.
    *   **Usage**: Specifically, `pool.SimpleConnectionPool` is used to create and manage a fixed-size pool of database connections, enhancing performance by reusing connections instead of establishing new ones for each operation.

## 3. Global Configuration and Initialization

This section covers the setup performed at the module level when the script is imported or run.

### 3.1. PostgreSQL Connection Pool (`pg_pool`)

```python
pg_pool = pool.SimpleConnectionPool(
    minconn=1,
    maxconn=10,
    database=config(PostgreSQL.PG_DB.value),
    user=config(PostgreSQL.PG_USER.value),
    password=config(PostgreSQL.PG_PASSWORD.value),
    host=config(PostgreSQL.PG_HOST.value),
    port=config(PostgreSQL.PG_PORT.value),
)
```

*   **`pg_pool = pool.SimpleConnectionPool(...)`**: Initializes a simple connection pool for PostgreSQL. This pool will manage a specified number of database connections, making them available for reuse.
    *   **`minconn=1`**: Specifies the minimum number of connections the pool will maintain at all times. If connections drop below this, the pool will attempt to create new ones.
    *   **`maxconn=10`**: Specifies the maximum number of connections the pool can hold. If all connections are in use, new requests will wait until a connection becomes available.
    *   **`database=config(PostgreSQL.PG_DB.value)`**: Retrieves the PostgreSQL database name from configuration (e.g., `.env` or environment variables) using the `PostgreSQL.PG_DB` constant as the key.
    *   **`user=config(PostgreSQL.PG_USER.value)`**: Retrieves the PostgreSQL username from configuration.
    *   **`password=config(PostgreSQL.PG_PASSWORD.value)`**: Retrieves the PostgreSQL password from configuration.
    *   **`host=config(PostgreSQL.PG_HOST.value)`**: Retrieves the PostgreSQL host address from configuration.
    *   **`port=config(PostgreSQL.PG_PORT.value)`**: Retrieves the PostgreSQL port number from configuration.

### 3.2. Logging Configuration

```python
logging.basicConfig(level=logging.INFO, format='%(asctime)s - %(levelname)s - %(message)s')
logger = logging.getLogger()
```

*   **`logging.basicConfig(...)`**: Configures the root logger for the application.
    *   **`level=logging.INFO`**: Sets the minimum logging level to `INFO`. Messages with a severity of `INFO`, `WARNING`, `ERROR`, or `CRITICAL` will be processed, while `DEBUG` messages will be ignored.
    *   **`format='%(asctime)s - %(levelname)s - %(message)s'`**: Defines the format for log messages.
        *   `%(asctime)s`: The time when the LogRecord was created.
        *   `%(levelname)s`: The textual logging level for the message (e.g., `INFO`, `WARNING`).
        *   `%(message)s`: The logged message itself.
*   **`logger = logging.getLogger()`**: Retrieves the root logger instance after it has been configured. This `logger` object will be used throughout the script to emit log messages.

## 4. Class: `FuzzyMatch`

This class encapsulates the logic for performing various fuzzy matching operations against a PostgreSQL database.

```python
class FuzzyMatch(PGSQLAdapter):
    # ... methods ...
```

### 4.1. Inheritance

*   The `FuzzyMatch` class inherits from `PGSQLAdapter`. This implies that `FuzzyMatch` instances will automatically gain access to database connection and cursor management functionalities provided by `PGSQLAdapter`. Typically, `PGSQLAdapter` would handle obtaining a connection from `pg_pool`, creating a cursor, executing queries, and potentially committing/rolling back transactions, and releasing the connection back to the pool.

### 4.2. `__init__` Method

```python
    def __init__(self):
        super().__init__()
```

*   **Purpose**: The constructor for the `FuzzyMatch` class.
*   **`super().__init__()`**: Calls the constructor of the parent class, `PGSQLAdapter`. This is crucial because the `PGSQLAdapter`'s `__init__` method is expected to initialize the database connection and cursor (e.g., `self.cursor`, `self.conn`) that the `FuzzyMatch` methods will rely on.

### 4.3. `fuzzy_match_team` Method

```python
    def fuzzy_match_team(self, sport_code, team_name):
        time_start = time.time()
        logger.info(f"Executing fuzzy match for team name: {team_name}")
        query = SQLTemplateLoader(f'app/sql_templates/fuzzyMatch/{sport_code}').load_template(
                "teamNames")
        query = query.format(team_name=f"'{team_name}'")
        logger.info(f"Executing SQL Query:\n{query}")

        self.cursor.execute(query)
        rows = self.cursor.fetchone()
        logger.info(f"Fuzzy match results: {rows}")
        time_end = time.time()

        logger.info(f"Time taken for fuzzy match: {time_end - time_start} seconds")
        if not rows:
            return team_name

        return rows[0]
```

*   **Purpose**: Performs a fuzzy match operation to find a canonical team name based on a given input `team_name` and `sport_code`.
*   **Parameters**:
    *   `self`: The instance of the `FuzzyMatch` class.
    *   `sport_code` (str): A string representing the sport (e.g., "NBA", "NFL"). This is used to locate the correct SQL template directory.
    *   `team_name` (str): The input team name to be fuzzy-matched.
*   **Return Value**:
    *   If a match is found in the database, it returns the canonical team name (the first element of the fetched row, `rows[0]`).
    *   If no match is found, it returns the original `team_name` input.
*   **Implementation Logic**:
    1.  **`time_start = time.time()`**: Records the starting timestamp to measure execution time.
    2.  **`logger.info(...)`**: Logs an informational message indicating the start of the fuzzy match process for the given `team_name`.
    3.  **`query = SQLTemplateLoader(f'app/sql_templates/fuzzyMatch/{sport_code}').load_template("teamNames")`**:
        *   Creates an instance of `SQLTemplateLoader`, pointing it to a directory specific to the `sport_code` (e.g., `app/sql_templates/fuzzyMatch/NBA`).
        *   Calls `load_template("teamNames")` to load the SQL query template named "teamNames" from that directory. This template is expected to contain placeholders for the `team_name`.
    4.  **`query = query.format(team_name=f"'{team_name}'")`**:
        *   Formats the loaded SQL query string by replacing the `team_name` placeholder.
        *   **CRITICAL NOTE**: The input `team_name` is directly embedded into the SQL query string using an f-string and enclosed in single quotes (`f"'{team_name}'"`). **This approach is vulnerable to SQL injection attacks.** A malicious `team_name` could alter the query's intent (e.g., `' OR '1'='1`). It is highly recommended to use parameterized queries (`self.cursor.execute(query, (team_name,))`) instead, where the database driver handles proper escaping.
    5.  **`logger.info(...)`**: Logs the fully constructed SQL query for debugging purposes.
    6.  **`self.cursor.execute(query)`**: Executes the formatted SQL query using the cursor provided by the `PGSQLAdapter` parent class.
    7.  **`rows = self.cursor.fetchone()`**: Fetches the first (or only) row returned by the query. Fuzzy match queries are typically designed to return at most one best match.
    8.  **`logger.info(...)`**: Logs the results obtained from the database.
    9.  **`time_end = time.time()`**: Records the ending timestamp.
    10. **`logger.info(...)`**: Logs the total time taken for the fuzzy match operation.
    11. **`if not rows:`**: Checks if any row was returned (i.e., if a match was found).
        *   If `rows` is `None` or empty, it means no match was found, so the original `team_name` is returned.
    12. **`return rows[0]`**: If a match was found, it returns the first element of the fetched row, which is assumed to be the canonical team name.

### 4.4. `fuzzy_match_conference` Method

```python
    def fuzzy_match_conference(self, sport_code, conference_name):
        time_start = time.time()
        logger.info(f"Executing fuzzy match for conference name: {conference_name}")
        query = SQLTemplateLoader(f'app/sql_templates/fuzzyMatch/{sport_code}').load_template(
                "conferenceNames")
        query = query.format(conference_name=f"'{conference_name}'")
        logger.info(f"Executing SQL Query:\n{query}")

        self.cursor.execute(query)
        rows = self.cursor.fetchone()
        logger.info(f"Fuzzy match results: {rows}")
        time_end = time.time()

        logger.info(f"Time taken for fuzzy match: {time_end - time_start} seconds")
        if not rows:
            return conference_name

        return rows[0]
```

*   **Purpose**: Performs a fuzzy match operation to find a canonical conference name.
*   **Parameters**:
    *   `self`: The instance of the `FuzzyMatch` class.
    *   `sport_code` (str): The sport code.
    *   `conference_name` (str): The input conference name to be fuzzy-matched.
*   **Return Value**:
    *   If a match is found, returns the canonical conference name (`rows[0]`).
    *   If no match is found, returns the original `conference_name` input.
*   **Implementation Logic**: The logic is almost identical to `fuzzy_match_team`, with the following differences:
    *   It uses the `conference_name` parameter.
    *   It loads the SQL template named `"conferenceNames"`.
    *   It formats the query with the `conference_name` placeholder.
    *   **SQL Injection Vulnerability**: This method also directly embeds `conference_name` into the SQL query, posing the same SQL injection risk as `fuzzy_match_team`.

### 4.5. `fuzzy_match_player` Method

```python
    def fuzzy_match_player(self, sport_code, player_name):
        time_start = time.time()
        logger.info(f"Executing fuzzy match for player name: {player_name}")
        query = SQLTemplateLoader(f'app/sql_templates/fuzzyMatch/{sport_code}').load_template(
                "playerNames")
        query = query.format(player_name=f"'{player_name}'")
        logger.info(f"Executing SQL Query:\n{query}")

        self.cursor.execute(query)
        rows = self.cursor.fetchone()
        logger.info(f"Fuzzy match results: {rows}")
        time_end = time.time()

        logger.info(f"Time taken for fuzzy match: {time_end - time_start} seconds")
        if not rows:
            return player_name


        return rows[0].replace("'", "''")
```

*   **Purpose**: Performs a fuzzy match operation to find a canonical player name.
*   **Parameters**:
    *   `self`: The instance of the `FuzzyMatch` class.
    *   `sport_code` (str): The sport code.
    *   `player_name` (str): The input player name to be fuzzy-matched.
*   **Return Value**:
    *   If a match is found, returns the canonical player name (`rows[0]`), with any single quotes within the name escaped by doubling them (`''`).
    *   If no match is found, returns the original `player_name` input.
*   **Implementation Logic**: The logic is largely similar to the other fuzzy match methods, with key differences:
    *   It uses the `player_name` parameter.
    *   It loads the SQL template named `"playerNames"`.
    *   It formats the query with the `player_name` placeholder.
    *   **SQL Injection Vulnerability**: This method also directly embeds `player_name` into the SQL query, posing the same SQL injection risk.
    *   **Special Character Handling**: **`return rows[0].replace("'", "''")`**: If a player name is found, it performs a string replacement. Any single quote (`'`) found in the returned player name (`rows[0]`) is replaced with two single quotes (`''`). This is a common technique to escape single quotes when constructing SQL strings *manually*. This suggests that the returned player name might be re-inserted into another SQL query later, and this escaping is done preemptively to prevent syntax errors in that subsequent query. This practice further emphasizes the manual SQL string construction prevalent in the current design.

## 5. Usage / Main Execution Flow (Implicit)

This script does not contain an `if __name__ == "__main__":` block, meaning it is not designed to be run directly as a standalone application. Instead, it is intended to be imported as a module into other parts of a larger application.

A typical usage pattern would look like this:

```python
# In another file, e.g., main_app.py

from app.fuzzy_match import FuzzyMatch
from app.db.pgsql_adapter import pg_pool # Assuming pg_pool is also globally accessible or passed

def main():
    # It's crucial that PGSQLAdapter (and thus FuzzyMatch) properly
    # acquires and releases connections from the pg_pool.
    # A context manager pattern is highly recommended for PGSQLAdapter.
    # For now, assuming PGSQLAdapter's __init__ and cleanup handle it.
    
    matcher = FuzzyMatch()
    
    try:
        nba_team = "LAL"
        canonical_nba_team = matcher.fuzzy_match_team("NBA", nba_team)
        print(f"Fuzzy match for '{nba_team}' (NBA): {canonical_nba_team}")

        nfl_player = "Tom Brady"
        canonical_nfl_player = matcher.fuzzy_match_player("NFL", nfl_player)
        print(f"Fuzzy match for '{nfl_player}' (NFL): {canonical_nfl_player}")
        
        college_conf = "SEC conf"
        canonical_college_conf = matcher.fuzzy_match_conference("NCAAFB", college_conf)
        print(f"Fuzzy match for '{college_conf}' (NCAAFB): {canonical_college_conf}")

    finally:
        # Assuming PGSQLAdapter has a close method or is used in a context manager
        # If PGSQLAdapter doesn't explicitly close, connections are returned to pool on object destruction
        # or at specific points in its lifecycle.
        # It's important to ensure connections are properly returned.
        # For the global pg_pool itself, it might need to be closed when the application shuts down.
        # E.g., pg_pool.closeall()
        pass 

if __name__ == "__main__":
    main()
```

## 6. Key Implementation Details and Considerations

### 6.1. Environment Variables with `decouple`

The use of `decouple.config` is a good practice for externalizing configuration. It ensures that sensitive information (like database credentials) is not hardcoded in the source code and can be easily changed across different environments (development, staging, production) without modifying the code. It checks environment variables first, then falls back to a `.env` file in the project root.

### 6.2. SQL Template Loading (`SQLTemplateLoader`)

Separating SQL queries into templates (e.g., in `app/sql_templates/fuzzyMatch/{sport_code}/teamNames.sql`) promotes:
*   **Modularity**: SQL logic is distinct from Python code.
*   **Maintainability**: SQL queries can be updated without touching Python code.
*   **Readability**: Complex SQL queries are easier to read and manage in dedicated files.
*   **Reusability**: Templates can be shared or adapted for similar operations.

### 6.3. Database Adapter (`PGSQLAdapter`)

Inheriting from `PGSQLAdapter` centralizes database interaction logic. This adapter is expected to abstract away the details of `psycopg2`, providing a consistent interface for managing connections, cursors, and executing queries. A well-designed adapter would also handle connection acquisition from the pool and its release back to the pool, potentially using context managers (`with` statements) for resource safety.

### 6.4. Connection Pooling (`psycopg2.pool.SimpleConnectionPool`)

Using `SimpleConnectionPool` is a performance optimization. Instead of opening a new database connection for every operation, connections are reused from the pool. This reduces the overhead associated with establishing new connections (handshakes, authentication) and makes database access faster and more efficient, especially in high-traffic applications.

### 6.5. Logging Practices

The script uses Python's `logging` module effectively to:
*   Track the start and end of fuzzy match operations.
*   Log the constructed SQL queries, which is invaluable for debugging and understanding what SQL is actually being executed.
*   Record execution times, providing insights into performance.
*   Capture fuzzy match results.
The `INFO` level is appropriate for general operational messages.

### 6.6. **CRITICAL SECURITY CONCERN: SQL Injection Vulnerability**

The current implementation of dynamically constructing SQL queries:
```python
query = query.format(team_name=f"'{team_name}'")
```
is highly vulnerable to **SQL Injection attacks**.
By directly embedding user-supplied input (`team_name`, `conference_name`, `player_name`) into the SQL string without proper escaping, a malicious user could craft an input that alters the query's logic, accesses unauthorized data, or even performs destructive operations on the database.

**Example Vulnerability:**
If `team_name` is provided as `' OR 1=1; --`, the query could become:
`SELECT * FROM teams WHERE name = '' OR 1=1; --'`
This would effectively bypass any name matching and return all team names, or worse, execute subsequent malicious commands.

**Recommendation**:
Always use parameterized queries provided by the `psycopg2` driver. The driver will handle the proper escaping of special characters.
Corrected approach:
```python
# In SQL template: SELECT id, name FROM teams WHERE name = %s;
# In Python:
query = SQLTemplateLoader(f'app/sql_templates/fuzzyMatch/{sport_code}').load_template("teamNames")
# ...
self.cursor.execute(query, (team_name,)) # Pass parameters as a tuple
rows = self.cursor.fetchone()
```
The same applies to `fuzzy_match_conference` and `fuzzy_match_player`.

### 6.7. Special Character Handling in `fuzzy_match_player`

The line `return rows[0].replace("'", "''")` in `fuzzy_match_player` attempts to escape single quotes by doubling them. While this is a common manual escaping technique for SQL strings, it primarily serves to make the *result* suitable for direct embedding into *another* SQL query. This is a symptom of the lack of proper parameterized queries throughout the system. If parameterized queries were used consistently, this manual escaping would be unnecessary and prone to errors (e.g., what if the database canonical name itself contains `''`?).
