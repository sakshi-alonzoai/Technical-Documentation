# Documentation for `hubspot_service.py`

This document provides a comprehensive, line-by-line technical explanation of the provided Python script, detailing its purpose, implementation, and usage.

---

## 1. High-Level Overview

This Python script provides a robust and flexible solution for creating tickets in HubSpot. It is designed to be integrated into applications, particularly web services built with frameworks like FastAPI, by offering both synchronous and asynchronous methods for ticket creation.

The core functionality involves making HTTP `POST` requests to the HubSpot CRM API to create new ticket objects. To prevent blocking the main application thread during I/O-bound operations (like network requests), it leverages `asyncio` for asynchronous execution and integrates with FastAPI's `BackgroundTasks` to offload ticket creation to a separate thread or background process. It also includes a specialized function for creating detailed "issue report" tickets.

## 2. Module Imports

The script begins by importing necessary modules from Python's standard library and external packages:

*   `import os`: Provides a way of using operating system dependent functionality, primarily for accessing environment variables.
*   `import logging`: Standard library for flexible event logging. Used to record informational, warning, and error messages.
*   `import requests`: A popular third-party library for making HTTP requests. It simplifies interacting with web services.
*   `from typing import Dict, Any, Optional`: Used for type hinting, which improves code readability and maintainability by explicitly declaring expected data types for variables, function parameters, and return values.
    *   `Dict`: Represents a dictionary.
    *   `Any`: Represents any type.
    *   `Optional`: Indicates that a value can either be of the specified type or `None`.
*   `from dotenv import load_dotenv`: A third-party library used to load environment variables from a `.env` file into `os.environ`. This is crucial for managing sensitive information like API tokens outside of the codebase.
*   `from fastapi import BackgroundTasks, HTTPException`: Imports specific components from the FastAPI framework.
    *   `BackgroundTasks`: A utility provided by FastAPI to run functions in the background after the HTTP response has been sent. This prevents long-running tasks from delaying the client's response.
    *   `HTTPException`: A class used to raise standard HTTP errors with specific status codes and detail messages, which FastAPI then translates into appropriate HTTP responses.
*   `import asyncio`: Python's standard library for writing concurrent code using the `async/await` syntax. It is used here to run synchronous code in an asynchronous context without blocking the event loop.

## 3. Global Configuration and Setup

This section configures logging, loads environment variables, and defines HubSpot API constants.

```python
# Setup logging
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

# Load environment variables
load_dotenv()

# HubSpot API configuration
HUBSPOT_TOKEN = os.getenv("HUBSPOT_TOKEN")
HUBSPOT_TICKET_ENDPOINT = "https://api.hubapi.com/crm/v3/objects/tickets"
```

*   **`# Setup logging`**:
    *   `logging.basicConfig(level=logging.INFO)`: Configures the root logger. It sets the default logging level to `INFO`, meaning all messages of level `INFO`, `WARNING`, `ERROR`, and `CRITICAL` will be processed. By default, messages are printed to the console (standard error).
    *   `logger = logging.getLogger(__name__)`: Creates a logger instance for the current module. Using `__name__` ensures that log messages specify the module from which they originated, which is useful for debugging.
*   **`# Load environment variables`**:
    *   `load_dotenv()`: Calls the `load_dotenv` function from the `dotenv` package. This function searches for a `.env` file in the current directory (or parent directories) and loads any key-value pairs found there into the system's environment variables (`os.environ`).
*   **`# HubSpot API configuration`**:
    *   `HUBSPOT_TOKEN = os.getenv("HUBSPOT_TOKEN")`: Retrieves the HubSpot API access token from the environment variables. It expects a variable named `HUBSPOT_TOKEN` to be set, typically loaded from the `.env` file. This token is essential for authenticating requests to the HubSpot API.
    *   `HUBSPOT_TICKET_ENDPOINT = "https://api.hubapi.com/crm/v3/objects/tickets"`: Defines the specific HubSpot API endpoint URL for creating new tickets. This is a constant string used in API requests.

## 4. Function Definitions

### 4.1. `_create_ticket_sync`

```python
def _create_ticket_sync(
    title: str, 
    description: str, 
    properties: Optional[Dict[str, Any]] = None,
    pipeline: str = "0",  # Default Support pipeline
    pipeline_stage: str = "1"  # Default 'New' stage
) -> Dict[str, Any]:
    """Synchronous function to create a ticket in HubSpot.
    
    Args:
        title: The subject of the ticket
        description: The content/description of the ticket
        properties: Additional ticket properties
        pipeline: HubSpot pipeline ID (default: "0" for Support pipeline)
        pipeline_stage: Pipeline stage ID (default: "1" for 'New' stage)
        
    Returns:
        Dictionary containing status, message, and ticket details
    """
    if not HUBSPOT_TOKEN:
        error_msg = "HubSpot API token not configured"
        logger.error(error_msg)
        raise HTTPException(status_code=500, detail=error_msg)

    try:
        ticket_props = {
            "subject": title,
            "content": description,
            "hs_pipeline": pipeline,
            "hs_pipeline_stage": pipeline_stage,
        }

        if properties:
            ticket_props.update(properties)

        headers = {
            "Authorization": f"Bearer {HUBSPOT_TOKEN}",
            "Content-Type": "application/json",
        }
        
        payload = {"properties": ticket_props}

        response = requests.post(
            HUBSPOT_TICKET_ENDPOINT,
            headers=headers,
            json=payload,
            timeout=10
        )
        
        response.raise_for_status()
        ticket_data = response.json()
        ticket_id = ticket_data.get("id")
        
        logger.info(f"HubSpot ticket created successfully: {ticket_id}")
        return {
            "status": "success",
            "message": "Ticket created successfully",
            "ticket_id": ticket_id,
            "data": ticket_data
        }
    except requests.exceptions.RequestException as e:
        error_msg = f"Failed to create HubSpot ticket: {str(e)}"
        logger.error(error_msg)
        if hasattr(e, 'response') and e.response is not None:
            try:
                error_details = e.response.json()
                logger.error(f"HubSpot API error details: {error_details}")
                error_msg = error_details.get('message', error_msg)
            except ValueError:
                error_msg = f"{error_msg} - {e.response.text}"
        raise HTTPException(
            status_code=getattr(e.response, 'status_code', 500) if hasattr(e, 'response') else 500,
            detail=error_msg
        )
    except Exception as e:
        error_msg = f"Unexpected error creating HubSpot ticket: {str(e)}"
        logger.error(error_msg, exc_info=True)
        raise HTTPException(status_code=500, detail=error_msg)
```

This is the core function responsible for interacting with the HubSpot API to create a ticket. It is a synchronous function, meaning it will block the calling thread until the HTTP request completes.

*   **Function Signature**:
    *   `title: str`: The subject line for the HubSpot ticket.
    *   `description: str`: The main content or body of the ticket.
    *   `properties: Optional[Dict[str, Any]] = None`: An optional dictionary of additional HubSpot ticket properties (e.g., `hs_priority`, `hs_ticket_priority`). If provided, these will be merged with the default properties.
    *   `pipeline: str = "0"`: The ID of the HubSpot pipeline to which the ticket should be assigned. `"0"` typically represents the default "Support" pipeline.
    *   `pipeline_stage: str = "1"`: The ID of the pipeline stage within the specified pipeline. `"1"` typically represents the "New" stage.
    *   `-> Dict[str, Any]`: The function is type-hinted to return a dictionary containing information about the operation's status and the created ticket.

*   **Implementation Logic**:
    1.  **Token Check**:
        *   `if not HUBSPOT_TOKEN:`: Checks if the `HUBSPOT_TOKEN` environment variable was successfully loaded.
        *   `error_msg = "HubSpot API token not configured"`: Defines an error message.
        *   `logger.error(error_msg)`: Logs the error using the configured logger.
        *   `raise HTTPException(status_code=500, detail=error_msg)`: Raises a FastAPI `HTTPException` with a 500 status code, indicating a server configuration error.
    2.  **API Call Block (`try...except`)**: All API interaction is wrapped in a `try...except` block to handle potential network or API errors gracefully.
    3.  **Construct `ticket_props`**:
        *   `ticket_props = {...}`: A dictionary `ticket_props` is initialized with essential HubSpot ticket properties: `subject`, `content`, `hs_pipeline`, and `hs_pipeline_stage`. These keys correspond to standard HubSpot ticket fields.
        *   `if properties: ticket_props.update(properties)`: If additional `properties` are provided, they are merged into `ticket_props`, allowing for customization.
    4.  **Prepare Headers**:
        *   `headers = {...}`: A dictionary `headers` is created for the HTTP request.
        *   `"Authorization": f"Bearer {HUBSPOT_TOKEN}"`: Sets the `Authorization` header with a Bearer token, using the retrieved `HUBSPOT_TOKEN` for authentication.
        *   `"Content-Type": "application/json"`: Specifies that the request body will be in JSON format.
    5.  **Prepare Payload**:
        *   `payload = {"properties": ticket_props}`: The final JSON payload for the HubSpot API request is constructed. HubSpot's CRM API expects a top-level `properties` key containing the ticket fields.
    6.  **Make API Request**:
        *   `response = requests.post(...)`: Performs an HTTP POST request to the `HUBSPOT_TICKET_ENDPOINT`.
            *   `headers=headers`: Passes the defined HTTP headers.
            *   `json=payload`: Automatically serializes the `payload` dictionary into JSON and sets it as the request body.
            *   `timeout=10`: Sets a timeout of 10 seconds for the request. If the server doesn't respond within this time, a `requests.exceptions.Timeout` exception is raised.
    7.  **Handle Response**:
        *   `response.raise_for_status()`: Checks if the HTTP response status code indicates an error (e.g., 4xx or 5xx). If it does, a `requests.exceptions.HTTPError` is raised.
        *   `ticket_data = response.json()`: Parses the JSON response body into a Python dictionary, which contains details of the newly created ticket.
        *   `ticket_id = ticket_data.get("id")`: Extracts the `id` of the created ticket from the response data.
        *   `logger.info(...)`: Logs a success message with the created ticket ID.
        *   `return {...}`: Returns a dictionary indicating success, a message, the `ticket_id`, and the full `data` received from HubSpot.
    8.  **Error Handling (`except` blocks)**:
        *   `except requests.exceptions.RequestException as e:`: Catches any exception related to the `requests` library (e.g., network errors, timeouts, HTTP errors from `raise_for_status`).
            *   Logs a generic error message.
            *   Attempts to extract more specific error details from the HubSpot API response if available (e.g., `e.response.json()`). This helps provide richer error messages to the client.
            *   `raise HTTPException(...)`: Raises an `HTTPException` with the appropriate status code (extracted from `e.response` if available, otherwise defaults to 500) and the refined error message.
        *   `except Exception as e:`: Catches any other unexpected errors during the function execution.
            *   Logs a generic error message, including stack trace (`exc_info=True`) for better debugging.
            *   `raise HTTPException(status_code=500, detail=error_msg)`: Raises a generic 500 `HTTPException`.

### 4.2. `create_ticket`

```python
async def create_ticket(
    title: str, 
    description: str, 
    properties: Optional[Dict[str, Any]] = None,
    pipeline: str = "0",
    pipeline_stage: str = "1",
    background_tasks: Optional[BackgroundTasks] = None
) -> Dict[str, Any]:
    """Create a ticket in HubSpot in a non-blocking way.
    
    Args:
        title: The subject of the ticket
        description: The content/description of the ticket
        properties: Additional ticket properties
        pipeline: HubSpot pipeline ID (default: "0" for Support pipeline)
        pipeline_stage: Pipeline stage ID (default: "1" for 'New' stage)
        background_tasks: FastAPI BackgroundTasks instance for background processing
        
    Returns:
        Dictionary with status and message
    """
    if background_tasks:
        # If BackgroundTasks is provided, use it to create the ticket in the background
        background_tasks.add_task(
            _create_ticket_sync, 
            title, 
            description, 
            properties,
            pipeline,
            pipeline_stage
        )
        return {
            "status": "queued", 
            "message": "Ticket creation queued"
        }
    
    # For backward compatibility or when BackgroundTasks is not available
    asyncio.create_task(
        _create_ticket_async(
            title, 
            description, 
            properties,
            pipeline,
            pipeline_stage
        )
    )
    return {"status": "queued", "message": "Ticket creation queued"}
```

This asynchronous function serves as the primary entry point for creating tickets, allowing for non-blocking execution. It intelligently uses `BackgroundTasks` if available (e.g., when called from a FastAPI endpoint) or falls back to `asyncio.create_task` for other async contexts.

*   **Function Signature**:
    *   Parameters are largely the same as `_create_ticket_sync`.
    *   `background_tasks: Optional[BackgroundTasks] = None`: An optional `BackgroundTasks` instance. This parameter is typically injected by FastAPI when the function is used as an endpoint dependency.
    *   `-> Dict[str, Any]`: Returns a dictionary indicating the queuing status of the ticket creation.

*   **Implementation Logic**:
    1.  **Check for `background_tasks`**:
        *   `if background_tasks:`: Determines if a `BackgroundTasks` instance has been provided. This is the preferred method when integrating with FastAPI, as it allows the HTTP response to be sent immediately while the ticket creation happens in the background.
        *   `background_tasks.add_task(...)`: If `background_tasks` is present, it adds `_create_ticket_sync` (the synchronous API call function) to the list of tasks to be executed in the background. The arguments for `_create_ticket_sync` are passed directly.
        *   `return {"status": "queued", "message": "Ticket creation queued"}`: Returns an immediate response indicating that the ticket creation has been queued, without waiting for the actual API call to complete.
    2.  **Fallback to `asyncio.create_task`**:
        *   `asyncio.create_task(...)`: If `background_tasks` is *not* provided (e.g., if this function is called directly in an `async` context outside of a FastAPI request cycle), this line creates a new task in the current event loop. This task will execute the `_create_ticket_async` coroutine concurrently. This is useful for non-FastAPI async applications or for backward compatibility.
        *   `return {"status": "queued", "message": "Ticket creation queued"}`: Similar to the `BackgroundTasks` case, it returns immediately, indicating the task has been launched asynchronously. The caller will not receive the result of the actual ticket creation.

### 4.3. `_create_ticket_async`

```python
async def _create_ticket_async(
    title: str, 
    description: str, 
    properties: Optional[Dict[str, Any]] = None,
    pipeline: str = "0",
    pipeline_stage: str = "1"
) -> Dict[str, Any]:
    """Async wrapper for _create_ticket_sync to use with asyncio.create_task.
    
    Args:
        title: The subject of the ticket
        description: The content/description of the ticket
        properties: Additional ticket properties
        pipeline: HubSpot pipeline ID
        pipeline_stage: Pipeline stage ID
        
    Returns:
        Dictionary containing status, message, and ticket details
    """
    loop = asyncio.get_event_loop()
    return await loop.run_in_executor(
        None, 
        _create_ticket_sync, 
        title, 
        description, 
        properties,
        pipeline,
        pipeline_stage
    )
```

This private asynchronous helper function is designed to run the synchronous `_create_ticket_sync` function in a separate thread, thereby preventing it from blocking the `asyncio` event loop. It is primarily used when `BackgroundTasks` is not available.

*   **Function Signature**: Identical parameters and return type to `_create_ticket_sync`.
*   **Implementation Logic**:
    1.  `loop = asyncio.get_event_loop()`: Retrieves the current running `asyncio` event loop.
    2.  `return await loop.run_in_executor(...)`:
        *   `run_in_executor` is a crucial `asyncio` method that allows running blocking (synchronous) functions in a separate thread or process pool without blocking the event loop.
        *   `None`: The first argument is the executor to use. `None` specifies the default `ThreadPoolExecutor`, which runs the function in a separate thread.
        *   `_create_ticket_sync`: The synchronous function to be executed.
        *   `title, description, properties, pipeline, pipeline_stage`: The arguments to be passed to `_create_ticket_sync`.
        *   `await`: The `await` keyword pauses the execution of `_create_ticket_async` until `_create_ticket_sync` completes in its separate thread, and then returns its result. However, the event loop itself remains unblocked, free to process other tasks.

### 4.4. `create_issue_report_ticket`

```python
async def create_issue_report_ticket(username: str, issue: str, search_query: str, aql_output: str, 
                                    sql_query: str, inference_time: float, positions: str, 
                                    classes: str, teams: str, background_tasks: BackgroundTasks = None) -> Dict[str, Any]:
    """Create a ticket for an issue report with all relevant information."""
    title = f"Issue Report from {username}"
    
    description = f"""
    Issue reported by '{username}':
    
    {issue}
    
    Search Query: {search_query}
    AQL Output: {aql_output}
    SQL Query: {sql_query}
    Inference Time: {inference_time}
    Positions: {positions}
    Classes: {classes}
    Teams: {teams}
    """
    
    # No custom properties; use default pipeline and stage
    return await create_ticket(
        title,
        description,
        background_tasks=background_tasks
    )
```

This is a convenience function that wraps the generic `create_ticket` function to specifically handle the creation of "issue report" tickets. It takes various pieces of diagnostic information and formats them into a HubSpot ticket's title and description.

*   **Function Signature**:
    *   `username: str`: The name of the user reporting the issue.
    *   `issue: str`: A detailed description of the reported problem.
    *   `search_query: str`: The search query related to the issue.
    *   `aql_output: str`: Output from an AQL (Analytic Query Language) query.
    *   `sql_query: str`: The SQL query related to the issue.
    *   `inference_time: float`: The time taken for an inference process.
    *   `positions: str`: Information about positions.
    *   `classes: str`: Information about classes.
    *   `teams: str`: Information about teams involved.
    *   `background_tasks: BackgroundTasks = None`: Optional `BackgroundTasks` for non-blocking execution, passed directly to `create_ticket`.
    *   `-> Dict[str, Any]`: Returns the result from the underlying `create_ticket` call.

*   **Implementation Logic**:
    1.  `title = f"Issue Report from {username}"`: Constructs a descriptive ticket title using the provided `username`.
    2.  `description = f"""..."""`: Formats a multi-line string for the ticket description. It cleanly incorporates all the detailed diagnostic parameters (`issue`, `search_query`, `aql_output`, etc.) into a readable structure.
    3.  `return await create_ticket(...)`: Calls the higher-level `create_ticket` function with the constructed `title` and `description`. It passes the `background_tasks` argument along, ensuring that the issue report ticket creation also benefits from background processing if available. It does not provide any `properties` for this specific ticket type, relying on the defaults defined in `create_ticket`.

## 5. Main Execution Flow (`__main__` block)

The provided script does not contain a `if __name__ == "__main__":` block, which means it is designed to be imported as a module into other applications rather than executed directly as a standalone script.

When imported, the following actions occur:
1.  All necessary modules are imported.
2.  Logging is configured.
3.  Environment variables are loaded from `.env`.
4.  HubSpot API constants (`HUBSPOT_TOKEN`, `HUBSPOT_TICKET_ENDPOINT`) are defined based on environment variables.
5.  All functions (`_create_ticket_sync`, `create_ticket`, `_create_ticket_async`, `create_issue_report_ticket`) are defined and made available for use by the importing application.

Typically, a FastAPI application might import this module and define an API endpoint that utilizes `create_ticket` or `create_issue_report_ticket`, often injecting `BackgroundTasks` via FastAPI's dependency injection system.

## 6. Important Implementation Details and Dependencies

*   **Asynchronous vs. Synchronous**: The script carefully separates synchronous (`_create_ticket_sync`) from asynchronous (`create_ticket`, `_create_ticket_async`) operations. `_create_ticket_sync` performs the actual blocking network call. `_create_ticket_async` is an `asyncio` specific wrapper that uses `run_in_executor` to offload the synchronous call to a thread pool, preventing it from blocking the `asyncio` event loop. `create_ticket` acts as a facade, deciding whether to use FastAPI's `BackgroundTasks` (preferred for FastAPI endpoints) or the generic `asyncio.create_task` with `_create_ticket_async`.
*   **`requests` Library**: Used for its simplicity and robustness in making HTTP requests.
*   **`dotenv` Library**: Essential for secure management of API keys and other sensitive configuration by loading them from a `.env` file, keeping them out of source control.
*   **FastAPI Integration**: The `BackgroundTasks` and `HTTPException` imports indicate a strong intent for this module to be used within a FastAPI application context. `HTTPException` allows API-specific errors to be raised and automatically converted into appropriate HTTP responses by FastAPI.
*   **Error Handling**: Comprehensive error handling is implemented, particularly within `_create_ticket_sync`, to catch network issues, API errors, and unexpected exceptions, providing informative log messages and raising `HTTPException` for client-facing error reporting.
*   **Type Hinting**: The extensive use of type hints (`typing` module) improves code clarity, enables static analysis tools to catch potential errors, and enhances developer experience.
*   **Pipelines and Stages**: HubSpot tickets are created with default pipeline ID "0" (Support) and stage ID "1" (New), but these are configurable through function parameters.

## 7. Configuration Variables

*   **`HUBSPOT_TOKEN`**: This is a critical environment variable that must be set for the script to function correctly. It holds the authentication token required to interact with the HubSpot API.
    *   **How to set**: It should be defined in a `.env` file in the root directory of the project, for example:
        ```
        HUBSPOT_TOKEN="YOUR_HUBSPOT_API_KEY_OR_ACCESS_TOKEN"
        ```
    *   Alternatively, it can be set directly in the environment where the application is run.
*   **`HUBSPOT_TICKET_ENDPOINT`**: A hardcoded constant pointing to the HubSpot CRM tickets API endpoint. This typically does not need to be changed unless HubSpot's API URL structure changes or a different API version is targeted.
