# Documentation for `player_dashboard.py`

This document provides a comprehensive technical overview of the provided Python script, which defines a set of FastAPI endpoints for retrieving various player-related statistics and information from a database.

---

## Technical Documentation: Player Dashboard API Endpoints

### 1. High-Level Overview

This Python script implements a collection of FastAPI endpoints designed to serve as a backend for a player dashboard. It leverages FastAPI's capabilities for defining RESTful APIs, Pydantic for data validation and serialization (via `app.models`), and a custom database adapter (`PlayerDashboardAdapter`) to interact with a database. The primary purpose is to expose endpoints for fetching player profiles, game logs, records, highs, and streaks data based on specific request parameters. Authentication is hinted at (though `verify_token` is imported but not directly used in these routes), and robust error handling is implemented.

### 2. Dependencies and Imports

The script begins by importing necessary modules and components:

*   `from datetime import date`: Imports the `date` class from the `datetime` module, used for handling date objects, particularly for setting default `start_date` and `end_date` values in the `/player-streaks` endpoint.
*   `from fastapi import APIRouter, HTTPException, Depends, Query`:
    *   `APIRouter`: Used to organize API routes into modular components, making the application scalable and maintainable.
    *   `HTTPException`: FastAPI's class for raising HTTP-specific errors, allowing custom status codes and detail messages.
    *   `Depends`: A FastAPI dependency injection utility, used here to inject the `PlayerDashboardAdapter` instance into endpoint functions.
    *   `Query`: Used with FastAPI to declare query parameters, allowing for additional validation and metadata.
*   `from app.models.player_dashboard_models import ...`: Imports various Pydantic models (data schemas) used for request bodies and response structures:
    *   `PlayerProfileRequest`, `PlayerProfileResponse`: For player profile data.
    *   `PlayerRecordsRequest`, `PlayerRecordsResponse`: For player records data.
    *   `PlayerHighsRequest`, `PlayerHighsResponse`: For player high statistics.
    *   `PlayerGameLogsRequest`, `PlayerGameLogsResponse`: For player game-by-game logs.
    *   `PlayerStreaksResponse`, `PlayerStreaksRequest`: For player streaks data.
    *   `MappingRequest`: Imported but not directly used in the provided script.
*   `from app.api.routes.auth import verify_token`: Imports a `verify_token` dependency, likely intended for authentication. While imported, it is not explicitly used as a `Depends` in the provided routes, suggesting authentication might be applied at a higher level (e.g., a global dependency) or is a placeholder.
*   `from app.db.playerDashboard import PlayerDashboardAdapter`: Imports the `PlayerDashboardAdapter` class, which encapsulates the logic for interacting with the database to retrieve player dashboard data. This promotes separation of concerns.
*   `from app.db.dependencies import get_player_dashboard_adapter`: Imports a dependency injector function that provides an instance of `PlayerDashboardAdapter`. This is crucial for FastAPI's dependency injection system.
*   `import logging`: Imports Python's standard logging library for recording events and errors.

### 3. Global and Module-Level Configurations

*   **`router = APIRouter()`**:
    This line initializes an `APIRouter` instance. This router will be used to define all the API endpoints within this module. It allows for modular organization of API routes, which can then be included into a main FastAPI application instance.

*   **`logging.basicConfig(level=logging.INFO)`**:
    Configures the root logger. It sets the logging level to `INFO`, meaning that messages with severity `INFO`, `WARNING`, `ERROR`, and `CRITICAL` will be processed. By default, these messages will be printed to the console (standard output).

*   **`logger = logging.getLogger(__name__)`**:
    Creates a logger instance specifically for this module, identified by its name (`__name__`). This allows for more granular control over logging behavior specific to this file.

*   **`# Adapters now use centralized connection pool by default`**:
    This comment indicates a crucial implementation detail: the `PlayerDashboardAdapter` (and likely other adapters in the system) are configured to use a centralized database connection pool. This improves database performance and resource management by reusing connections instead of opening and closing new ones for every request.

### 4. API Endpoints

The script defines several POST and GET endpoints using the `router` instance. Each endpoint handles a specific request type for player data.

#### 4.1. `POST /player_profile`

**`@router.post("/player_profile", response_model=PlayerProfileResponse)`**
This decorator registers the `get_player_profile` function as an HTTP POST endpoint at the path `/player_profile`. It also specifies `PlayerProfileResponse` as the expected response model, which FastAPI uses for automatic data serialization and OpenAPI documentation.

**`async def get_player_profile(request: PlayerProfileRequest, adapter: PlayerDashboardAdapter = Depends(get_player_dashboard_adapter))`**
*   **`async def get_player_profile(...)`**: Declares an asynchronous function, suitable for I/O-bound operations like database calls, allowing the server to handle other requests concurrently.
*   **`request: PlayerProfileRequest`**: Defines `request` as a parameter that expects a Pydantic model `PlayerProfileRequest`. FastAPI will automatically parse the request body into an instance of this model, validating the input data against its schema.
*   **`adapter: PlayerDashboardAdapter = Depends(get_player_dashboard_adapter)`**: This uses FastAPI's dependency injection. `get_player_dashboard_adapter` is a function that provides an instance of `PlayerDashboardAdapter`. This adapter instance is then injected into the `adapter` parameter, allowing the endpoint to interact with the database.

**Docstring Explanation:**
*   **Primary Purpose**: Endpoint for retrieving comprehensive player profile information.
*   **Logic**: Fetches detailed player profile data from the database based on `player ID` and `sport code`.
*   **Args**:
    *   `request (PlayerProfileRequest)`: Contains `player ID` and `sport code`.
    *   `username (str)`: Authenticated username from JWT token (commented out, indicating it might have been used previously or is for future implementation of per-user logging).
*   **Returns**: `PlayerProfileResponse`: Contains the player's profile information.
*   **Raises**: `HTTPException`: For various error conditions, specifically a `500 Internal Server Error` in case of unexpected exceptions.

**Implementation Logic:**
1.  **`try...except` block**: Encapsulates the core logic for error handling.
2.  **`logger.info(f"Received player profile request: {request.model_dump()}")`**: Logs the incoming request details, converting the Pydantic request model to a dictionary for logging purposes. The commented-out `username` logging suggests a prior or future intention for user-specific request tracking.
3.  **`player_data = adapter.execute_player_profile(...)`**: Calls the `execute_player_profile` method on the injected `adapter` instance. This method is responsible for querying the database. It passes `sport_code`, `entity` (hardcoded as "Player"), and `player_id` from the `request` object.
4.  **`return PlayerProfileResponse(**player_data)`**: If the adapter call is successful, the returned `player_data` (expected to be a dictionary) is unpacked and used to construct a `PlayerProfileResponse` object, which is then returned. FastAPI automatically serializes this Pydantic model into JSON.
5.  **`except Exception as e:`**: Catches any unexpected exceptions that occur during the process.
6.  **`logger.error(...)`**: Logs the error details, including the exception traceback (`exc_info=True`), for debugging.
7.  **`raise HTTPException(status_code=500, detail=f"Server error: {str(e)}")`**: Raises an `HTTPException` with a 500 status code and a detailed error message, which FastAPI then converts into an appropriate HTTP error response.

#### 4.2. `POST /player-game-logs`

**`@router.post("/player-game-logs")`**
Registers the `get_player_game_logs` function as an HTTP POST endpoint at `/player-game-logs`. No `response_model` is explicitly specified here, implying the adapter's return value will be directly returned, or FastAPI will infer it from the Pydantic model returned.

**`async def get_player_game_logs(request: PlayerGameLogsRequest, adapter: PlayerDashboardAdapter = Depends(get_player_dashboard_adapter))`**
*   **`request: PlayerGameLogsRequest`**: Expects a `PlayerGameLogsRequest` Pydantic model in the request body.
*   **`adapter: PlayerDashboardAdapter = Depends(get_player_dashboard_adapter)`**: Injects the database adapter.

**Docstring Explanation:**
*   **Primary Purpose**: Endpoint for retrieving player game logs.
*   **Logic**: Fetches detailed player game logs from the database based on `player ID` and `sport code`.
*   **Args**:
    *   `request (PlayerGameLogsRequest)`: Contains `player ID` and `sport code`.
    *   `username (str)`: Authenticated username (commented out).
*   **Returns**: `PlayerGameLogsResponse`: Contains player game logs information.
*   **Raises**: `HTTPException`: For errors.

**Implementation Logic:**
1.  **`try...except` block**: Standard error handling.
2.  **`logger.info(f"Received player game logs request: {request.model_dump()}")`**: Logs the incoming request.
3.  **`player_data = adapter.execute_player_game_logs(...)`**: Calls the `execute_player_game_logs` method on the adapter, passing `sport_code`, `entity` ("Player"), `season`, and `player_id` from the request.
4.  **`return player_data`**: Returns the data directly from the adapter. It's implicitly expected that `player_data` will conform to `PlayerGameLogsResponse` or a similar serializable structure.
5.  **Error Handling**: Similar `except` block for logging and raising `HTTPException(500)`.

#### 4.3. `POST /player-records`

**`@router.post("/player-records", response_model=PlayerRecordsResponse)`**
Registers the `get_player_records` function as an HTTP POST endpoint at `/player-records` with `PlayerRecordsResponse` as the expected response model.

**`async def get_player_records(request: PlayerRecordsRequest, adapter: PlayerDashboardAdapter = Depends(get_player_dashboard_adapter))`**
*   **`request: PlayerRecordsRequest`**: Expects a `PlayerRecordsRequest` Pydantic model.
*   **`adapter: PlayerDashboardAdapter = Depends(get_player_dashboard_adapter)`**: Injects the database adapter.

**Docstring Explanation:**
*   **Primary Purpose**: Endpoint for retrieving player records.
*   **Logic**: Fetches detailed player records data from the database.
*   **Args**:
    *   `request (PlayerRecordsRequest)`: Contains `player ID`, `sport code`, `teamcode`, `stat_period`, `rank_threshold`.
    *   `username (str)`: Authenticated username (commented out).
*   **Returns**: `PlayerRecordsResponse`: Contains player records information.
*   **Raises**: `HTTPException`: For errors.

**Implementation Logic:**
1.  **`try...except` block**: Standard error handling.
2.  **`logger.info(f"Received player record request: {request.model_dump()}")`**: Logs the incoming request.
3.  **`player_data = adapter.execute_player_records(...)`**: Calls the `execute_player_records` method, passing all relevant parameters from the `request` object.
4.  **`return PlayerRecordsResponse(**player_data)`**: Returns the data wrapped in a `PlayerRecordsResponse` model.
5.  **Error Handling**: Similar `except` block for logging and raising `HTTPException(500)`.

#### 4.4. `POST /player-highs`

**`@router.post("/player-highs", response_model=PlayerHighsResponse)`**
Registers the `get_player_highs` function as an HTTP POST endpoint at `/player-highs` with `PlayerHighsResponse` as the expected response model.

**`async def get_player_highs(request: PlayerHighsRequest, adapter: PlayerDashboardAdapter = Depends(get_player_dashboard_adapter))`**
*   **`request: PlayerHighsRequest`**: Expects a `PlayerHighsRequest` Pydantic model.
*   **`adapter: PlayerDashboardAdapter = Depends(get_player_dashboard_adapter)`**: Injects the database adapter.

**Docstring Explanation:**
*   **Primary Purpose**: Endpoint for retrieving player high statistics. (Note: Docstring incorrectly states "player records" in the description and returns section, it should be "player highs").
*   **Logic**: Fetches detailed player highs data from the database.
*   **Args**:
    *   `request (PlayerHighsRequest)`: Contains `player ID`, `sport code`, `stat_period`.
    *   `username (str)`: Authenticated username (commented out).
*   **Returns**: `PlayerHighsResponse`: Contains player high statistics information.
*   **Raises**: `HTTPException`: For errors.

**Implementation Logic:**
1.  **`try...except` block**: Standard error handling.
2.  **`logger.info(f"Received player record request: {request.model_dump()}")`**: Logs the incoming request (note: logs "player record request", should ideally be "player highs request").
3.  **`player_highs = adapter.execute_player_highs(...)`**: Calls the `execute_player_highs` method, passing `sport_code`, `entity` ("Player"), `player_id`, and `stat_period` from the request.
4.  **`return PlayerHighsResponse(**player_highs)`**: Returns the data wrapped in a `PlayerHighsResponse` model.
5.  **Error Handling**: Similar `except` block for logging and raising `HTTPException(500)`.

#### 4.5. `GET /player-streaks-mapping`

**`@router.get('/player-streaks-mapping')`**
Registers the `get_player_streaks_category` function as an HTTP GET endpoint at `/player-streaks-mapping`.

**`async def get_player_streaks_category( ... )`**
*   **`sport_code: str = Query(..., alias="SportCode", description="The sport code")`**: Defines `sport_code` as a required query parameter. `Query(...)` makes it required, `alias="SportCode"` maps the URL parameter `SportCode` to the `sport_code` Python variable, and `description` is for OpenAPI documentation.
*   **`playerposition: str = Query("ALL", alias="playerposition", description="The player position (defaults to ALL if not provided)")`**: Defines `playerposition` as an optional query parameter with a default value of "ALL".
*   **`adapter: PlayerDashboardAdapter = Depends(get_player_dashboard_adapter)`**: Injects the database adapter.

**Docstring Explanation:**
*   **Primary Purpose**: Retrieves player streaks category mapping based on sport code and player position.

**Implementation Logic:**
1.  **`try...except` block**: Standard error handling.
2.  **`player_streaks_category = adapter.fetch_streaks_mapping(...)`**: Calls the `fetch_streaks_mapping` method on the adapter, passing `sport_code` and `entity` ("Player"). This method is expected to return a dictionary-like structure containing streak mappings, potentially nested by position.
3.  **`if sport_code in ['MBB', 'WBB']:`**:
    *   **`playerposition = 'ALL'`**: For specific sport codes like 'MBB' (Men's Basketball) and 'WBB' (Women's Basketball), the `playerposition` is explicitly set to 'ALL', overriding any user-provided value. This suggests that for these sports, streaks mapping is only available or meaningful at an aggregated 'ALL' position level.
4.  **`return player_streaks_category[playerposition]`**: Accesses the `player_streaks_category` dictionary using the (potentially adjusted) `playerposition` as a key and returns the corresponding mapping.
5.  **`except KeyError as ke:`**: Catches `KeyError` specifically if the `playerposition` is not found in the returned `player_streaks_category` dictionary.
    *   **`logger.error(...)`**: Logs the specific `KeyError`.
    *   **`raise HTTPException(status_code=404, detail=f"Player position '{playerposition}' not found")`**: Raises a `404 Not Found` error if the position is not mapped.
6.  **`except Exception as e:`**: Catches any other unexpected exceptions.
    *   **`logger.error(...)`**: Logs the general API error.
    *   **`raise HTTPException(status_code=500, detail=str(e))`**: Raises a `500 Internal Server Error`.

#### 4.6. `POST /player-streaks`

**`@router.post("/player-streaks", response_model=PlayerStreaksResponse)`**
Registers the `get_player_streaks` function as an HTTP POST endpoint at `/player-streaks` with `PlayerStreaksResponse` as the expected response model.

**`async def get_player_streaks(request: PlayerStreaksRequest, adapter: PlayerDashboardAdapter = Depends(get_player_dashboard_adapter))`**
*   **`request: PlayerStreaksRequest`**: Expects a `PlayerStreaksRequest` Pydantic model.
*   **`adapter: PlayerDashboardAdapter = Depends(get_player_dashboard_adapter)`**: Injects the database adapter.

**Implementation Logic:**
1.  **`try...except` block**: Standard error handling.
2.  **`logger.info(f"Received player record request: {request.model_dump()}")`**: Logs the incoming request (note: logs "player record request", should ideally be "player streaks request").
3.  **Default Date Handling**:
    *   **`if request.start_date is None:`**: If `start_date` is not provided in the request, it defaults to `date(1950, 1, 1)`.
    *   **`request.start_date = date(1950, 1, 1).strftime("%Y-%m-%d")`**: The default date is formatted as "YYYY-MM-DD". This effectively sets a very old date as the beginning of the search period if none is specified.
    *   **`if request.end_date is None:`**: If `end_date` is not provided, it defaults to the current date.
    *   **`request.end_date = date.today().strftime("%Y-%m-%d")`**: The current date is formatted as "YYYY-MM-DD".
4.  **`player_streaks = adapter.get_player_streaks(...)`**: Calls the `get_player_streaks` method on the adapter, passing all streak-specific parameters from the (potentially modified) `request` object.
5.  **`return PlayerStreaksResponse(**player_streaks)`**: Returns the data wrapped in a `PlayerStreaksResponse` model.
6.  **Error Handling**: Similar `except` block for logging and raising `HTTPException(500)`.

### 5. Implementation Details and Considerations

*   **Pydantic Models**: The extensive use of Pydantic models (e.g., `PlayerProfileRequest`, `PlayerProfileResponse`) ensures strong type hinting, automatic request body parsing, input validation, and clear API documentation through OpenAPI.
*   **Dependency Injection**: FastAPI's `Depends` system is central to how the `PlayerDashboardAdapter` is provided to each endpoint. This promotes testability, reusability, and modularity. The comment "Adapter is automatically injected" reinforces this.
*   **Database Adapter (`PlayerDashboardAdapter`)**: This adapter acts as an abstraction layer for database interactions. Its methods (`execute_player_profile`, `execute_player_game_logs`, etc.) are responsible for executing specific queries and retrieving data. The comment "Connection is automatically returned to pool" highlights its interaction with a connection pooling mechanism, which is crucial for performance and scalability in database-intensive applications.
*   **Error Handling**: A consistent `try-except` block pattern is used across all endpoints to catch general exceptions. When an error occurs, it's logged using `logger.error` (with `exc_info=True` for traceback) and then re-raised as an `HTTPException` with a `500 Internal Server Error` status code, providing a user-friendly error message. A specific `KeyError` is handled for `get_player_streaks_category` to return a `404 Not Found`.
*   **Logging**: `logging.basicConfig` and `logger` instances are used to provide visibility into the application's runtime behavior, aiding in debugging and monitoring.
*   **Authentication (Implied)**: The import of `verify_token` suggests that authentication is handled elsewhere in the application, possibly at a higher level that applies to all routes or via a global dependency for the `APIRouter`.

### 6. Main Execution Flow (`__main__` block)

The provided script defines API routes and associated logic. It does not contain a `if __name__ == "__main__":` block, which means it's intended to be imported and included into a larger FastAPI application (e.g., in a main `app.py` file) using `app.include_router(router)`. The server itself would be started externally, typically using a ASGI server like Uvicorn.
