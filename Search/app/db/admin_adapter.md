# Documentation for `admin_adapter.py`

This document provides comprehensive technical documentation for the `AdminAdapter` Python script, detailing its purpose, structure, dependencies, and the functionality of each method.

---

## `AdminAdapter` Technical Documentation

### 1. High-Level Overview

The `AdminAdapter` class serves as a specialized data access layer (DAL) for administrative operations within an application, specifically interacting with a PostgreSQL database. It extends `PGSQLAdapter`, inheriting its core database connection and cursor management capabilities, and provides a set of methods to manage users and teams. This includes approving/rejecting users, listing various categories of users and teams, onboarding new teams, managing password reset tokens, and performing CRUD operations (Create, Read, Update, Search) on user records. The adapter primarily relies on pre-defined SQL templates to execute database queries, ensuring separation of concerns and maintainability.

### 2. Dependencies and Imports

The script begins by importing necessary modules and classes:

*   `from collections import defaultdict`: While imported, `defaultdict` is *not actually used* in the provided script. It might be a leftover from previous development or intended for future use.
*   `import logging`: Essential for logging informational messages, warnings, and errors throughout the application.
*   `from app.data_config.mappings.mappings_handler import MappingsHandler`: Similar to `defaultdict`, `MappingsHandler` is imported but *not used* in this specific script.
*   `from app.constants import PostgreSQL`: Imports a constant, likely related to PostgreSQL database configuration. *Not directly used in `AdminAdapter`'s methods, but implicitly by `PGSQLAdapter`.*
*   `from app.sql_templates.sql_template_loader import SQLTemplateLoader`: A critical dependency responsible for loading SQL query templates from specified file paths. This promotes clean code by externalizing SQL queries.
*   `import uuid`: Used for generating universally unique identifiers (UUIDs), specifically for creating password reset tokens.
*   `from app.db.pgsql_adapter import PGSQLAdapter`: The base class from which `AdminAdapter` inherits. It provides the fundamental database connection and cursor management functionalities.

### 3. Logging Configuration

```python
logging.basicConfig(level=logging.INFO, format='%(asctime)s - %(levelname)s - %(message)s')
logger = logging.getLogger()
```

*   `logging.basicConfig(...)`: Configures the root logger.
    *   `level=logging.INFO`: Sets the minimum logging level to `INFO`. This means messages with `INFO`, `WARNING`, `ERROR`, and `CRITICAL` levels will be processed, while `DEBUG` messages will be ignored.
    *   `format='%(asctime)s - %(levelname)s - %(message)s'`: Defines the format for log messages. It includes the timestamp (`asctime`), log level (`levelname`), and the actual log message (`message`).
*   `logger = logging.getLogger()`: Retrieves the root logger instance. This `logger` object is then used throughout the `AdminAdapter` class to record events and errors.

### 4. `AdminAdapter` Class Definition

```python
class AdminAdapter(PGSQLAdapter):
    def __init__(self, pg_pool=None):
        super().__init__(pg_pool)
        self.logger = logger
```

*   **`class AdminAdapter(PGSQLAdapter):`**: Defines `AdminAdapter` as a class that inherits from `PGSQLAdapter`. This inheritance implies that `AdminAdapter` will have access to all public and protected methods and attributes of `PGSQLAdapter`, including database connection pooling and cursor management.
*   **`def __init__(self, pg_pool=None):`**: The constructor for the `AdminAdapter` class.
    *   **`pg_pool=None`**: An optional parameter, likely a connection pool object (e.g., `psycopg2.pool.SimpleConnectionPool` or a similar custom pool). If provided, the base `PGSQLAdapter` will use this pool for database connections.
    *   **`super().__init__(pg_pool)`**: Calls the constructor of the parent class, `PGSQLAdapter`, passing the `pg_pool` argument. This initializes the inherited database connection and related resources.
    *   **`self.logger = logger`**: Assigns the globally configured `logger` instance to an instance variable `self.logger`. This allows each instance of `AdminAdapter` to use the same logger for consistency.

### 5. Methods of `AdminAdapter`

Each method follows a common pattern: it acquires a database cursor using `self.get_cursor()`, loads a specific SQL template, executes the query (often with parameters), processes the results, and handles exceptions by logging errors and re-raising them. The `self.get_cursor()` method, inherited from `PGSQLAdapter`, likely manages transactions (e.g., committing on success, rolling back on failure, or closing the cursor) via a context manager, which explains why explicit `self.conn.commit()` and `self.conn.rollback()` calls are commented out in various methods.

#### 5.1 `approve_user(self, user_id: int)`

*   **Purpose**: Approves a user by updating their status in the database.
*   **Parameters**:
    *   `user_id` (int): The unique identifier of the user to be approved.
*   **Returns**:
    *   `False` if `user_id` is invalid.
    *   `None` if no row is found after execution (meaning no user with that `user_id` was approved or existed).
    *   A `dict` representing the approved user's details if the operation is successful and a row is returned.
*   **Implementation Logic**:
    1.  `with self.get_cursor() as cursor:`: Obtains a database cursor using a context manager.
    2.  `if not isinstance(user_id, int) or user_id <= 0:`: Performs input validation. If `user_id` is not a positive integer, an error is logged, and `False` is returned.
    3.  `sql = SQLTemplateLoader('app/sql_templates/admin').load_template("approve_user")`: Loads the SQL template named "approve_user" from the `app/sql_templates/admin` directory.
    4.  `cursor.execute(sql.format(user_id=user_id))`: Executes the SQL query. The `user_id` is directly formatted into the SQL string. **Note**: This direct formatting can be vulnerable to SQL injection if `user_id` originates from untrusted input. It's generally safer to pass parameters as a tuple to `execute()` to allow the database driver to handle escaping.
    5.  `row = cursor.fetchone()`: Fetches a single row of results.
    6.  `if not row: return None`: If no row is returned, it means no user was found or updated, so `None` is returned.
    7.  `column_names = [desc[0].lower() for desc in cursor.description]`: Retrieves column names from the cursor's description and converts them to lowercase.
    8.  `return dict(zip(column_names, row))`: Combines the lowercase column names with the fetched row data into a dictionary and returns it.
    9.  **Error Handling**: A `try-except Exception as e` block catches any database query errors, logs them, and re-raises the exception.

#### 5.2 `reject_user(self, user_id: int)`

*   **Purpose**: Rejects a user by updating their status in the database.
*   **Parameters**:
    *   `user_id` (int): The unique identifier of the user to be rejected.
*   **Returns**:
    *   `False` if `user_id` is invalid.
    *   `True` if the user rejection operation is performed (regardless of whether a user was actually updated, assuming the SQL execution itself succeeded).
*   **Implementation Logic**:
    1.  Acquires a cursor.
    2.  Validates `user_id` similarly to `approve_user`.
    3.  Loads the "reject_user" SQL template.
    4.  Executes the SQL query, again using string formatting for `user_id`.
    5.  `return True`: Returns `True` indicating the operation was attempted. This method doesn't fetch rows, so it assumes success if no exceptions occur.
    6.  **Error Handling**: Logs and re-raises any database query errors.

#### 5.3 `list_pending_users(self)`

*   **Purpose**: Retrieves a list of users who are currently in a "pending" status.
*   **Parameters**: None.
*   **Returns**:
    *   An empty list (`[]`) if no pending users are found.
    *   A `list` of `dict`s, where each dictionary represents a pending user's details.
*   **Implementation Logic**:
    1.  Acquires a cursor.
    2.  Loads the "pending_users" SQL template.
    3.  `cursor.execute(sql)`: Executes the SQL query without parameters.
    4.  `rows = cursor.fetchall()`: Fetches all rows returned by the query.
    5.  `if not rows: return []`: If no rows are found, returns an empty list.
    6.  Converts fetched rows into a list of dictionaries with lowercase column names, similar to `approve_user`.
    7.  **Error Handling**: Logs and re-raises any database query errors.

#### 5.4 `list_recent_users(self, since_date, limit=10, offset=0)`

*   **Purpose**: Retrieves a paginated list of users added or updated since a specific date.
*   **Parameters**:
    *   `since_date`: The starting date (type not strictly enforced but likely a string or datetime object compatible with the SQL query).
    *   `limit` (int): The maximum number of user records to return (defaults to 10).
    *   `offset` (int): The number of user records to skip from the beginning (defaults to 0).
*   **Returns**:
    *   An empty list (`[]`) if `since_date` is `None` or no recent users are found.
    *   A `list` of `dict`s, where each dictionary represents a recent user's details.
*   **Implementation Logic**:
    1.  Acquires a cursor.
    2.  `if since_date is None: ...`: Validates that `since_date` is provided. If `None`, an error is logged, and an empty list is returned.
    3.  Loads the "recent_users" SQL template.
    4.  `cursor.execute(sql, (since_date, limit, offset))`: Executes the SQL query, passing `since_date`, `limit`, and `offset` as parameters. This is the **correct and safe way** to pass parameters to `execute()`, preventing SQL injection.
    5.  `rows = cursor.fetchall()`: Fetches all rows.
    6.  If no rows, returns an empty list.
    7.  Converts fetched rows into a list of dictionaries with lowercase column names.
    8.  **Error Handling**: Logs and re-raises any database query errors.

#### 5.5 `list_all_users(self, limit=10, offset=0)`

*   **Purpose**: Retrieves a paginated list of all users in the system.
*   **Parameters**:
    *   `limit` (int): The maximum number of user records to return (defaults to 10).
    *   `offset` (int): The number of user records to skip from the beginning (defaults to 0).
*   **Returns**:
    *   An empty list (`[]`) if no users are found.
    *   A `list` of `dict`s, where each dictionary represents a user's details.
*   **Implementation Logic**:
    1.  Acquires a cursor.
    2.  Loads the "list_users" SQL template.
    3.  `cursor.execute(f"{sql} LIMIT %s OFFSET %s", (limit, offset))`: Executes the SQL query. It appends `LIMIT %s OFFSET %s` to the loaded SQL template and passes `limit` and `offset` as parameters. This dynamic addition of `LIMIT/OFFSET` suggests the base "list_users" template might not include pagination clauses.
    4.  Fetches all rows.
    5.  If no rows, returns an empty list.
    6.  Converts fetched rows into a list of dictionaries with lowercase column names.
    7.  **Error Handling**: Logs and re-raises any database query errors.

#### 5.6 `list_team_requests(self)`

*   **Purpose**: Retrieves a list of pending team requests.
*   **Parameters**: None.
*   **Returns**:
    *   An empty list (`[]`) if no team requests are found.
    *   A `list` of `dict`s, where each dictionary represents a team request's details.
*   **Implementation Logic**:
    1.  Acquires a cursor.
    2.  Loads the "team_requests" SQL template.
    3.  Executes the SQL query without parameters.
    4.  Fetches all rows.
    5.  If no rows, returns an empty list.
    6.  Converts fetched rows into a list of dictionaries with lowercase column names.
    7.  **Error Handling**: Logs and re-raises any database query errors.

#### 5.7 `listAllTeams(self)`

*   **Purpose**: Retrieves a list of all teams in the system.
*   **Parameters**: None.
*   **Returns**:
    *   `None` if no teams are found. (Note: This differs from `list_pending_users` which returns `[]`).
    *   A `list` of `dict`s, where each dictionary represents a team's details.
*   **Implementation Logic**:
    1.  Acquires a cursor.
    2.  Loads the "allteams" SQL template.
    3.  Executes the SQL query.
    4.  Fetches all rows.
    5.  `if not rows: return None`: If no rows are found, returns `None`.
    6.  Converts fetched rows into a list of dictionaries with lowercase column names.
    7.  **Error Handling**: Logs and re-raises any database query errors.

#### 5.8 `getTeamDetails(self, team_code)`

*   **Purpose**: Retrieves detailed information for a specific team identified by its team code.
*   **Parameters**:
    *   `team_code`: The unique code of the team to retrieve details for. (Type not specified, but likely int or string).
*   **Returns**:
    *   `None` if no team is found with the given `team_code`.
    *   A `dict` representing the team's details.
*   **Implementation Logic**:
    1.  Acquires a cursor.
    2.  Loads the "getTeamDetails" SQL template.
    3.  `cursor.execute(query,(team_code,))`: Executes the SQL query, safely passing `team_code` as a parameter.
    4.  `row = cursor.fetchone()`: Fetches a single row.
    5.  `if not row: return None`: If no row is found, returns `None`.
    6.  Converts the fetched row into a dictionary with lowercase column names.
    7.  **Error Handling**: Logs and re-raises any database query errors.

#### 5.9 `onboard_team(self, team_name: str, team_code: int, tidy_name: str, img_url: str, primary_color_code: str, secondary_color_code: str, sports: list[int])`

*   **Purpose**: Inserts a new team into the database with all its associated details.
*   **Parameters**:
    *   `team_name` (str): The official name of the team.
    *   `team_code` (int): A unique numerical code for the team.
    *   `tidy_name` (str): A 'tidied' or formatted name for the team.
    *   `img_url` (str): URL for the team's image/logo.
    *   `primary_color_code` (str): Hex code or name for the team's primary color.
    *   `secondary_color_code` (str): Hex code or name for the team's secondary color.
    *   `sports` (list[int]): A list of integer IDs representing the sports the team participates in.
*   **Returns**:
    *   `True` if the team onboarding operation is performed.
*   **Implementation Logic**:
    1.  Acquires a cursor.
    2.  Loads the "onboard_team" SQL template.
    3.  `cursor.execute(sql.format(...))`: Executes the SQL query, using string formatting to insert all team details. **Note**: This method uses string formatting, which is less secure than parameterized queries, especially given the number and types of parameters. It's recommended to switch to `cursor.execute(sql, (param1, param2, ...))` for all parameters.
    4.  `return True`: Returns `True` indicating the operation was attempted.
    5.  **Error Handling**: Logs and re-raises any database query errors.

#### 5.10 `getSports(self) -> list`

*   **Purpose**: Retrieves a list of all available sports from the database.
*   **Parameters**: None.
*   **Returns**:
    *   An empty list (`[]`) if no sports are found.
    *   A `list` of `dict`s, where each dictionary represents a sport's details.
*   **Implementation Logic**:
    1.  Acquires a cursor.
    2.  Loads the "getSports" SQL template.
    3.  Executes the SQL query.
    4.  Fetches all rows.
    5.  If no rows, returns an empty list.
    6.  Converts fetched rows into a list of dictionaries with lowercase column names.
    7.  **Error Handling**: Logs and re-raises any database query errors.

#### 5.11 `generatePasswordResetToken(self, usernameorEmail: str) -> str`

*   **Purpose**: Generates a unique password reset token, stores it in the database for a given user (identified by username or email), and returns the token.
*   **Parameters**:
    *   `usernameorEmail` (str): The username or email of the user requesting a password reset.
*   **Returns**:
    *   `str`: The newly generated UUID token.
*   **Implementation Logic**:
    1.  Acquires a cursor.
    2.  `token = str(uuid.uuid4())`: Generates a new version 4 UUID (random) and converts it to a string.
    3.  Loads the "generateAdminPasswordResetToken" SQL template.
    4.  `cursor.execute(query.format(usernameorEmail=usernameorEmail, token=token))`: Executes the SQL query, using string formatting for `usernameorEmail` and `token`. **Note**: Similar to `approve_user` and `onboard_team`, using string formatting for `usernameorEmail` is a potential SQL injection vulnerability. Parameterized queries are preferred.
    5.  `return token`: Returns the generated token.
    6.  **Error Handling**: Logs and re-raises any database query errors.

#### 5.12 `createUser(self, user_data)`

*   **Purpose**: Creates a new user record in the system with the provided `user_data`, including hashing the password.
*   **Parameters**:
    *   `user_data`: An object (likely a custom data transfer object or a simple `dict` with dot access) containing user registration details such as `UserName`, `Email`, `fullName`, `password`, `designation`, and `TeamId`.
*   **Returns**:
    *   `True` if the user creation operation is performed.
*   **Implementation Logic**:
    1.  Acquires a cursor.
    2.  `import bcrypt`: Imports the `bcrypt` library (typically imported at the top of the file, but here it's lazy-loaded within the method).
    3.  `password_hash = bcrypt.hashpw(user_data.password.encode('utf-8'), bcrypt.gensalt()).decode('utf-8')`: Hashes the user's password using `bcrypt`.
        *   `user_data.password.encode('utf-8')`: Encodes the password string into bytes, as `bcrypt` works with bytes.
        *   `bcrypt.gensalt()`: Generates a random salt, which is crucial for security as it prevents rainbow table attacks and ensures identical passwords have different hashes.
        *   `bcrypt.hashpw(...)`: Performs the hashing.
        *   `.decode('utf-8')`: Decodes the resulting hash bytes back into a UTF-8 string for storage.
    4.  Loads the "create_user" SQL template.
    5.  `cursor.execute(sql, (...))`: Executes the SQL query, passing all user data fields and the generated `password_hash` as a tuple of parameters. This is a **secure parameterized query**.
    6.  `True # is_approved - Admin created users are approved by default`: A comment indicating that newly created users via this admin function are automatically approved.
    7.  `return True`: Returns `True` indicating the operation was attempted.
    8.  **Error Handling**: Logs any database query errors and re-raises the exception. The commented-out `self.conn.rollback()` suggests manual transaction management might have been considered or is handled by `PGSQLAdapter`'s context manager.

#### 5.13 `updateUser(self, user_id, user_data)`

*   **Purpose**: Updates an existing user's details in the system. It handles password updates conditionally.
*   **Parameters**:
    *   `user_id`: The ID of the user to update.
    *   `user_data`: An object (similar to `createUser`) containing the updated user details. It may optionally include a new `password`.
*   **Returns**:
    *   `True` if the user update operation is performed.
*   **Implementation Logic**:
    1.  Acquires a cursor.
    2.  `if hasattr(user_data, 'password') and user_data.password:`: Checks if the `user_data` object has a `password` attribute and if it's not empty. This determines if the password needs to be updated.
    3.  **Password Update Path**:
        *   `import bcrypt`: Imports `bcrypt` (lazy-loaded).
        *   `password_hash = bcrypt.hashpw(...)`: Hashes the new password using `bcrypt` (same logic as `createUser`).
        *   Loads the "update_user_with_password" SQL template.
        *   Executes the SQL query, passing all updated user data including the `password_hash` and `user_id` as parameters. This is a **secure parameterized query**.
    4.  **No Password Update Path**:
        *   Loads the "update_user" SQL template.
        *   Executes the SQL query, passing updated user data (excluding password) and `user_id` as parameters. This is also a **secure parameterized query**.
    5.  `return True`: Returns `True` after successful execution of either update path.
    6.  **Error Handling**: Logs any database query errors and re-raises the exception. The commented-out `self.conn.rollback()` suggests similar transaction handling considerations as `createUser`.

#### 5.14 `search_users(self, query, limit=10, offset=0)`

*   **Purpose**: Searches for users based on a query string across multiple fields (username, email, full name) with pagination.
*   **Parameters**:
    *   `query` (str): The search string.
    *   `limit` (int): Maximum number of results (defaults to 10).
    *   `offset` (int): Number of results to skip (defaults to 0).
*   **Returns**:
    *   `list`: A list of dictionaries, where each dictionary represents a matching user record. Returns an empty list (`[]`) if no matches or on error.
*   **Implementation Logic**:
    1.  Acquires a cursor.
    2.  `search_pattern = f"%{query}%"`: Creates a wildcard search pattern for SQL `ILIKE` (case-insensitive LIKE).
    3.  Loads the "search_users" SQL template.
    4.  `cursor.execute(sql, (search_pattern, search_pattern, search_pattern, limit, offset))`: Executes the SQL query, passing the `search_pattern` three times (for different columns), `limit`, and `offset` as parameters. This is a **secure parameterized query**.
    5.  Fetches all rows.
    6.  If no rows, returns an empty list.
    7.  Converts fetched rows into a list of dictionaries with lowercase column names.
    8.  **Error Handling**: Logs any database query errors. Instead of re-raising the exception, it **returns an empty list (`[]`)** to prevent breaking the frontend, making it more resilient to database issues during search.

---

### Important Implementation Details and Considerations

*   **SQL Template Loader (`SQLTemplateLoader`)**: This is a key design choice. It centralizes SQL queries, making them easier to manage, review, and modify without touching the Python code logic. The path `app/sql_templates/admin` indicates where these templates are stored.
*   **`PGSQLAdapter` Inheritance**: `AdminAdapter` relies heavily on `PGSQLAdapter` for core database interactions. This implies that `PGSQLAdapter` handles connection pooling, error handling, and potentially transaction management through its `get_cursor()` context manager.
*   **Transaction Management (Commented `commit`/`rollback`)**: The commented-out `self.conn.commit()` and `self.conn.rollback()` calls suggest that transaction control is either implicitly handled by the `PGSQLAdapter`'s `get_cursor()` context manager (e.g., `psycopg2` context managers often commit on exit if no exception occurred, and rollback otherwise), or it is expected to be managed at a higher application layer. If `PGSQLAdapter` does not implicitly commit, then operations that modify data (like `approve_user`, `reject_user`, `onboard_team`, `generatePasswordResetToken`, `createUser`, `updateUser`) would not persist changes unless `self.conn.commit()` is explicitly called or wrapped in a transaction manager.
*   **SQL Injection Vulnerabilities**:
    *   Methods like `approve_user`, `reject_user`, `onboard_team`, and `generatePasswordResetToken` use `sql.format(...)` to embed parameters directly into the SQL string. This is a significant security risk if the input parameters (e.g., `user_id`, `team_name`, `usernameorEmail`) originate from untrusted sources (e.g., user input in a web application).
    *   **Recommendation**: All methods should use parameterized queries, passing parameters as a tuple to `cursor.execute()`, for example: `cursor.execute(sql, (param1, param2, ...))`. This allows the database driver to properly escape special characters, mitigating SQL injection risks. The `list_recent_users`, `list_all_users`, `getTeamDetails`, `createUser`, `updateUser`, and `search_users` methods already follow this best practice.
*   **Password Hashing**: The `createUser` and `updateUser` methods correctly use `bcrypt` for password hashing with a generated salt, which is a robust security practice. The lazy import of `bcrypt` could be moved to the top of the file for better clarity and to ensure the dependency is visible early.
*   **Result Formatting**: The consistent pattern of converting `cursor.description` to lowercase column names and then zipping them with `fetchone()` or `fetchall()` results into a list of dictionaries (`[dict(zip(column_names, row)) for row in rows]`) provides a clean and consistent data structure for the application consuming these methods.
*   **Error Handling**: All database interaction methods include `try-except` blocks to catch `Exception`. Errors are logged with contextual information (`Database Query Error in [method_name]`), which is good for debugging and monitoring. Most methods re-raise the exception, pushing error handling to the calling layer, except for `search_users` which returns an empty list, providing a more graceful failure for user-facing features.
*   **Consistency in Empty Returns**: There's a slight inconsistency in return values for "not found" scenarios. Some methods return `[]` (e.g., `list_pending_users`), while others return `None` (e.g., `approve_user`, `listAllTeams`). For list-type operations, returning an empty list is generally preferred as it simplifies iteration in the calling code.
*   **Unused Imports**: `defaultdict` and `MappingsHandler` are imported but not used, which indicates potential dead code or future plans. They should be removed if not needed to keep the module clean.
