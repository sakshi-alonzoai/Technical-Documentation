# Documentation for `mail_service.py`

This document provides comprehensive technical documentation for the Python script designed for sending various types of transactional emails, primarily within the context of a web application (e.g., built with FastAPI).

---

## Technical Documentation: Email Service Module

### 1. High-Level Overview

This Python script (`email_service.py` or similar) provides a robust and asynchronous email sending utility. Its primary purpose is to facilitate the sending of transactional emails, such as registration confirmations, password reset links, and administrative notifications, often in the background to avoid blocking the main application's execution flow. It leverages SMTP for email communication and integrates with FastAPI's `BackgroundTasks` for non-blocking operations, falling back to `asyncio` for environments where `BackgroundTasks` might not be explicitly passed.

The script is highly configurable via environment variables, ensuring sensitive information like SMTP credentials is not hardcoded. It defines several helper functions for common email types related to user registration and college onboarding processes for an application named "AthlyteSports".

### 2. Dependencies and Imports

The script relies on several standard and third-party Python libraries:

*   `import smtplib`:
    *   **Purpose:** Standard Python library for sending emails using the Simple Mail Transfer Protocol (SMTP). It handles the actual communication with an SMTP server.
*   `import os`:
    *   **Purpose:** Standard Python library for interacting with the operating system. It's used here primarily for accessing environment variables (`os.getenv`).
*   `import logging`:
    *   **Purpose:** Standard Python library for flexible event logging. It's used to log information messages (e.g., email sent successfully) and error messages (e.g., email sending failures).
*   `from email.message import EmailMessage`:
    *   **Purpose:** Part of Python's email package, this class provides a high-level API for creating and manipulating email messages, making it easy to set subjects, recipients, and content.
*   `from dotenv import load_dotenv`:
    *   **Purpose:** A third-party library (`python-dotenv`) that loads environment variables from a `.env` file into `os.environ`. This is crucial for local development and configuration management without hardcoding secrets.
*   `from decouple import config`:
    *   **Purpose:** A third-party library (`python-decouple`) that helps organize settings by separating parameters from code, often reading from environment variables or `.env` files. It provides more robust handling of default values and type casting compared to `os.getenv`.
*   `from fastapi import BackgroundTasks`:
    *   **Purpose:** A specific import from the FastAPI framework. `BackgroundTasks` is used to run a function in the background *after* returning a response to the client, preventing the email sending process from delaying the HTTP response.
*   `import asyncio`:
    *   **Purpose:** Standard Python library for writing concurrent code using the async/await syntax. It's used here to run synchronous email sending operations in a separate thread when `BackgroundTasks` is not available, ensuring non-blocking behavior.
*   `from typing import Dict, Any`:
    *   **Purpose:** Standard Python library for type hints. `Dict` is used to indicate a dictionary, and `Any` indicates that a value can be of any type, improving code readability and maintainability.

### 3. Configuration and Global Variables

The script heavily relies on environment variables for configuration. `load_dotenv()` is called at the beginning to ensure these variables are loaded.

*   **Logging Setup:**
    ```python
    logging.basicConfig(level=logging.INFO)
    logger = logging.getLogger(__name__)
    ```
    *   Configures the root logger to output messages at `INFO` level and above.
    *   `logger = logging.getLogger(__name__)` creates a named logger for this module, which is good practice for more granular control over logging in larger applications.

*   **Environment Variable Loading:**
    ```python
    load_dotenv()
    ```
    *   This line calls the `load_dotenv` function, which scans the current directory and its parents for a `.env` file and loads any key-value pairs found within it into the application's environment variables. This allows for flexible configuration without changing the codebase.

*   **SMTP Server Configuration:**
    ```python
    SMTP_SERVER = os.getenv("SMTP_SERVER", "smtp.gmail.com")
    SMTP_PORT = int(os.getenv("SMTP_PORT", 587))
    SMTP_USERNAME = os.getenv("SMTP_USERNAME")
    SMTP_PASSWORD = os.getenv("SMTP_PASSWORD")
    FROM_EMAIL = os.getenv("FROM_EMAIL")
    ADMIN_EMAIL = os.getenv("ADMIN_EMAIL", "srikari@alonzoai.in")
    ```
    *   `SMTP_SERVER`: The hostname or IP address of the SMTP server. Defaults to "smtp.gmail.com" if not provided in environment variables.
    *   `SMTP_PORT`: The port number for the SMTP server. Defaults to 587 (standard for TLS/STARTTLS) if not provided. The value is cast to an integer.
    *   `SMTP_USERNAME`: The username for authenticating with the SMTP server. **Must be provided** via environment variables (e.g., `SMTP_USERNAME="your_email@example.com"`).
    *   `SMTP_PASSWORD`: The password for authenticating with the SMTP server. **Must be provided** via environment variables (e.g., `SMTP_PASSWORD="your_app_password"`).
    *   `FROM_EMAIL`: The email address that will appear as the sender of the emails. **Must be provided** via environment variables.
    *   `ADMIN_EMAIL`: The default email address for administrators to receive internal notifications. Defaults to "srikari@alonzoai.in" if not provided.

*   **Reset Password URL Configuration:**
    ```python
    RESET_PASSWORD_URL = config("RESET_PASSWORD_URL", default="http://localhost:3000/reset-password")
    ```
    *   `RESET_PASSWORD_URL`: The base URL for the password reset page in the client application. Uses `decouple.config` for potentially more robust variable loading, with a default of "http://localhost:3000/reset-password" for development.

### 4. Function Definitions

This section details each function, explaining its purpose, parameters, return values, and implementation logic.

#### 4.1. `_send_email_sync(to_email: str, subject: str, body: str) -> Dict[str, Any]`

*   **Purpose:** This is the core synchronous function responsible for the actual communication with the SMTP server to send an email. It's designed to be called by background tasks or asynchronously wrapped functions.
*   **Parameters:**
    *   `to_email` (`str`): The recipient's email address.
    *   `subject` (`str`): The subject line of the email.
    *   `body` (`str`): The main content of the email.
*   **Return Value:** (`Dict[str, Any]`) A dictionary indicating the status of the email sending operation.
    *   On success: `{"status": "success", "message": "Email sent successfully"}`
    *   On failure: `{"status": "error", "message": str(e)}` where `e` is the exception object.
*   **Implementation Logic:**
    1.  **Email Message Creation:** An `EmailMessage` object is instantiated.
    2.  **Header Population:** The `Subject`, `From`, and `To` headers are set using the provided arguments and the `FROM_EMAIL` global variable.
    3.  **Content Setting:** The `set_content()` method is used to set the email body.
    4.  **SMTP Connection:**
        *   It uses a `with` statement to ensure the SMTP connection is properly closed after use.
        *   `smtplib.SMTP(SMTP_SERVER, SMTP_PORT)` establishes a connection to the specified SMTP server and port.
        *   `server.starttls()`: Initiates Transport Layer Security (TLS) negotiation, upgrading the connection from plain text to an encrypted one. This is crucial for securing credentials and email content.
        *   `server.login(SMTP_USERNAME, SMTP_PASSWORD)`: Authenticates with the SMTP server using the provided username and password.
        *   `server.send_message(msg)`: Sends the constructed `EmailMessage` object.
    5.  **Logging and Return:**
        *   If successful, an `INFO` level log message is recorded, and a success dictionary is returned.
        *   If any `Exception` occurs during the process (e.g., connection error, authentication failure), an `ERROR` level log message is recorded with the exception details, and an error dictionary is returned.

#### 4.2. `async def send_email(to_email: str, subject: str, body: str, background_tasks: BackgroundTasks = None) -> Dict[str, Any]`

*   **Purpose:** This is the public asynchronous interface for sending emails. It abstracts away the synchronous email sending logic by delegating it to either FastAPI's `BackgroundTasks` or a new `asyncio` task, ensuring the main application thread remains unblocked.
*   **Parameters:**
    *   `to_email` (`str`): The recipient's email address.
    *   `subject` (`str`): The subject line of the email.
    *   `body` (`str`): The main content of the email.
    *   `background_tasks` (`BackgroundTasks`, optional): An instance of FastAPI's `BackgroundTasks` dependency. Defaults to `None`.
*   **Return Value:** (`Dict[str, Any]`) A dictionary indicating that the email has been queued for delivery.
    *   Returns: `{"status": "queued", "message": "Email queued for delivery"}`
*   **Implementation Logic:**
    1.  **BackgroundTasks Check:**
        *   `if background_tasks:`: Checks if a `BackgroundTasks` object was provided (typically by a FastAPI endpoint).
        *   `background_tasks.add_task(_send_email_sync, to_email, subject, body)`: If provided, the synchronous `_send_email_sync` function is added as a background task. This means FastAPI will execute this task *after* sending the HTTP response to the client, without blocking the response.
    2.  **Asyncio Fallback:**
        *   `else:`: If `background_tasks` is `None` (e.g., when calling this function outside a FastAPI request context or without passing `BackgroundTasks`), it falls back to `asyncio`.
        *   `asyncio.create_task(_send_email_async(to_email, subject, body))`: Creates a new `asyncio` task to run the `_send_email_async` wrapper. This allows the email sending to happen concurrently in the background without blocking the caller.
    3.  **Return Queued Status:** In both cases, the function immediately returns a "queued" status, as the actual sending happens in the background.

#### 4.3. `async def _send_email_async(to_email: str, subject: str, body: str) -> Dict[str, Any]`

*   **Purpose:** An asynchronous wrapper for the `_send_email_sync` function. It enables the synchronous email sending logic to be executed in a separate thread, preventing it from blocking the `asyncio` event loop. This is used when `BackgroundTasks` is not available.
*   **Parameters:**
    *   `to_email` (`str`): The recipient's email address.
    *   `subject` (`str`): The subject line of the email.
    *   `body` (`str`): The main content of the email.
*   **Return Value:** (`Dict[str, Any]`) The result dictionary returned by `_send_email_sync`.
*   **Implementation Logic:**
    1.  `loop = asyncio.get_event_loop()`: Retrieves the current event loop.
    2.  `return await loop.run_in_executor(None, _send_email_sync, to_email, subject, body)`:
        *   `run_in_executor(executor, func, *args)`: This method schedules a callable `func` to be run in a separate thread.
        *   `None` as the first argument indicates that the default thread pool executor should be used.
        *   `_send_email_sync` is the synchronous function to execute.
        *   `to_email, subject, body` are the arguments passed to `_send_email_sync`.
        *   `await` ensures that the current coroutine pauses until `_send_email_sync` completes its execution in the background thread, and then the result is returned.

#### 4.4. `async def registration_request_received_email(to_email: str, username: str, background_tasks: BackgroundTasks = None)`

*   **Purpose:** Sends a welcome email to a user immediately after their registration request has been received, informing them that their request is under review.
*   **Parameters:**
    *   `to_email` (`str`): The email address of the user.
    *   `username` (`str`): The username of the newly registered user.
    *   `background_tasks` (`BackgroundTasks`, optional): FastAPI `BackgroundTasks` instance.
*   **Return Value:** The result dictionary from `send_email`.
*   **Implementation Logic:**
    1.  Defines a `subject` and a multi-line `body` string, personalizing it with the `username`.
    2.  Calls the `send_email` function with the constructed subject, body, and the provided `background_tasks`.

#### 4.5. `async def registration_request_received_email_to_admin(username: str, background_tasks: BackgroundTasks = None)`

*   **Purpose:** Notifies the administrator(s) that a new user registration request has been received and requires review.
*   **Parameters:**
    *   `username` (`str`): The username of the newly registered user.
    *   `background_tasks` (`BackgroundTasks`, optional): FastAPI `BackgroundTasks` instance.
*   **Return Value:** The result dictionary from `send_email`.
*   **Implementation Logic:**
    1.  Defines a `subject` and a multi-line `body` string, including the `username`.
    2.  Calls the `send_email` function, directing the email to the `ADMIN_EMAIL` global variable, and passes the `background_tasks`.

#### 4.6. `async def registration_confirmation_email(to_email: str, username: str, reset_token: str, background_tasks: BackgroundTasks = None)`

*   **Purpose:** Sends a confirmation email to a user after their registration has been approved. This email includes a link to set their initial password using a reset token.
*   **Parameters:**
    *   `to_email` (`str`): The email address of the confirmed user.
    *   `username` (`str`): The username of the confirmed user.
    *   `reset_token` (`str`): A unique token used to authenticate and allow the user to reset their password.
    *   `background_tasks` (`BackgroundTasks`, optional): FastAPI `BackgroundTasks` instance.
*   **Return Value:** The result dictionary from `send_email`.
*   **Implementation Logic:**
    1.  Defines a `subject` and a multi-line `body` string.
    2.  Constructs the password reset URL by appending the `reset_token` as a query parameter to `RESET_PASSWORD_URL`.
    3.  Mentions that the link will expire in 1 hour.
    4.  Calls the `send_email` function with the constructed subject, body, and the provided `background_tasks`.

#### 4.7. `async def college_onboard_request_mail_toadmin(college_name: str, user: str, useremail: str, background_tasks: BackgroundTasks = None)`

*   **Purpose:** Notifies the administrator(s) about a new request to onboard a college into the system.
*   **Parameters:**
    *   `college_name` (`str`): The name of the college requesting onboarding.
    *   `user` (`str`): The name of the contact person for the college.
    *   `useremail` (`str`): The email of the contact person for the college.
    *   `background_tasks` (`BackgroundTasks`, optional): FastAPI `BackgroundTasks` instance.
*   **Return Value:** The result dictionary from `send_email`.
*   **Implementation Logic:**
    1.  Defines a `subject` and a multi-line `body` string, including details about the `college_name`, contact `user`, and `useremail`.
    2.  Calls the `send_email` function, directing the email to the `ADMIN_EMAIL` global variable, and passes the `background_tasks`.

#### 4.8. `async def college_onboard_request_mail_to_user(college_name: str, user: str, useremail: str, background_tasks: BackgroundTasks = None)`

*   **Purpose:** Sends a confirmation email to the user who submitted a college onboarding request, acknowledging receipt and informing them of the review process.
*   **Parameters:**
    *   `college_name` (`str`): The name of the college requesting onboarding.
    *   `user` (`str`): The name of the contact person for the college.
    *   `useremail` (`str`): The email of the contact person for the college.
    *   `background_tasks` (`BackgroundTasks`, optional): FastAPI `BackgroundTasks` instance.
*   **Return Value:** The result dictionary from `send_email`.
*   **Implementation Logic:**
    1.  Defines a `subject` and a multi-line `body` string, including details about the `college_name`, contact `user`, and `useremail`.
    2.  Calls the `send_email` function, directing the email to the `useremail` (the contact person), and passes the `background_tasks`.

#### 4.9. `async def password_reset_request_mail(to_email: str, username: str, reset_token: str, background_tasks: BackgroundTasks = None)`

*   **Purpose:** Sends a password reset email to a user, providing a link to reset their password.
*   **Parameters:**
    *   `to_email` (`str`): The email address of the user requesting the reset.
    *   `username` (`str`): The username of the user.
    *   `reset_token` (`str`): A unique token used to authenticate and allow the user to reset their password.
    *   `background_tasks` (`BackgroundTasks`, optional): FastAPI `BackgroundTasks` instance.
*   **Return Value:** The result dictionary from `send_email`.
*   **Implementation Logic:**
    1.  Defines a `subject` and a multi-line `body` string.
    2.  Constructs the password reset URL by appending the `reset_token` as a query parameter to `RESET_PASSWORD_URL`.
    3.  Mentions that the link will expire in 15 minutes and provides contact information for unauthorized requests.
    4.  Calls the `send_email` function with the constructed subject, body, and the provided `background_tasks`.

### 5. Main Execution Flow

This script does not contain a traditional `if __name__ == "__main__":` block for direct execution. Instead, it is designed to be imported and used as a module within a larger Python application, most likely a web framework like FastAPI.

The intended usage pattern is:
1.  An endpoint in the web application (e.g., a FastAPI route for user registration or password reset) receives a request.
2.  After processing the core logic of the request (e.g., creating a user, generating a token), it calls one of the `async` email helper functions (e.g., `registration_confirmation_email`).
3.  The web application provides a `BackgroundTasks` object to the email function.
4.  The email function adds the actual synchronous email sending to these background tasks, allowing the web application to immediately return an HTTP response to the client while the email is sent concurrently in the background.
5.  If `BackgroundTasks` is not available, the `send_email` function falls back to using `asyncio.create_task` and `loop.run_in_executor` to achieve a similar non-blocking background operation.

### 6. Important Implementation Details and Considerations

*   **Asynchronous Design:** The core design principle is to make email sending non-blocking. This is critical for web applications where long-running operations like external API calls (like SMTP) can severely impact response times and user experience.
*   **FastAPI Integration:** The use of `fastapi.BackgroundTasks` provides an elegant and integrated way to handle background jobs directly within FastAPI request handlers.
*   **`asyncio.run_in_executor`:** This is a crucial `asyncio` feature for integrating synchronous I/O-bound operations (like `smtplib`'s network calls) into an `async` application without blocking the event loop. It effectively runs the synchronous code in a separate thread.
*   **Environment Variable Configuration:** All sensitive information (SMTP credentials) and changeable configurations (server, port, URLs) are externalized into environment variables. This is a security best practice, preventing secrets from being committed to source control and allowing easy deployment across different environments (development, staging, production).
*   **Error Handling and Logging:** The `_send_email_sync` function includes `try-except` blocks to catch potential errors during email transmission and logs these errors, providing visibility into failures without crashing the application. Success messages are also logged.
*   **Email Content Formatting:** The email bodies are defined as f-strings, allowing for easy personalization with dynamic data like usernames and reset tokens. For more complex HTML emails, the `EmailMessage` object supports `add_alternative` for different content types.
*   **Token Expiration:** The email templates explicitly mention the expiration time for password reset links (15 minutes) and registration confirmation links (1 hour). This is a good security practice, but the actual token validity enforcement must be handled by the backend authentication system.
*   **`python-dotenv` and `python-decouple`:** These libraries enhance environment variable management. `python-dotenv` loads `.env` files, and `python-decouple` provides a unified interface for configuration, often with improved default handling and type casting capabilities over `os.getenv`.
