# Documentation for `error_handler.py`

This document provides comprehensive technical documentation for the `error_handling_utilities.py` module, which is designed to manage and translate raw database and system error messages into user-friendly responses within the R2 API.

---

## Technical Documentation: R2 API Error Handling Utilities

### 1. Overview

The `error_handling_utilities.py` module serves as a centralized mechanism for robust error management within the R2 API. Its primary purpose is to intercept various types of exceptions, particularly those originating from database operations, and transform their often cryptic messages into clear, actionable, and user-friendly messages for API consumers. It integrates with FastAPI's `HTTPException` to ensure consistent error responses across the application.

### 2. Module Docstring

```python
"""
Error handling utilities for the R2 API.

This module provides functions to handle common database errors and convert them
into user-friendly error messages.
"""
```
The module's docstring clearly states its purpose: to provide utilities for error handling within the R2 API, specifically focusing on converting common database errors into user-friendly messages.

### 3. Imports

The module relies on several standard and third-party libraries:

*   **`import re`**:
    *   **Purpose**: The `re` module provides regular expression operations. It is used here to define and match patterns within raw error messages, allowing for flexible and robust identification of specific error types (e.g., unique constraint violations, foreign key errors).
*   **`import logging`**:
    *   **Purpose**: The `logging` module provides a flexible framework for emitting log messages from Python programs. It is used to log original, raw error messages for debugging purposes, while user-friendly messages are returned to the API client.
*   **`from fastapi import HTTPException`**:
    *   **Purpose**: `HTTPException` is a specific exception class provided by the FastAPI framework. It is used to raise HTTP-specific errors (e.g., 400 Bad Request, 404 Not Found, 500 Internal Server Error) that FastAPI automatically converts into appropriate HTTP responses. This module uses it to encapsulate user-friendly error messages with relevant HTTP status codes.

### 4. Global Variables and Constants

#### `logger`

```python
logger = logging.getLogger(__name__)
```
*   **Purpose**: Initializes a logger instance for this module. `logging.getLogger(__name__)` ensures that the logger's name is set to the module's name (`error_handling_utilities`), which is good practice for managing log messages from different parts of an application. This logger is used to record details of the original errors for internal debugging, separate from the user-facing messages.

#### `ERROR_PATTERNS`

```python
ERROR_PATTERNS = {
    # ... dictionary contents ...
}
```
*   **Type**: `dict`
*   **Purpose**: This dictionary is a central configuration point for mapping database error message patterns (using regular expressions) to corresponding user-friendly error messages. It allows the application to abstract away the technical details of database errors and present a more understandable message to the end-user. Each key is a regular expression string, and each value is the user-friendly message string. Some friendly messages use placeholder formatting (`{0}`) which will be filled by capture groups from the regex pattern.

*   **Detailed Breakdown of `ERROR_PATTERNS` Entries:**

    *   **Unique constraint violations:**
        *   `r"duplicate key value violates unique constraint \"users_email_key\""`:
            *   **Matches**: PostgreSQL unique constraint violation error specifically for the `email` column in the `users` table.
            *   **Friendly Message**: `"This email address is already registered. Please use a different email."`
        *   `r"duplicate key value violates unique constraint \"users_username_key\""`:
            *   **Matches**: PostgreSQL unique constraint violation error specifically for the `username` column in the `users` table.
            *   **Friendly Message**: `"This username is already taken. Please choose a different username."`

    *   **Foreign key constraint violations:**
        *   `r"violates foreign key constraint \"users_team_id_fkey\""`:
            *   **Matches**: PostgreSQL foreign key constraint violation error related to the `team_id` in the `users` table.
            *   **Friendly Message**: `"The selected team does not exist. Please choose a valid team."`

    *   **Not null constraint violations:**
        *   `r"null value in column \"(\w+)\" violates not-null constraint"`:
            *   **Matches**: PostgreSQL `NOT NULL` constraint violation. It uses a **capture group `(\w+)`** to extract the name of the column that violated the constraint.
            *   **Friendly Message**: `"Required field '{0}' is missing. Please provide all required information."` (The `{0}` will be replaced by the captured column name).

    *   **Check constraint violations:**
        *   `r"new row for relation \"users\" violates check constraint"`:
            *   **Matches**: PostgreSQL `CHECK` constraint violation specifically for the `users` table. These constraints define rules that data in a column must satisfy.
            *   **Friendly Message**: `"Invalid input data. Please check your information and try again."`

    *   **Connection errors:**
        *   `r"could not connect to server"`:
            *   **Matches**: Generic database connection failure message.
            *   **Friendly Message**: `"Unable to connect to the database. Please try again later."`

    *   **Authentication errors:**
        *   `r"password authentication failed"`:
            *   **Matches**: Database authentication failure message (e.g., incorrect password).
            *   **Friendly Message**: `"Invalid credentials. Please check your username and password."`

    *   **General duplicate errors (catch-all):**
        *   `r"duplicate key value violates unique constraint"`:
            *   **Matches**: A more general pattern for any PostgreSQL unique constraint violation, serving as a fallback if more specific unique constraint patterns are not matched.
            *   **Friendly Message**: `"This record already exists. Please check your data and try again."`

    *   **General errors:**
        *   `r"relation \"(\w+)\" does not exist"`:
            *   **Matches**: Error indicating that a database table (relation) does not exist. Uses a **capture group `(\w+)`** to extract the name of the missing relation. This often indicates a serious configuration or deployment issue.
            *   **Friendly Message**: `"System error: Database table not found. Please contact support."`

### 5. Functions

#### `get_friendly_error_message(error_message)`

```python
def get_friendly_error_message(error_message):
    """
    Convert a database error message to a user-friendly message.
    
    Args:
        error_message (str): The original error message
        
    Returns:
        str: A user-friendly error message
    """
    # ... implementation ...
```

*   **Purpose**: This function takes a raw, technical error message (typically from a database or system exception) and attempts to convert it into a more understandable, user-friendly string based on predefined patterns.
*   **Parameters**:
    *   `error_message` (`str`): The original, technical error message to be processed.
*   **Returns**:
    *   `str`: A user-friendly error message. If no specific pattern matches, a generic message is returned.
*   **Implementation Logic**:
    1.  **`if not error_message:`**:
        *   Checks if the input `error_message` is empty or `None`.
        *   If so, it immediately returns a generic "An unexpected error occurred..." message, preventing further processing of an invalid input.
    2.  **`logger.error(f"Original error: {error_message}")`**:
        *   Logs the full original error message at an `ERROR` level. This is crucial for debugging, allowing developers to see the exact technical error even when a friendly message is shown to the user.
    3.  **`for pattern, friendly_message in ERROR_PATTERNS.items():`**:
        *   Iterates through each key-value pair in the `ERROR_PATTERNS` dictionary.
        *   `pattern` is the regular expression string.
        *   `friendly_message` is the corresponding user-friendly message string.
    4.  **`match = re.search(pattern, str(error_message))`**:
        *   For each pattern, it attempts to find a match within the `error_message` string using `re.search()`.
        *   `str(error_message)` is used to ensure the `error_message` is a string, as some exceptions might have non-string representations.
        *   If a match is found, `match` will be a `re.MatchObject`; otherwise, it will be `None`.
    5.  **`if match:`**:
        *   If a pattern successfully matches the `error_message`:
            *   **`if match.groups():`**:
                *   Checks if the regex `pattern` contained any capture groups (e.g., `(\w+)`).
                *   If `match.groups()` returns a non-empty tuple (meaning capture groups exist), the `friendly_message` is formatted using these captured values. This allows dynamic message generation (e.g., "Required field 'username' is missing.").
                *   `friendly_message.format(*match.groups())` unpacks the captured groups and inserts them into the format string.
            *   **`return friendly_message`**:
                *   If no capture groups, or after formatting, the determined `friendly_message` is returned immediately, as the most specific error has been identified and processed.
    6.  **`return "An unexpected error occurred. Please try again later."`**:
        *   If the loop completes without finding any matching pattern in `ERROR_PATTERNS`, this generic fallback message is returned.

#### `handle_exception(exception, status_code=500)`

```python
def handle_exception(exception, status_code=500):
    """
    Handle an exception and return an appropriate HTTPException.

    If it's already an HTTPException, re-raise it so the original status and message are preserved.
    """
    # ... implementation ...
```

*   **Purpose**: This function acts as a centralized exception handler, primarily designed for use in API endpoints or middleware. It takes any exception, determines its type, and either re-raises existing `HTTPException` instances or converts other exceptions into a FastAPI `HTTPException` with a user-friendly message and a specified HTTP status code.
*   **Parameters**:
    *   `exception` (`Exception`): The exception object that was caught.
    *   `status_code` (`int`, optional): The HTTP status code to associate with the `HTTPException` if a new one is created. Defaults to `500` (Internal Server Error) for generic unhandled exceptions.
*   **Returns**:
    *   `HTTPException`: A FastAPI `HTTPException` object if a new one is created.
*   **Implementation Logic**:
    1.  **`if isinstance(exception, HTTPException):`**:
        *   Checks if the caught `exception` is already an instance of FastAPI's `HTTPException`.
        *   **`logger.error(f"Re-raising HTTPException: {exception.detail}")`**:
            *   Logs the detail of the `HTTPException`. This helps in debugging cases where an `HTTPException` might be caught and then re-raised.
        *   **`raise exception`**:
            *   If the exception is already an `HTTPException`, it is re-raised. This is critical because `HTTPException` instances already contain the desired status code and detail message, and re-raising allows FastAPI's internal exception handlers to process them directly, preserving their original intent.
    2.  **`friendly_message = get_friendly_error_message(str(exception))`**:
        *   If the `exception` is *not* an `HTTPException`, its string representation (`str(exception)`) is passed to `get_friendly_error_message()` to obtain a user-friendly message.
    3.  **`logger.error(f"Error handled: {exception} -> {friendly_message}")`**:
        *   Logs both the original exception (its `repr` or `str` representation) and the `friendly_message` that was generated. This provides a clear audit trail for developers.
    4.  **`return HTTPException(status_code=status_code, detail=friendly_message)`**:
        *   A new `HTTPException` is created and returned. This `HTTPException` encapsulates the `status_code` (either the default 500 or one provided by the caller) and the `friendly_message`. FastAPI will then convert this `HTTPException` into a standard HTTP error response.

### 6. Main Execution Flow

This module does not contain a `if __name__ == "__main__":` block, indicating it is not designed to be run as a standalone script. Instead, it is intended to be imported and used as a utility library by other parts of the R2 API application, specifically within FastAPI routes or exception handling middleware.

### 7. Dependencies

*   Python standard library: `re`, `logging`
*   Third-party library: `FastAPI` (specifically `fastapi.HTTPException`)

### 8. Important Implementation Details

*   **Regex-based Pattern Matching**: The use of `re.search` with `ERROR_PATTERNS` provides a powerful and flexible way to identify specific error conditions without relying on exact string matches, which can be brittle with varying database error messages.
*   **Separation of Concerns**: The module clearly separates the logic for converting raw error messages (`get_friendly_error_message`) from the logic for handling exceptions within an API context (`handle_exception`). This enhances modularity and testability.
*   **Logging for Debugging**: Crucially, the original, technical error messages are logged internally. This ensures that while users receive helpful messages, developers still have access to the raw details needed for troubleshooting and debugging.
*   **FastAPI Integration**: The module is specifically designed to integrate seamlessly with FastAPI by generating or re-raising `HTTPException` instances, allowing FastAPI's built-in error handling to format these into consistent JSON responses.
*   **Extensibility**: The `ERROR_PATTERNS` dictionary is easily extendable. New error patterns and their corresponding friendly messages can be added without modifying the core logic of the functions, making the error handling robust and maintainable.
*   **Dynamic Messages**: The ability to use capture groups in regex patterns and `str.format()` allows for dynamic, context-specific error messages (e.g., specifying which field is missing).
