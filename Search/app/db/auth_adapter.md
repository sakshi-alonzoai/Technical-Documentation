# Documentation for `auth_adapter.py`

This document provides comprehensive technical documentation for the provided Python script, `AuthAdapter.py`. It details the script's purpose, its components, functions, and important implementation considerations.

---

## Technical Documentation: `AuthAdapter.py`

### 1. High-Level Overview

The `AuthAdapter.py` script defines the `AuthAdapter` class, which extends `PGSQLAdapter` to provide a set of methods for interacting with a PostgreSQL database specifically for user authentication and authorization functionalities. This adapter handles operations such as user registration, login (including fetching user details and password verification), team listing, password reset token generation, password updates, and tracking user agreement to terms. It leverages SQL templates for database queries, `bcrypt` for secure password hashing, and integrates with a database connection pool managed by the base `PGSQLAdapter`.

### 2. Dependencies and Imports

The script relies on several external modules and internal application components:

*   **`collections.defaultdict`**: (Imported but not used in the provided code snippet). A dictionary subclass that calls a factory function to supply missing values.
*   **`logging`**: Used for setting up and managing application-wide logging. Errors and informational messages are logged to assist in debugging and monitoring.
*   **`uuid`**: Used for generating universally unique identifiers (UUIDs), specifically for creating secure password reset tokens.
*   **`app.data_config.mappings.mappings_handler.MappingsHandler`**: (Imported but not used in the provided code snippet). Likely an internal component for handling data mappings, though not directly utilized in `AuthAdapter`.
*   **`app.constants.PostgreSQL`**: (Imported but not used in the provided code snippet). A module likely containing constants related to PostgreSQL, such as database names or connection parameters.
*   **`app.sql_templates.sql_template_loader.SQLTemplateLoader`**: An internal utility class responsible for loading SQL queries from predefined template files. This promotes separation of SQL logic from application code.
*   **`app.db.pgsql_adapter.PGSQLAdapter`**: The base class from which `AuthAdapter` inherits. It is expected to provide fundamental database interaction capabilities, such as obtaining database cursors and managing connection pools.
*   **`app.db.connection.get_db_pool`**: (Imported but not directly used in the current implementation, its functionality is likely encapsulated within `PGSQLAdapter`). A function presumably used to retrieve a PostgreSQL connection pool.
*   **`fastapi.HTTPException`**: Used to raise HTTP-specific exceptions, typically within the context of a FastAPI web application, indicating authentication or authorization failures.
*   **`bcrypt`**: A library for hashing passwords securely. It generates strong, salted hashes and provides a method for verifying a plain-text password against a hash.

### 3. Logging Configuration

```python
logging.basicConfig(level=logging.INFO, format='%(asctime)s - %(levelname)s - %(message)s')
logger = logging.getLogger()
```

*   **`logging.basicConfig(...)`**: Configures the root logger.
    *   `level=logging.INFO`: Sets the minimum logging level to `INFO`. Messages with levels `INFO`, `WARNING`, `ERROR`, and `CRITICAL` will be processed.
    *   `format='%(asctime)s - %(levelname)s - %(message)s'`: Defines the format for log messages, including timestamp, log level, and the message itself.
*   **`logger = logging.getLogger()`**: Retrieves the root logger instance, which will be used throughout the `AuthAdapter` class for logging messages.

### 4. Class: `AuthAdapter`

The `AuthAdapter` class is designed to encapsulate all database operations related to user authentication and authorization. It inherits from `PGSQLAdapter`, suggesting it relies on its parent for robust database connection and cursor management.

```python
class AuthAdapter(PGSQLAdapter):
    # ... methods ...
```

#### 4.1. `__init__` Method

```python
    def __init__(self):
        super().__init__()
```

*   **Purpose**: The constructor for the `AuthAdapter` class.
*   **Logic**: It simply calls the constructor of its parent class, `PGSQLAdapter`, using `super().__init__()`. This ensures that any initialization logic or setup required by the base database adapter (e.g., connection pool setup, configuration loading) is executed.

#### 4.2. `listTeams` Method

```python
    def listTeams(self):
        with self.get_cursor() as cursor:
            try:
                query = SQLTemplateLoader(f'app/sql_templates/auth').load_template("teams")
                cursor.execute(query)
                rows = cursor.fetchall()
                if not rows:
                    return []
                column_names = [desc[0].lower() for desc in cursor.description]
                result = [dict(zip(column_names, row)) for row in rows]
                return result
            except Exception as e:
                logger.error(f"Database Query Error in listTeams: {e}")
                raise
```

*   **Purpose**: Retrieves a list of all teams from the database.
*   **Parameters**: None.
*   **Return Value**:
    *   A list of dictionaries, where each dictionary represents a team with column names as keys and row values as values.
    *   An empty list `[]` if no teams are found.
*   **Logic**:
    1.  **`with self.get_cursor() as cursor:`**: Obtains a database cursor using a context manager provided by the base `PGSQLAdapter` class. This ensures proper resource management (e.g., cursor closing, connection return to pool).
    2.  **`query = SQLTemplateLoader(f'app/sql_templates/auth').load_template("teams")`**: Loads the SQL query named "teams" from the `app/sql_templates/auth` directory.
    3.  **`cursor.execute(query)`**: Executes the loaded SQL query.
    4.  **`rows = cursor.fetchall()`**: Fetches all rows returned by the query.
    5.  **`if not rows: return []`**: If no rows are returned, an empty list is immediately returned.
    6.  **`column_names = [desc[0].lower() for desc in cursor.description]`**: Extracts column names from the cursor's description and converts them to lowercase. This prepares for mapping row data to dictionary keys.
    7.  **`result = [dict(zip(column_names, row)) for row in rows]`**: Converts the fetched rows (which are typically tuples) into a list of dictionaries, using the `column_names` as keys.
    8.  **`return result`**: Returns the list of team dictionaries.
    9.  **`except Exception as e:`**: Catches any database-related exceptions, logs the error, and re-raises it to propagate the issue upstream.

#### 4.3. `registerUser` Method

```python
    def registerUser(self, username: str, email: str, full_name: str, password_hash: str, user_role: str, team_id: int):
        with self.get_cursor() as cursor:
            try:
                query = SQLTemplateLoader(f'app/sql_templates/auth').load_template("register")
                # self.pg_pool = get_db_pool() # Commented-out connection management
                # ...
                cursor.execute(query, (username, email, full_name, password_hash, user_role, team_id))
                # self.conn.commit() # Commented-out explicit commit (implies autocommit or base class handles it)
            except Exception as e:
                logger.error(f"Database Query Error in registerUser: {e}")
                raise
        # finally: # Commented-out connection release
        #     self.pg_pool.putconn(self.conn)
        #     self.conn = None
        #     self.cursor = None
```

*   **Purpose**: Registers a new user in the database.
*   **Parameters**:
    *   `username` (`str`): The user's chosen username.
    *   `email` (`str`): The user's email address.
    *   `full_name` (`str`): The user's full name.
    *   `password_hash` (`str`): The bcrypt-hashed password of the user.
    *   `user_role` (`str`): The role assigned to the user (e.g., 'admin', 'user').
    *   `team_id` (`int`): The ID of the team the user belongs to.
*   **Return Value**: None.
*   **Logic**:
    1.  **`with self.get_cursor() as cursor:`**: Obtains a database cursor.
    2.  **`query = SQLTemplateLoader(f'app/sql_templates/auth').load_template("register")`**: Loads the "register" SQL insert query.
    3.  **`cursor.execute(query, (username, email, full_name, password_hash, user_role, team_id))`**: Executes the query with the provided user details as a parameterized query, preventing SQL injection.
    4.  **Commented-out code**: The commented-out lines indicate a previous or alternative manual connection management approach that has likely been refactored into the `PGSQLAdapter` base class's `get_cursor()` method, which now handles connection pooling and transaction management (e.g., autocommit or explicit commit in the context manager exit).
    5.  **`except Exception as e:`**: Logs and re-raises any database errors during registration.

#### 4.4. `get_user_by_username_or_email` Method

```python
    def get_user_by_username_or_email(self, username_or_email: str):
        """Get user by username or email without password verification"""
        with self.get_cursor() as cursor:
            try:
                query = SQLTemplateLoader(f'app/sql_templates/auth').load_template("login")
                # ... commented-out connection management ...
                cursor.execute(query, (username_or_email, username_or_email))
                row = cursor.fetchone()
                if not row:
                    return None
                column_names = [desc[0].lower() for desc in cursor.description]
                return dict(zip(column_names, row))
            except Exception as e:
                logger.error(f"Database Query Error in get_user_by_username_or_email: {e}")
                raise
```

*   **Purpose**: Retrieves a user's details from the database using either their username or email, without performing password verification. This is typically a precursor to actual login.
*   **Parameters**:
    *   `username_or_email` (`str`): The username or email address of the user to retrieve.
*   **Return Value**:
    *   A dictionary containing the user's details (e.g., `id`, `username`, `email`, `password_hash`, `full_name`, `user_role`, `team_id`), if found.
    *   `None` if no user matches the provided username or email.
*   **Logic**:
    1.  **Docstring**: Explains the method's specific purpose.
    2.  **`with self.get_cursor() as cursor:`**: Obtains a database cursor.
    3.  **`query = SQLTemplateLoader(f'app/sql_templates/auth').load_template("login")`**: Loads the "login" SQL query, which is expected to select user details based on either username or email.
    4.  **`cursor.execute(query, (username_or_email, username_or_email))`**: Executes the query, passing the input string twice to match against both username and email columns in the SQL template.
    5.  **`row = cursor.fetchone()`**: Fetches a single row (the first match) from the query result.
    6.  **`if not row: return None`**: If no row is found, returns `None`.
    7.  **`column_names = [desc[0].lower() for desc in cursor.description]`**: Extracts and lowercases column names.
    8.  **`return dict(zip(column_names, row))`**: Converts the fetched row into a dictionary and returns it.
    9.  **`except Exception as e:`**: Logs and re-raises any database errors.

#### 4.5. `login` Method

```python
    def login(self, usernameorEmail: str, password: str):
        """Deprecated: Use get_user_by_username_or_email and verify password separately"""
        user = self.get_user_by_username_or_email(usernameorEmail)
        if not user:
            raise HTTPException(status_code=401, detail="User not found")
        if not bcrypt.checkpw(password.encode('utf-8'), user["password_hash"].encode('utf-8')):
            raise HTTPException(status_code=401, detail="Wrong password")
        return user
```

*   **Purpose**: Performs user authentication by checking credentials. This method is marked as deprecated, suggesting that the logic of fetching a user and verifying their password should be handled separately in the application layer.
*   **Parameters**:
    *   `usernameorEmail` (`str`): The username or email of the user attempting to log in.
    *   `password` (`str`): The plain-text password provided by the user.
*   **Return Value**:
    *   A dictionary containing the authenticated user's details.
*   **Raises**:
    *   `HTTPException` (status code 401): If the user is not found or the password is incorrect.
*   **Logic**:
    1.  **Docstring**: Clearly states that the method is deprecated.
    2.  **`user = self.get_user_by_username_or_email(usernameorEmail)`**: Calls the helper method to retrieve user details based on the provided identifier.
    3.  **`if not user: raise HTTPException(...)`**: If `get_user_by_username_or_email` returns `None`, an `HTTPException` is raised, indicating "User not found".
    4.  **`if not bcrypt.checkpw(password.encode('utf-8'), user["password_hash"].encode('utf-8')):`**: Uses `bcrypt.checkpw` to compare the provided plain-text `password` (encoded to UTF-8) against the stored `password_hash` (also encoded). `bcrypt` handles the salting and hashing internally for verification.
    5.  **`raise HTTPException(...)`**: If the password does not match, an `HTTPException` is raised, indicating "Wrong password".
    6.  **`return user`**: If both checks pass, the user dictionary is returned, signifying successful authentication.

#### 4.6. `checkUserExists` Method

```python
    def checkUserExists(self, usernameorEmail: str):
        with self.get_cursor() as cursor:
            try:
                query = SQLTemplateLoader(f'app/sql_templates/auth').load_template("checkUserExists")
                # !!! POTENTIAL BUG: Missing cursor.execute(query, (usernameorEmail,)) call here !!!
                row = cursor.fetchone()
                if not row:
                    return None
                column_names = [desc[0].lower() for desc in cursor.description]
                result = dict(zip(column_names, row))
                return result
            except Exception as e:
                logger.error(f"Database Query Error in checkUserExists: {e}")
                raise
```

*   **Purpose**: Checks if a user with the given username or email exists in the database.
*   **Parameters**:
    *   `usernameorEmail` (`str`): The username or email to check.
*   **Return Value**:
    *   A dictionary containing the user's details if they exist.
    *   `None` if no user is found.
*   **Logic**:
    1.  **`with self.get_cursor() as cursor:`**: Obtains a database cursor.
    2.  **`query = SQLTemplateLoader(f'app/sql_templates/auth').load_template("checkUserExists")`**: Loads the "checkUserExists" SQL query.
    3.  **`!!! POTENTIAL BUG: Missing cursor.execute(...)`**: *Critically, the `cursor.execute(query, (usernameorEmail, usernameorEmail))` call (or similar parameterized execution) is missing after loading the query.* As a result, `cursor.fetchone()` will attempt to fetch a row from a query that was never executed or from a previous query on the same cursor, leading to incorrect behavior (likely always returning `None` or an error depending on the cursor's state).
    4.  **`row = cursor.fetchone()`**: Attempts to fetch a single row.
    5.  **`if not row: return None`**: If no row is found, returns `None`.
    6.  **`column_names = [desc[0].lower() for desc in cursor.description]`**: Extracts and lowercases column names if a row was found.
    7.  **`result = dict(zip(column_names, row))`**: Converts the row to a dictionary.
    8.  **`return result`**: Returns the user dictionary.
    9.  **`except Exception as e:`**: Logs and re-raises any database errors.

#### 4.7. `generatePasswordResetToken` Method

```python
    def generatePasswordResetToken(self, usernameorEmail: str) -> str:
        """Generate and store a password reset token for the user and return it."""
        with self.get_cursor() as cursor:
            try:
                token = str(uuid.uuid4())
                query = SQLTemplateLoader(f'app/sql_templates/auth').load_template("generatePasswordResetToken")

                cursor.execute(query.format(usernameorEmail=usernameorEmail, token=token))
                return token
            except Exception as e:
                logger.error(f"Database Query Error in generatePasswordResetToken: {e}")
                raise
        # finally: # Commented-out connection release
        #     self.pg_pool.putconn(self.conn)
        #     self.conn = None
        #     self.cursor = None
```

*   **Purpose**: Generates a unique password reset token, stores it in the database associated with a user, and returns the token.
*   **Parameters**:
    *   `usernameorEmail` (`str`): The username or email of the user requesting a password reset.
*   **Return Value**:
    *   `str`: The newly generated UUID token.
*   **Logic**:
    1.  **Docstring**: Describes the method's functionality.
    2.  **`with self.get_cursor() as cursor:`**: Obtains a database cursor.
    3.  **`token = str(uuid.uuid4())`**: Generates a new, unique UUID and converts it to a string to be used as a reset token.
    4.  **`query = SQLTemplateLoader(f'app/sql_templates/auth').load_template("generatePasswordResetToken")`**: Loads the SQL query template for generating and storing a password reset token.
    5.  **`cursor.execute(query.format(usernameorEmail=usernameorEmail, token=token))`**: Executes the query.
        *   **CRITICAL IMPLEMENTATION DETAIL**: This line uses f-string formatting (`query.format(...)`) to embed `usernameorEmail` and `token` directly into the SQL query string. **This approach is highly susceptible to SQL Injection vulnerabilities** if `usernameorEmail` originates from untrusted user input and is not properly sanitized. It is strongly recommended to use parameterized queries (`cursor.execute(query, (usernameorEmail, token))`) for security.
    6.  **`return token`**: Returns the generated token.
    7.  **`except Exception as e:`**: Logs and re-raises any database errors.

#### 4.8. `updatePassword` Method

```python
    def updatePassword(self, token: str, password: str):
        """Validate reset token, update user password, and mark token as used."""
        with self.get_cursor() as cursor:
            try:
                # Validate reset token
                validate_sql = "SELECT user_id FROM password_reset_requests WHERE token = %s AND expires_at > NOW() AND is_used = FALSE"
                # ... commented-out connection management ...
                # self.cursor.execute(validate_sql, (token,)) # Commented-out execution
                row = cursor.fetchone()
                if not row:
                    raise Exception("Invalid or expired reset token")
                user_id = row[0]
                # Hash the password before storing it
                hashed_password = bcrypt.hashpw(password.encode('utf-8'), bcrypt.gensalt()).decode('utf-8')
                update_sql = "UPDATE users SET password_hash = %s WHERE id = %s"
                cursor.execute(update_sql, (hashed_password, user_id))
                mark_sql = "UPDATE password_reset_requests SET is_used = TRUE, updated_at = CURRENT_TIMESTAMP WHERE token = %s"
                cursor.execute(mark_sql, (token,))
                # self.conn.commit() # Commented-out explicit commit
            except Exception as e:
                logger.error(f"Database Query Error in updatePassword: {e}")
                raise
        # finally: # Commented-out connection release
        #     self.pg_pool.putconn(self.conn)
        #     self.conn = None
        #     self.cursor = None
```

*   **Purpose**: Validates a password reset token, updates the user's password if the token is valid and unexpired, and then marks the token as used to prevent reuse.
*   **Parameters**:
    *   `token` (`str`): The password reset token provided by the user.
    *   `password` (`str`): The new plain-text password chosen by the user.
*   **Return Value**: None.
*   **Raises**:
    *   `Exception`: If the provided token is invalid, expired, or already used.
*   **Logic**:
    1.  **Docstring**: Outlines the three main steps of the method.
    2.  **`with self.get_cursor() as cursor:`**: Obtains a database cursor.
    3.  **`validate_sql = "SELECT user_id FROM password_reset_requests WHERE token = %s AND expires_at > NOW() AND is_used = FALSE"`**: Defines an inline SQL query to validate the token. It checks for a matching token that is not expired and not yet used.
    4.  **`# self.cursor.execute(validate_sql, (token,))`**: *CRITICAL IMPLEMENTATION DETAIL*: Similar to `checkUserExists`, the `cursor.execute` call for `validate_sql` is commented out or missing. This means `row = cursor.fetchone()` will fetch from a prior query or an empty result, leading to `row` being `None` and the method always raising "Invalid or expired reset token" unless another query was executed immediately before this block on the same cursor. This is a significant bug.
    5.  **`row = cursor.fetchone()`**: Attempts to fetch the result of the (missing) validation query.
    6.  **`if not row: raise Exception(...)`**: If no valid row is found, an exception is raised.
    7.  **`user_id = row[0]`**: If a row is found, extracts the `user_id`.
    8.  **`hashed_password = bcrypt.hashpw(password.encode('utf-8'), bcrypt.gensalt()).decode('utf-8')`**: Hashes the new plain-text password using `bcrypt.gensalt()` for a new salt and `bcrypt.hashpw()`. The result is decoded to a UTF-8 string.
    9.  **`update_sql = "UPDATE users SET password_hash = %s WHERE id = %s"`**: Defines the SQL query to update the user's password.
    10. **`cursor.execute(update_sql, (hashed_password, user_id))`**: Executes the update query with the new hashed password and the retrieved `user_id`.
    11. **`mark_sql = "UPDATE password_reset_requests SET is_used = TRUE, updated_at = CURRENT_TIMESTAMP WHERE token = %s"`**: Defines the SQL query to mark the reset token as used.
    12. **`cursor.execute(mark_sql, (token,))`**: Executes the query to update the token's status.
    13. **Commented-out code**: Again, commented-out lines suggest connection/transaction management is handled by the base class.
    14. **`except Exception as e:`**: Logs and re-raises any database or logical errors during the process.

#### 4.9. `updateTermsAgreed` Method

```python
    def updateTermsAgreed(self, username: str):
        with self.get_cursor() as cursor:
            try:
                query = SQLTemplateLoader(f'app/sql_templates/auth').load_template("updateTermsAgreed")
                # ... commented-out connection management ...
                cursor.execute(query, (username,))
                # selfconn.commit() # Typo: selfconn instead of self.conn. Also commented-out.
            except Exception as e:
                logger.error(f"Database Query Error in updateTermsAgreed: {e}")
                raise
        # finally: # Commented-out connection release
        #     self.pg_pool.putconn(self.conn)
        #     self.conn = None
        #     self.cursor = None
```

*   **Purpose**: Updates a user's record to indicate that they have agreed to the terms and conditions.
*   **Parameters**:
    *   `username` (`str`): The username of the user whose terms agreement status needs to be updated.
*   **Return Value**: None.
*   **Logic**:
    1.  **`with self.get_cursor() as cursor:`**: Obtains a database cursor.
    2.  **`query = SQLTemplateLoader(f'app/sql_templates/auth').load_template("updateTermsAgreed")`**: Loads the "updateTermsAgreed" SQL query.
    3.  **`cursor.execute(query, (username,))`**: Executes the update query using the provided username as a parameter.
    4.  **Commented-out code**: Indicates refactored connection/transaction management. Note a typo `selfconn.commit()`.
    5.  **`except Exception as e:`**: Logs and re-raises any database errors.

### 5. Main Execution Block (`if __name__ == "__main__":`)

The provided script does not contain a `if __name__ == "__main__":` block. This means it is designed to be imported as a module into another application (e.g., a FastAPI application) rather than being executed directly as a standalone script. The `AuthAdapter` class and its methods would be instantiated and called by other parts of the application as needed.

### 6. Key Implementation Details and Considerations

*   **Database Interaction Model (`PGSQLAdapter`, `get_cursor`)**: The `AuthAdapter` inherits from `PGSQLAdapter` and consistently uses `with self.get_cursor() as cursor:`. This pattern strongly suggests that `PGSQLAdapter` provides a robust mechanism for obtaining and managing database connections and cursors, likely involving a connection pool (e.g., `psycopg2.pool`) and handling transaction boundaries (commits/rollbacks) or autocommit behavior within the context manager.
*   **SQL Template Usage (`SQLTemplateLoader`)**: All database queries, except for the `updatePassword` validation, are loaded from external SQL template files using `SQLTemplateLoader`. This promotes cleaner code, easier management of complex SQL, and separation of concerns.
*   **Password Hashing (`bcrypt`)**: The `bcrypt` library is correctly used for hashing and verifying passwords. `bcrypt.gensalt()` ensures a unique salt for each password hash, enhancing security.
*   **Error Handling**: All database interaction methods include `try...except` blocks to catch `Exception` during query execution. Errors are logged using the `logger` instance and then re-raised, allowing calling code to handle them appropriately.
*   **Deprecated `login` Method**: The `login` method is explicitly marked as deprecated. This is good practice, indicating that its functionality (combining user fetch and password verification) should be refactored into separate steps in the application layer for better flexibility, testability, and potentially to align with API authentication flows (e.g., JWT token generation after successful password verification).
*   **Potential Bug in `checkUserExists` and `updatePassword` (`validate_sql` section)**: Both `checkUserExists` and the token validation part of `updatePassword` are missing the crucial `cursor.execute()` call after loading or defining the SQL query. As written, `cursor.fetchone()` will operate on an uninitialized result set or the result of a previous query on the same cursor, leading to logical errors. This should be corrected by adding `cursor.execute(query, (usernameorEmail, usernameorEmail))` for `checkUserExists` and `cursor.execute(validate_sql, (token,))` for `updatePassword`.
*   **SQL Injection Vulnerability in `generatePasswordResetToken`**: The method `generatePasswordResetToken` uses `query.format(...)` to construct its SQL query. This is a severe security risk. If `usernameorEmail` is user-provided and not sanitized, an attacker could inject malicious SQL. This should be refactored to use a parameterized query: `cursor.execute(query, (usernameorEmail, token))` (assuming the SQL template uses `%s` or `?` placeholders).
*   **Commented-out Connection Management**: Numerous commented-out lines throughout the methods indicate an earlier manual approach to connection management (e.g., `self.pg_pool = get_db_pool()`, `self.conn.autocommit = True`, `self.conn.commit()`, `self.pg_pool.putconn(self.conn)`). These lines are now commented, strongly suggesting that the `PGSQLAdapter` base class's `get_cursor()` context manager is responsible for handling these concerns, providing a cleaner and potentially more robust approach to connection pooling and transaction management.
*   **FastAPI Integration**: The use of `fastapi.HTTPException` implies that this `AuthAdapter` is intended to be used within a FastAPI web application, allowing it to raise API-specific error responses directly.
