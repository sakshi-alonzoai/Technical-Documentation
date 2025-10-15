# Documentation for `game_dashboard.py`

This document provides a comprehensive technical overview and line-by-line explanation of the provided Python script, which defines a set of FastAPI endpoints for a game dashboard application.

---

## 1. High-Level Overview

This Python script (`game_dashboard_routes.py`) serves as a module within a larger FastAPI application, responsible for defining API endpoints related to game dashboard functionality. Its primary purpose is to expose various game-centric data, such as general game details, player statistics, team statistics, and rosters, through a RESTful interface.

The script leverages FastAPI's capabilities for request validation, dependency injection, and automatic documentation. It interacts with a `GameDashboardAdapter` (injected via FastAPI's dependency system) to retrieve data from an underlying data source (e.g., a database or an external API). Robust error handling and logging are integrated into each endpoint to ensure reliability and traceability.

## 2. Dependencies and Imports

The script begins by importing necessary modules and classes:

*   **`from datetime import date`**:
    *   **Purpose**: Imports the `date` class from Python's standard `datetime` module.
    *   **Usage in Script**: This import is present but the `date` class is *not directly used* in the provided code snippet. It might be a remnant from previous development or intended for future use within the models or adapter logic.
*   **`from fastapi import APIRouter, HTTPException, Depends, Query`**:
    *   **Purpose**: Imports core components from the FastAPI framework.
    *   **`APIRouter`**: Used to create modular routes, allowing the organization of endpoints into separate files.
    *   **`HTTPException`**: Used to raise standard HTTP errors with specific status codes and detail messages.
    *   **`Depends`**: Used for dependency injection, allowing functions to declare dependencies that FastAPI will resolve.
    *   **`Query`**: Used to declare query parameters for GET requests, providing validation, documentation, and default values.
*   **`from app.models.game_dashboard_models import GameDashboardRequest, GameDashboardResponse, GamePlayerStatsResponse, GameTeamStatsResponse, GameRosters`**:
    *   **Purpose**: Imports Pydantic models that define the structure of request bodies and response data for the API endpoints.
    *   **`GameDashboardRequest`**: Defines the expected structure for the POST request body of the `game_dashboard` endpoint.
    *   **`GameDashboardResponse`**: Defines the expected structure for the response of the `game_dashboard` endpoint.
    *   **`GamePlayerStatsResponse`**: Defines the expected structure for the response of the `game_player_stats` endpoint.
    *   **`GameTeamStatsResponse`**: Defines the expected structure for the response of the `game_team_stats` endpoint.
    *   **`GameRosters`**: Defines the expected structure for the response of the `game_rosters` endpoint.
*   **`from app.api.routes.auth import verify_token`**:
    *   **Purpose**: Imports a `verify_token` function, likely intended for authentication.
    *   **Usage in Script**: This import is present but the `verify_token` function is *not used* in any of the provided endpoints. It suggests that authentication middleware could be integrated in the future.
*   **`from app.db.gameDashboard import GameDashboardAdapter`**:
    *   **Purpose**: Imports the `GameDashboardAdapter` class, which is responsible for abstracting the data access logic for game dashboard data. This adapter would typically interact with a database or another data source.
*   **`from app.db.dependencies import get_game_dashboard_adapter`**:
    *   **Purpose**: Imports a dependency function `get_game_dashboard_adapter`. This function is designed to be used with FastAPI's `Depends` to provide an instance of `GameDashboardAdapter` to the route handlers. This pattern ensures proper resource management (e.g., database connection pooling) and testability.
*   **`import logging`**:
    *   **Purpose**: Imports Python's standard `logging` module for event logging.

## 3. Global Configuration and Variables

*   **`logging.basicConfig(level=logging.INFO)`**:
    *   **Purpose**: Configures the basic logging settings for the application.
    *   **Details**: Sets the root logger's level to `INFO`, meaning messages with `INFO`, `WARNING`, `ERROR`, and `CRITICAL` severity will be processed and displayed (typically to the console by default).
*   **`logger = logging.getLogger(__name__)`**:
    *   **Purpose**: Creates a logger instance specifically for this module.
    *   **Details**: Using `__name__` ensures that log messages are attributed to this specific module, which is helpful for tracing the origin of logs in a larger application.
*   **`router = APIRouter()`**:
    *   **Purpose**: Initializes an `APIRouter` instance.
    *   **Details**: This `router` object is used to define all the API endpoints within this file. In a main FastAPI application file, this `router` would be included using `app.include_router(router)` to add these endpoints to the main application.

## 4. API Endpoints

This section details each API endpoint defined in the script.

---

### 4.1. `game_dashboard`

*   **Decorator**: `@router.post("/game-dashboard", response_model=GameDashboardResponse)`
    *   **`@router.post(...)`**: Declares this asynchronous function as an HTTP POST endpoint.
    *   **`"/game-dashboard"`**: The URL path for this endpoint.
    *   **`response_model=GameDashboardResponse`**: Specifies that the response body returned by this endpoint will be automatically validated and serialized according to the `GameDashboardResponse` Pydantic model. This also generates OpenAPI documentation for the response structure.
*   **Signature**: `async def game_dashboard(request: GameDashboardRequest, adapter: GameDashboardAdapter = Depends(get_game_dashboard_adapter))`
    *   **`request: GameDashboardRequest`**: This parameter expects the incoming request body to conform to the `GameDashboardRequest` Pydantic model. FastAPI automatically handles parsing and validating the JSON request body against this model.
    *   **`adapter: GameDashboardAdapter = Depends(get_game_dashboard_adapter)`**: This uses FastAPI's dependency injection. The `get_game_dashboard_adapter` function will be called to provide an instance of `GameDashboardAdapter`, which is then passed as the `adapter` argument to this route handler.
*   **Implementation Logic**:
    ```python
    try:
        logger.info("Received game dashboard request")
        # raise Exception("Simulated internal server error") # Commented out for debugging
        
        # Validate request data
        if not request.SportCode or not request.Match_id:
            raise HTTPException(status_code=400, detail="Invalid request data")
            
        # Fetch game dashboard data using injected adapter (connection handling is automatic)
        game_dashboard_data = adapter.game_dashboard(
            sport_code=request.SportCode,
            match_id = request.Match_id,
            home_team_code = request.HomeTeamCode
        )
        
        if not game_dashboard_data:
            raise HTTPException(status_code=404, detail="Game dashboard data not found")
        return GameDashboardResponse(**game_dashboard_data)
    except Exception as e:
        logger.error(f"Error fetching game dashboard data: {str(e)}")
        raise HTTPException(status_code=500, detail="Internal server error")
    ```
    1.  `logger.info("Received game dashboard request")`: Logs an informational message indicating that the endpoint has been hit.
    2.  **Request Validation**: `if not request.SportCode or not request.Match_id:`
        *   Checks if the `SportCode` and `Match_id` fields are present in the request body. While `GameDashboardRequest` Pydantic model would provide initial validation, this explicit check adds an immediate barrier for critical missing fields and provides a custom error message.
        *   If validation fails, it raises `HTTPException(status_code=400, detail="Invalid request data")`, returning a 400 Bad Request error to the client.
    3.  **Data Fetching**:
        *   `game_dashboard_data = adapter.game_dashboard(...)`: Calls the `game_dashboard` method of the injected `adapter` instance. It passes the `SportCode`, `Match_id`, and `HomeTeamCode` from the `request` object.
    4.  **Data Not Found Check**: `if not game_dashboard_data:`
        *   If the `adapter` returns `None` or an empty data structure, it indicates that no data was found for the given criteria.
        *   Raises `HTTPException(status_code=404, detail="Game dashboard data not found")`, returning a 404 Not Found error.
    5.  **Successful Response**: `return GameDashboardResponse(**game_dashboard_data)`:
        *   If data is found, it unpacks the `game_dashboard_data` dictionary into the `GameDashboardResponse` Pydantic model and returns it. FastAPI handles the automatic JSON serialization based on `response_model`.
    6.  **Error Handling**: `except Exception as e:`
        *   A `try...except` block catches any unexpected exceptions during the process.
        *   `logger.error(f"Error fetching game dashboard data: {str(e)}")`: Logs the specific error message.
        *   `raise HTTPException(status_code=500, detail="Internal server error")`: Returns a 500 Internal Server Error to the client, masking the internal exception details for security, while logging the full error on the server side.

---

### 4.2. `game_player_stats`

*   **Decorator**: `@router.get("/game-player-stats", response_model=GamePlayerStatsResponse)`
    *   **`@router.get(...)`**: Declares this asynchronous function as an HTTP GET endpoint.
    *   **`"/game-player-stats"`**: The URL path for this endpoint.
    *   **`response_model=GamePlayerStatsResponse`**: Specifies the response body will conform to `GamePlayerStatsResponse`.
*   **Signature**: `async def game_player_stats(sport_code: str = Query(..., alias="SportCode", description="The sport code"), ...)`
    *   **`sport_code: str = Query(..., alias="SportCode", description="The sport code")`**:
        *   **`Query(...)`**: Declares `sport_code` as a required query parameter.
        *   **`alias="SportCode"`**: Maps the Python parameter `sport_code` to the query parameter name `SportCode` in the URL (e.g., `?SportCode=MLB`).
        *   **`description="..."`**: Provides documentation for the parameter, visible in OpenAPI (Swagger UI).
    *   **`match_id: str = Query(..., alias="Match_id", description="The match id")`**: Required query parameter, aliased as `Match_id`.
    *   **`home_team_code: str = Query(..., alias="HomeTeamCode", description="The home team code")`**: Required query parameter, aliased as `HomeTeamCode`.
    *   **`category: str = Query("Passes", alias="category", description="The category (defaults to Passes if not provided)")`**: Optional query parameter.
        *   **`"Passes"`**: Provides a default value, meaning if `category` is not provided in the URL, it will default to "Passes".
        *   **`alias="category"`**: Maps the Python parameter `category` to the query parameter `category`.
    *   **`adapter: GameDashboardAdapter = Depends(get_game_dashboard_adapter)`**: Injected `GameDashboardAdapter` instance.
*   **Implementation Logic**:
    ```python
    try:
        logger.info("Received game player stats request")
        
        # Validate request data
        if not sport_code or not match_id:
            raise HTTPException(status_code=400, detail="Invalid request data")

        # Fetch game player stats data using the injected adapter
        game_player_stats_data = adapter.game_player_stats(
            sport_code=sport_code,
            match_id=match_id,
            home_team_code=home_team_code,
            category=category
        )
        if not game_player_stats_data:
            raise HTTPException(status_code=404, detail="Game player stats data not found")
        
        # Return response
        return GamePlayerStatsResponse(**game_player_stats_data)
    except Exception as e:
        logger.error(f"Error fetching game player stats data: {str(e)}")
        raise HTTPException(status_code=500, detail="Internal server error")
    ```
    1.  `logger.info("Received game player stats request")`: Logs an informational message.
    2.  **Request Validation**: `if not sport_code or not match_id:`
        *   Performs an explicit check for the presence of `sport_code` and `match_id`. FastAPI's `Query(...)` already marks them as required, but this adds an immediate, custom error.
        *   If validation fails, raises `HTTPException(status_code=400, detail="Invalid request data")`.
    3.  **Data Fetching**:
        *   `game_player_stats_data = adapter.game_player_stats(...)`: Calls the `game_player_stats` method of the `adapter`, passing all query parameters.
    4.  **Data Not Found Check**: `if not game_player_stats_data:`
        *   If the adapter returns no data, raises `HTTPException(status_code=404, detail="Game player stats data not found")`.
    5.  **Successful Response**: `return GamePlayerStatsResponse(**game_player_stats_data)`:
        *   Returns the fetched data unpacked into a `GamePlayerStatsResponse` model.
    6.  **Error Handling**: Catches exceptions, logs them, and raises `HTTPException(status_code=500, detail="Internal server error")`.

---

### 4.3. `game_team_stats`

*   **Decorator**: `@router.get("/game-team-stats")`
    *   **`@router.get(...)`**: Declares this asynchronous function as an HTTP GET endpoint.
    *   **`"/game-team-stats"`**: The URL path.
    *   **Note**: `response_model=GameTeamStatsResponse` is *not explicitly defined* in the decorator, but FastAPI will infer it from the return type `GameTeamStatsResponse(...)` in the function. It's generally good practice to explicitly state `response_model` for clarity and robust OpenAPI generation.
*   **Signature**: `async def game_team_stats(sport_code: str = Query(..., alias="SportCode", description="The sport code"), ...)`
    *   **`sport_code`, `match_id`, `home_team_code`**: Required query parameters, aliased similar to `game_player_stats`.
    *   **`adapter: GameDashboardAdapter = Depends(get_game_dashboard_adapter)`**: Injected `GameDashboardAdapter` instance.
*   **Implementation Logic**:
    ```python
    try:
        logger.info("Received game team stats request")
        # raise Exception("Simulated internal server error") # Commented out for debugging

        # Validate request data
        if not sport_code or not match_id:
            raise HTTPException(status_code=400, detail="Invalid request data")

        # Fetch game team stats data using injected adapter
        game_team_stats_data = adapter.game_team_stats(
            sport_code=sport_code,
            match_id=match_id,
            home_team_code=home_team_code
        )
        if not game_team_stats_data:
            raise HTTPException(status_code=404, detail="Game team stats data not found")
        
        return GameTeamStatsResponse(**game_team_stats_data)
    except Exception as e:
        logger.error(f"Error fetching game team stats data: {str(e)}")
        raise HTTPException(status_code=500, detail="Internal server error")
    ```
    1.  Logs an informational message.
    2.  **Request Validation**: Checks `sport_code` and `match_id`. If missing, raises `HTTPException(status_code=400)`.
    3.  **Data Fetching**: Calls `adapter.game_team_stats` with the query parameters.
    4.  **Data Not Found Check**: If no data is returned, raises `HTTPException(status_code=404)`.
    5.  **Successful Response**: Returns the fetched data unpacked into a `GameTeamStatsResponse` model.
    6.  **Error Handling**: Catches exceptions, logs them, and raises `HTTPException(status_code=500)`.

---

### 4.4. `game_rosters`

*   **Decorator**: `@router.get("/game-rosters")`
    *   **`@router.get(...)`**: Declares this asynchronous function as an HTTP GET endpoint.
    *   **`"/game-rosters"`**: The URL path.
    *   **Note**: Similar to `game_team_stats`, `response_model=GameRosters` is not explicitly defined but inferred from the return type `GameRosters(...)`.
*   **Signature**: `async def game_rosters(sport_code: str = Query(..., alias="SportCode", description="The sport code"), ...)`
    *   **`sport_code`, `match_id`, `home_team_code`**: Required query parameters, aliased as before.
    *   **`show_starters_only: bool = Query(False, alias="ShowStartersOnly", description="Whether to show only starters")`**: Optional boolean query parameter.
        *   **`False`**: Provides a default value. If `ShowStartersOnly` is not present in the URL, it defaults to `False`.
        *   **`alias="ShowStartersOnly"`**: Maps to the query parameter `ShowStartersOnly`.
    *   **`adapter: GameDashboardAdapter = Depends(get_game_dashboard_adapter)`**: Injected `GameDashboardAdapter` instance.
*   **Implementation Logic**:
    ```python
    try:
        logger.info("Received game rosters request")
        # raise Exception("Simulated internal server error") # Commented out for debugging
        
        # Validate request data
        if not sport_code or not match_id:
            raise HTTPException(status_code=400, detail="Invalid request data")
            print("not found game roster") # Debugging print statement

        # Validate request data (redundant check)
        if not sport_code or not match_id:
            raise HTTPException(status_code=400, detail="Invalid request data")

        # Fetch game rosters data using injected adapter
        game_rosters_data = adapter.game_rosters(
            sport_code=sport_code,
            match_id=match_id,
            home_team_code=home_team_code,
            show_starters_only=show_starters_only
        )
        print("game_rosters_data", game_rosters_data) # Debugging print statement
        if not game_rosters_data:
            raise HTTPException(status_code=404, detail="Game rosters data not found")
        
        return GameRosters(**game_rosters_data)
    except Exception as e:
        logger.error(f"Error fetching game rosters data: {str(e)}")
        logger.error(f"Received data structure: {game_rosters_data}") # Logs data structure during error
        raise HTTPException(status_code=500, detail="Internal server error")
    ```
    1.  Logs an informational message.
    2.  **Request Validation (Redundant)**: There are two consecutive identical `if not sport_code or not match_id:` checks. This is redundant and should be consolidated into one.
        *   If validation fails, raises `HTTPException(status_code=400)`.
        *   Includes a `print("not found game roster")` statement, which is a debugging leftover and should ideally be `logger.debug()` or removed in production.
    3.  **Data Fetching**: Calls `adapter.game_rosters` with all query parameters, including `show_starters_only`.
    4.  **Debugging Output**: `print("game_rosters_data", game_rosters_data)` is another debugging `print` statement, which should be replaced by a logger call.
    5.  **Data Not Found Check**: If no data is returned, raises `HTTPException(status_code=404)`.
    6.  **Successful Response**: Returns the fetched data unpacked into a `GameRosters` model.
    7.  **Error Handling**:
        *   Catches exceptions, logs them.
        *   `logger.error(f"Received data structure: {game_rosters_data}")`: This additional logging line captures the state of `game_rosters_data` during an error, which can be very useful for debugging issues where the data might be malformed or partially received before an error occurs.
        *   Raises `HTTPException(status_code=500)`.

## 5. Main Execution Flow

This script does **not** contain a traditional `if __name__ == "__main__":` block. This is characteristic of a module designed to be imported and used by a larger application. In a typical FastAPI setup, a main application file (e.g., `main.py` or `app.py`) would import this `router` and include it using `app.include_router(router)`. This modular approach promotes better organization of API endpoints.

## 6. Important Implementation Details

*   **FastAPI `APIRouter`**: Facilitates the modular organization of API endpoints, making the codebase scalable and maintainable.
*   **Dependency Injection (`Depends`)**: Key for managing resources (like database adapters) and promoting loose coupling between components. It makes testing easier as `GameDashboardAdapter` can be mocked.
*   **Pydantic Models**: Used for declarative request and response body validation and serialization, ensuring data consistency and providing automatic API documentation (OpenAPI/Swagger UI).
*   **Query Parameters (`Query`)**: Provides robust handling of URL query parameters, including type conversion, validation, default values, aliases, and documentation.
*   **Centralized Error Handling**: Consistent use of `try...except` blocks with `HTTPException` ensures that all API endpoints provide standardized error responses (e.g., 400 for bad input, 404 for not found, 500 for server errors), enhancing API usability and client predictability.
*   **Logging**: Basic logging is configured to provide visibility into the application's runtime behavior, crucial for monitoring, debugging, and auditing.
*   **Asynchronous Operations (`async def`)**: All route handlers are defined as `async def` functions, indicating that they are designed to run asynchronously, which is a core feature of FastAPI for handling concurrent requests efficiently.
*   **Code Quality Notes**:
    *   The redundant validation check in `game_rosters` should be removed.
    *   `print` statements used for debugging (e.g., `print("not found game roster")`, `print("game_rosters_data", ...)`) should ideally be replaced with `logger.debug()` or `logger.info()` calls to integrate with the standard logging system and allow for controlled output in different environments.
    *   The `datetime.date` import is unused.
    *   The `verify_token` import is unused.
