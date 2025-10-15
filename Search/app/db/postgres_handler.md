# Documentation for `postgres_handler.py`

This document provides comprehensive technical documentation for the `postgres_handler.py` script, detailing its purpose, class structure, methods, and underlying implementation logic.

---

## `db/postgres_handler.py` Technical Documentation

### Table of Contents
1.  [Overview](#1-overview)
2.  [Dependencies](#2-dependencies)
3.  [Class: `PostgresHandler`](#3-class-postgreshandler)
    *   [Constructor: `__init__`](#31-constructor-__init__)
    *   [Private Method: `_get_connection`](#32-private-method-_get_connection)
    *   [Private Method: `_execute_with_retry`](#33-private-method-_execute_with_retry)
    *   [Public Method: `fetch_one`](#34-public-method-fetch_one)
    *   [Public Method: `fetch_all`](#35-public-method-fetch_all)
    *   [Public Method: `execute`](#36-public-method-execute)
    *   [Public Method: `close`](#37-public-method-close)
    *   [Business Logic Method: `get_team_by_code`](#38-business-logic-method-get_team_by_code)
    *   [Business Logic Method: `get_user_by_username`](#39-business-logic-method-get_user_by_username)
    *   [Business Logic Method: `update_terms_agreed`](#310-business-logic-method-update_terms_agreed)
    *   [Business Logic Method: `insert_user`](#311-business-logic-method-insert_user)
    *   [Business Logic Method: `get_all_teams`](#312-business-logic-method-get_all_teams)
    *   [Business Logic Method: `get_team_by_team_code`](#313-business-logic-method-get_team_by_team_code)
    *   [Business Logic Method: `get_team_by_name`](#314-business-logic-method-get_team_by_name)
    *   [Business Logic Method: `search_players_by_prefix`](#315-business-logic-method-search_players_by_prefix)
    *   [Business Logic Method: `search_teams_by_prefix`](#316-business-logic-method-search_teams_by_prefix)
    *   [Business Logic Method: `get_cached_aql`](#317-business-logic-method-get_cached_aql)
    *   [Business Logic Method: `cache_aql`](#318-business-logic-method-cache_aql)
4.  [Main Execution Flow](#4-main-execution-flow) (Not applicable for this module)
5.  [Implementation Details & Best Practices](#5-implementation-details--best-practices)

---

### 1. Overview

The `postgres_handler.py` script defines a `PostgresHandler` class, which serves as a robust and efficient interface for interacting with a PostgreSQL database. Its primary purpose is to abstract the complexities of database connection management (leveraging a connection pool), query execution, and error handling for various common database operations. The handler includes generic methods for fetching single rows, multiple rows, and executing DML statements, as well as specific methods tailored for user, team, player, and caching-related data operations. The class emphasizes connection pooling for resource efficiency and includes a retry mechanism for transient connection issues.

### 2. Dependencies

The script relies on the following Python modules and internal components:

*   **`json`**: Standard Python library for working with JSON data. Used for serializing Python dictionaries into JSON strings for storage in the database (`cache_aql`).
*   **`datetime`**: Standard Python library for working with dates and times. Used for timestamping cached entries (`cache_aql`).
*   **`app.db.connection.get_db_pool`**: An internal utility function expected to return a configured PostgreSQL connection pool object (e.g., from `psycopg2.pool` or similar library). This is crucial for managing database connections efficiently.

### 3. Class: `PostgresHandler`

This class provides a set of methods to perform various database operations against a PostgreSQL database, utilizing a connection pool for efficiency and resilience.

#### 3.1. Constructor: `__init__`

Initializes a new `PostgresHandler` instance, setting up the database connection pool.

```python
def __init__(self, pg_pool=None):
    self.pg_pool = pg_pool if pg_pool is not None else get_db_pool()
    self.conn = None
    self.cursor = None
```

*   **Parameters**:
    *   `pg_pool` (Optional[`psycopg2.pool.SimpleConnectionPool` or similar]): An existing PostgreSQL connection pool object. If provided, the handler will use this pool. If `None`, `get_db_pool()` is called to obtain a new connection pool.
*   **Implementation Logic**:
    *   `self.pg_pool`: Stores the connection pool instance. If an external pool is passed, it uses that; otherwise, it initializes a new one by calling `get_db_pool()`.
    *   `self.conn`: Initializes to `None`. This attribute will hold the active database connection obtained from the pool.
    *   `self.cursor`: Initializes to `None`. This attribute will hold the cursor object associated with `self.conn`, used for executing SQL queries.

#### 3.2. Private Method: `_get_connection`

This helper method retrieves a connection from the connection pool if one is not already active or if the existing connection is closed. It also logs connection pool statistics.

```python
def _get_connection(self):
    used_connections = len(self.pg_pool._used)
    free_connections = len(self.pg_pool._pool)
    total_connections = used_connections + free_connections

    print(f"Used connections: {used_connections}")
    print(f"Free connections: {free_connections}")
    print(f"Total connections in pool: {total_connections}")

    if self.conn is None or self.conn.closed:
        self.conn = self.pg_pool.getconn()
        self.conn.autocommit = True
        self.cursor = self.conn.cursor()
```

*   **Parameters**: None
*   **Returns**: None
*   **Implementation Logic**:
    *   **Connection Pool Statistics**: It first prints the current state of the connection pool by accessing the private attributes `_used` (connections checked out) and `_pool` (idle connections available) of the `pg_pool` object. This provides visibility into resource usage.
    *   **Connection Check**: It checks if `self.conn` is `None` or if the existing connection is `closed`.
    *   **Get Connection**: If no valid connection exists, it retrieves one from `self.pg_pool` using `getconn()`.
    *   **Autocommit**: Sets `self.conn.autocommit = True`. This means every SQL statement executed will be committed immediately without requiring an explicit `conn.commit()`. This is suitable for many web service scenarios but should be considered carefully for transactions involving multiple statements.
    *   **Create Cursor**: Obtains a new cursor object from the connection using `self.conn.cursor()`.

#### 3.3. Private Method: `_execute_with_retry`

This is the core method for executing SQL queries, incorporating connection management, a retry mechanism, and result fetching logic.

```python
def _execute_with_retry(self, query, params=None, fetch_method='fetchall'):
    max_retries = 2
    for attempt in range(max_retries + 1):
        try:
            # ... (connection pool statistics and _get_connection logic repeated) ...
            if self.conn is None or self.conn.closed:
                self.conn = self.pg_pool.getconn()
                self.conn.autocommit = True
                self.cursor = self.conn.cursor()

            self.cursor.execute(query, params or ())

            if fetch_method == 'fetchone':
                return self.cursor.fetchone()
            elif fetch_method == 'fetchall':
                return self.cursor.fetchall()
            elif fetch_method == 'execute':
                return None
                
        except Exception as e:
            if attempt < max_retries:
                self.close() # Close bad connection and retry
                continue
            else:
                raise e

        finally:
            self.pg_pool.putconn(self.conn)
            # ... (connection pool statistics repeated) ...
            self.conn = None
            self.cursor = None
```

*   **Parameters**:
    *   `query` (str): The SQL query string to be executed.
    *   `params` (Optional[tuple or list]): A tuple or list of parameters to substitute into the query. Defaults to `None`.
    *   `fetch_method` (str): Specifies how results should be fetched. Can be `'fetchone'`, `'fetchall'`, or `'execute'`. Defaults to `'fetchall'`.
*   **Returns**:
    *   A single row (tuple) if `fetch_method` is `'fetchone'`.
    *   A list of rows (list of tuples) if `fetch_method` is `'fetchall'`.
    *   `None` if `fetch_method` is `'execute'` (for DML operations).
*   **Implementation Logic**:
    *   **Retry Loop**: The method attempts to execute the query up to `max_retries + 1` times (3 attempts in this case).
    *   **Connection Management (inside `try` block)**:
        *   It prints connection pool statistics (similar to `_get_connection`).
        *   It explicitly checks `if self.conn is None or self.conn.closed` and retrieves a fresh connection from the pool if needed. This repetition of `_get_connection`'s logic ensures a valid connection is used for each attempt in the retry loop.
        *   `self.conn.autocommit = True` is set again for the newly acquired connection.
    *   **Query Execution**: `self.cursor.execute(query, params or ())` executes the SQL query. `params or ()` handles cases where `params` is `None`, providing an empty tuple to `execute`.
    *   **Result Fetching**: Based on `fetch_method`:
        *   `fetchone()`: Retrieves the next row of a query result set or `None` if no more rows are available.
        *   `fetchall()`: Retrieves all (remaining) rows of a query result set as a list of tuples.
        *   `execute()`: Returns `None` as DML operations (INSERT, UPDATE, DELETE) typically don't return results.
    *   **Error Handling (`except` block)**:
        *   If an `Exception` occurs and `attempt` is less than `max_retries`, the `close()` method is called to return the potentially bad connection to the pool (or close it directly if it's unusable), and the loop `continue`s for another retry.
        *   If `max_retries` are exhausted, the exception is re-raised.
    *   **Cleanup (`finally` block)**:
        *   `self.pg_pool.putconn(self.conn)`: *Crucially*, the connection `self.conn` is always returned to the pool, ensuring resources are freed.
        *   Connection Pool Statistics: Prints pool statistics after releasing the connection.
        *   `self.conn = None` and `self.cursor = None`: Resets the instance's connection and cursor references to `None` to prevent accidental reuse of a connection that has been returned to the pool and is no longer under the handler's direct control. This also forces `_get_connection` to fetch a new connection for subsequent operations.

#### 3.4. Public Method: `fetch_one`

A convenience method to execute a `SELECT` query and return a single row.

```python
def fetch_one(self, query, params=None):
    """Execute SELECT query and return single result"""
    return self._execute_with_retry(query, params, 'fetchone')
```

*   **Parameters**:
    *   `query` (str): The SQL `SELECT` query.
    *   `params` (Optional[tuple or list]): Parameters for the query.
*   **Returns**: A single row as a tuple, or `None` if no results are found.
*   **Implementation Logic**: Delegates directly to `_execute_with_retry` with `fetch_method` set to `'fetchone'`.

#### 3.5. Public Method: `fetch_all`

A convenience method to execute a `SELECT` query and return all matching rows.

```python
def fetch_all(self, query, params=None):
    """Execute SELECT query and return all rows"""
    return self._execute_with_retry(query, params, 'fetchall')
```

*   **Parameters**:
    *   `query` (str): The SQL `SELECT` query.
    *   `params` (Optional[tuple or list]): Parameters for the query.
*   **Returns**: A list of rows, where each row is a tuple. Returns an empty list if no results.
*   **Implementation Logic**: Delegates directly to `_execute_with_retry` with `fetch_method` set to `'fetchall'`.

#### 3.6. Public Method: `execute`

A convenience method to execute DML (Data Manipulation Language) queries like `INSERT`, `UPDATE`, or `DELETE`.

```python
def execute(self, query, params=None):
    """Run INSERT/UPDATE/DELETE"""
    return self._execute_with_retry(query, params, 'execute')
```

*   **Parameters**:
    *   `query` (str): The SQL DML query.
    *   `params` (Optional[tuple or list]): Parameters for the query.
*   **Returns**: `None`.
*   **Implementation Logic**: Delegates directly to `_execute_with_retry` with `fetch_method` set to `'execute'`.

#### 3.7. Public Method: `close`

Returns the current active database connection to the pool.

```python
def close(self):
    """Return connection to pool"""
    try:
        if self.conn and self.pg_pool:
            self.pg_pool.putconn(self.conn)
        self.conn = None
        self.cursor = None
        print(f"Closed Adapter: PGHandler")

    except Exception as e:
        print(f"PG Handler Exception occurred {e}")
        if self.conn:
            self.conn.close() # If can't return to pool, close directly
        self.conn = None
        self.cursor = None
```

*   **Parameters**: None
*   **Returns**: None
*   **Implementation Logic**:
    *   **Graceful Return to Pool**: Attempts to return `self.conn` to `self.pg_pool` using `putconn()`. This is done only if `self.conn` and `self.pg_pool` are not `None`.
    *   **Cleanup**: Regardless of success, `self.conn` and `self.cursor` are set to `None` to ensure the handler does not hold onto a released connection.
    *   **Error Handling**: If an exception occurs during `putconn()` (e.g., the connection is truly bad or the pool is misconfigured), it prints an error message. As a fallback, if `self.conn` still exists, it attempts to close the connection directly using `self.conn.close()`, then performs the standard cleanup of `self.conn` and `self.cursor`.

#### 3.8. Business Logic Method: `get_team_by_code`

Fetches a team's name based on its team code.

```python
def get_team_by_code(self, team_code):
    """Fetch team by team_code from PostgreSQL"""
    query = "SELECT team_name FROM teams WHERE team_code = %s"
    result = self.fetch_one(query, (str(team_code),))
    if result:
        return {"team_name": result[0]}
    return None
```

*   **Parameters**:
    *   `team_code` (str): The unique code identifying the team.
*   **Returns**:
    *   A dictionary `{"team_name": "Team Name"}` if a team is found.
    *   `None` if no team matches the code.
*   **Implementation Logic**:
    *   Executes a `SELECT` query on the `teams` table to retrieve `team_name` where `team_code` matches the input.
    *   Uses `fetch_one` as only a single result is expected.
    *   Converts the `team_code` to a string using `str()` before passing it as a parameter, ensuring type compatibility with the `%s` placeholder.
    *   If a result is found, it's extracted from the tuple (`result[0]`) and returned as a dictionary.

#### 3.9. Business Logic Method: `get_user_by_username`

Fetches all columns for a user based on their username.

```python
def get_user_by_username(self, username):
    """Fetch user by username from PostgreSQL"""
    query = "SELECT * FROM users WHERE username = %s"
    return self.fetch_one(query, (username,))
```

*   **Parameters**:
    *   `username` (str): The username of the user to fetch.
*   **Returns**:
    *   A tuple containing all user data if found.
    *   `None` if no user matches the username.
*   **Implementation Logic**:
    *   Executes a `SELECT *` query on the `users` table to retrieve all columns for a user where `username` matches the input.
    *   Uses `fetch_one` as only a single user is expected.

#### 3.10. Business Logic Method: `update_terms_agreed`

Updates the `terms_agreed` status for a specific user to `TRUE`.

```python
def update_terms_agreed(self, username):
    query = "UPDATE users SET terms_agreed = TRUE WHERE username = %s"
    return self.execute(query, (username,))
```

*   **Parameters**:
    *   `username` (str): The username of the user whose `terms_agreed` status is to be updated.
*   **Returns**: `None` (as per the `execute` method's return).
*   **Implementation Logic**:
    *   Executes an `UPDATE` query on the `users` table, setting `terms_agreed` to `TRUE` for the row where `username` matches.
    *   Uses `execute` for this DML operation.

#### 3.11. Business Logic Method: `insert_user`

Inserts a new user record into the `users` table and returns the generated `user_id`.

```python
def insert_user(self, user_data):
    query = """
        INSERT INTO users (email, team, sports, username, is_active, hashed_pwd, terms_agreed)
        VALUES (%s, %s, %s, %s, %s, %s, %s)
        RETURNING user_id
    """
    params = (
        user_data["email"],
        user_data["team"],
        user_data["sports"],
        user_data["username"],
        user_data["is_active"],
        user_data["hashed_pwd"],
        user_data["terms_agreed"]
    )
    return self.fetch_one(query, params)
```

*   **Parameters**:
    *   `user_data` (dict): A dictionary containing user details, expected to have keys like `"email"`, `"team"`, `"sports"`, `"username"`, `"is_active"`, `"hashed_pwd"`, and `"terms_agreed"`.
*   **Returns**:
    *   A tuple `(user_id,)` containing the ID of the newly inserted user.
    *   `None` if the insertion failed or did not return an ID.
*   **Implementation Logic**:
    *   Constructs an `INSERT` query for the `users` table.
    *   The `RETURNING user_id` clause is used to get the generated `user_id` back from the database immediately after insertion.
    *   Prepares a tuple `params` from the `user_data` dictionary, ensuring the order matches the columns in the `INSERT` statement.
    *   Uses `fetch_one` because `RETURNING user_id` will yield a single row with one column.

#### 3.12. Business Logic Method: `get_all_teams`

Retrieves all team names from the `tmn_temp` table.

```python
def get_all_teams(self):
    query = "SELECT team_name FROM tmn_temp"
    print("Finding all team from PostgresSQL)")
    return [row[0] for row in self.fetch_all(query)]
```

*   **Parameters**: None
*   **Returns**: A list of strings, where each string is a team name.
*   **Implementation Logic**:
    *   Executes a `SELECT` query to get `team_name` from the `tmn_temp` table.
    *   Prints a log message indicating the operation.
    *   Uses `fetch_all` to get all results.
    *   A list comprehension `[row[0] for row in ...]` extracts just the team name (first element of each tuple) into a flat list.

#### 3.13. Business Logic Method: `get_team_by_team_code`

Retrieves a team name based on its team code. This method is functionally similar to `get_team_by_code` but returns only the name string directly.

```python
def get_team_by_team_code(self, team_code):
    query = "SELECT team_name FROM teams WHERE team_code = %s"
    print(f"Finding tean name from postgresSQL : {query}")
    result = self.fetch_one(query,(team_code,))
    if result:
        return result[0]
    return None
```

*   **Parameters**:
    *   `team_code` (str): The unique code identifying the team.
*   **Returns**:
    *   A string representing the `team_name` if a team is found.
    *   `None` if no team matches the code.
*   **Implementation Logic**:
    *   Executes a `SELECT` query on the `teams` table to retrieve `team_name` where `team_code` matches the input.
    *   Prints a log message with the query.
    *   Uses `fetch_one`.
    *   If a result is found, `result[0]` directly returns the team name string.

#### 3.14. Business Logic Method: `get_team_by_name`

Retrieves a team code based on its team name.

```python
def get_team_by_name(self, team_name):
    query = "SELECT team_code FROM teams WHERE team_name = %s"
    print(f"Finding team code from postgresSQL : {query}")
    result = self.fetch_one(query, (team_name,))
    if result:
        return result[0]
    return None
```

*   **Parameters**:
    *   `team_name` (str): The name of the team to search for.
*   **Returns**:
    *   A string representing the `team_code` if a team is found.
    *   `None` if no team matches the name.
*   **Implementation Logic**:
    *   Executes a `SELECT` query on the `teams` table to retrieve `team_code` where `team_name` matches the input.
    *   Prints a log message with the query.
    *   Uses `fetch_one`.
    *   If a result is found, `result[0]` directly returns the team code string.

#### 3.15. Business Logic Method: `search_players_by_prefix`

Searches for player names where either `playerName` or `firstName` starts with a given prefix, case-insensitively.

```python
def search_players_by_prefix(self,prefix):
    query="""
    SELECT "playerName" FROM active_roster
    WHERE "playerName" ILIKE %s OR "firstName" ILIKE %s
    LIMIT 15
    """
    like_pattern = f"{prefix}%"
    return [row[0] for row in self.fetch_all(query, (like_pattern, like_pattern))]
```

*   **Parameters**:
    *   `prefix` (str): The prefix to match against player names.
*   **Returns**: A list of strings, where each string is a player name.
*   **Implementation Logic**:
    *   Constructs a `SELECT` query for `playerName` from the `active_roster` table.
    *   Uses the `ILIKE` operator for case-insensitive pattern matching. It checks both `"playerName"` and `"firstName"` fields.
    *   The `LIMIT 15` clause restricts the number of results to the top 15.
    *   Creates a `like_pattern` by appending `"%"` to the `prefix` for `LIKE` clause matching (e.g., "jo" becomes "jo%").
    *   Passes `like_pattern` twice to the query parameters for both `ILIKE` conditions.
    *   Uses `fetch_all` and extracts the player names into a flat list.

#### 3.16. Business Logic Method: `search_teams_by_prefix`

Searches for team names that start with a given prefix, case-insensitively.

```python
def search_teams_by_prefix(self,prefix):
    query = """
    SELECT team_name FROM team_master_new
    WHERE team_name ILIKE %s
    LIMIT 10
    """
    like_pattern = f"{prefix}%"
    return [row[0] for row in self.fetch_all(query, (like_pattern,))]
```

*   **Parameters**:
    *   `prefix` (str): The prefix to match against team names.
*   **Returns**: A list of strings, where each string is a team name.
*   **Implementation Logic**:
    *   Constructs a `SELECT` query for `team_name` from the `team_master_new` table.
    *   Uses the `ILIKE` operator for case-insensitive pattern matching on the `team_name` field.
    *   The `LIMIT 10` clause restricts the number of results to the top 10.
    *   Creates a `like_pattern` by appending `"%"` to the `prefix`.
    *   Uses `fetch_all` and extracts the team names into a flat list.

#### 3.17. Business Logic Method: `get_cached_aql`

Checks if a specific AQL (Analytics Query Language) query result is already cached for a given user.

```python
def get_cached_aql(self, userid: str, query: str):
    """Check if a query is already cached for the user"""
    sql = """
        SELECT aql
        FROM cache_queries
        WHERE user_id = %s AND query = %s
        ORDER BY createdon DESC
        LIMIT 1
    """
    result = self.fetch_one(sql, (userid, query))
    return result[0] if result else None
```

*   **Parameters**:
    *   `userid` (str): The ID of the user.
    *   `query` (str): The AQL query string to check for.
*   **Returns**:
    *   A string (JSON representation of the AQL output) if a matching cached entry is found.
    *   `None` if no cache entry is found.
*   **Implementation Logic**:
    *   Executes a `SELECT` query on the `cache_queries` table to retrieve the `aql` column.
    *   Filters by `user_id` and `query`.
    *   `ORDER BY createdon DESC LIMIT 1` ensures that if multiple entries exist for the same user and query, the most recent one is retrieved.
    *   Uses `fetch_one`.
    *   If a result is found, `result[0]` (the AQL output string) is returned; otherwise, `None`.

#### 3.18. Business Logic Method: `cache_aql`

Inserts a new AQL query and its output into the cache.

```python
def cache_aql(self, userid: str, query: str, aql_output: dict, sport_code: str, entity: str):
    """Insert a new AQL record into cache"""
    sql = """
        INSERT INTO cache_queries (user_id, query, aql, createdon, sport_code, entity)
        VALUES (%s, %s, %s, %s, %s , %s)
    """
    self.execute(sql, (userid, query, json.dumps(aql_output), datetime.now(), sport_code, entity))
```

*   **Parameters**:
    *   `userid` (str): The ID of the user.
    *   `query` (str): The AQL query string.
    *   `aql_output` (dict): The dictionary containing the result of the AQL query.
    *   `sport_code` (str): The sports code associated with the query.
    *   `entity` (str): The entity related to the query (e.g., 'player', 'team').
*   **Returns**: `None` (as per the `execute` method's return).
*   **Implementation Logic**:
    *   Constructs an `INSERT` query for the `cache_queries` table, inserting the `user_id`, `query`, `aql` (output), `createdon`, `sport_code`, and `entity`.
    *   `json.dumps(aql_output)`: The `aql_output` dictionary is serialized into a JSON string before being stored in the database, assuming the `aql` column is of type `JSONB` or `TEXT`.
    *   `datetime.now()`: The current timestamp is used for the `createdon` column.
    *   Uses `execute` for this DML operation.

### 4. Main Execution Flow

The provided code defines a class `PostgresHandler` and its methods. It does not contain a `if __name__ == "__main__":` block, meaning it is intended to be imported as a module and its class instantiated and used by other parts of an application rather than being run directly as a standalone script.

### 5. Implementation Details & Best Practices

*   **Connection Pooling**: The core strength of `PostgresHandler` is its reliance on a connection pool (`self.pg_pool`). This significantly improves performance and resource utilization by reusing established database connections instead of opening and closing them for every operation.
*   **`autocommit = True`**: While convenient for simple, independent queries, using `autocommit=True` means each `execute` call (including `fetch_one` and `fetch_all` which use `execute` internally) implicitly commits its transaction immediately. For multi-statement transactions that require atomicity (all or nothing), explicit transaction management (e.g., `conn.begin()`, `conn.commit()`, `conn.rollback()`) would be necessary, and `autocommit` should be set to `False`. The current implementation is suitable for many API/microservice use cases where individual database calls are largely independent.
*   **Retry Mechanism**: `_execute_with_retry` provides resilience against transient database connection issues or temporary network glitches. It attempts to re-establish a connection and retry the query a few times before failing.
*   **Connection Cleanup**: The `finally` block in `_execute_with_retry` and the `close()` method are critical for ensuring that connections are *always* returned to the pool (or closed if unusable), preventing connection leaks and resource exhaustion.
*   **Placeholder Usage (`%s`)**: The queries correctly use `%s` placeholders for parameters, which is the standard and safest way to pass data to SQL queries, preventing SQL injection vulnerabilities.
*   **Logging/Debugging**: The `print` statements provide basic visibility into connection pool usage and query execution, which can be helpful for debugging but might be replaced with a more robust logging framework (e.g., Python's `logging` module) in a production environment.
*   **Type Hinting**: The `get_cached_aql` and `cache_aql` methods demonstrate the use of type hints (e.g., `userid: str`), improving code readability and enabling static analysis tools.
*   **Case-Insensitive Search (`ILIKE`)**: Methods like `search_players_by_prefix` and `search_teams_by_prefix` leverage PostgreSQL's `ILIKE` operator for case-insensitive pattern matching, which is often a user-friendly requirement for search functionalities.
*   **JSON Storage**: `cache_aql` demonstrates how to store Python dictionaries in a PostgreSQL database by converting them to JSON strings using `json.dumps()`. This typically works well with `JSONB` column types in PostgreSQL for efficient querying and storage of semi-structured data.
