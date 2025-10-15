# Documentation for `main.py`

This document provides comprehensive technical documentation for the given Python script, detailing its purpose, structure, and functionality.

---

## Technical Documentation: R2 Search API Main Application

### 1. High-Level Overview

This Python script `main.py` serves as the entry point for a FastAPI-based RESTful API, specifically named "R2 Search API". Its primary purpose is to initialize the FastAPI application, configure core services like Cross-Origin Resource Sharing (CORS), load environment variables, and register various route handlers for different functionalities such as authentication, search, administration, reports, and dashboards. The API is designed to handle both public and protected endpoints, with protection implemented using JWT-based token verification and role-based (admin) authentication.

### 2. Implementation Details and Dependencies

The application relies on several key libraries and internal modules:

*   **FastAPI**: The web framework for building the API.
*   **python-dotenv**: For managing environment variables, ensuring sensitive information or configuration settings are not hardcoded.
*   **Uvicorn (implicit)**: While not directly imported here, FastAPI applications are typically served using an ASGI server like Uvicorn.
*   **Internal Modules**: The `app.api.routes` and `app.api.middleware` packages house the actual API endpoints (routers) and custom middleware/authentication logic, respectively. These modules are structured to promote modularity and separation of concerns.

### 3. Script Breakdown

#### 3.1. Standard Library Imports

```python
# Standard library imports
import os
```

*   `import os`: This line imports the `os` module from Python's standard library. The `os` module provides a way of using operating system dependent functionality, such as interacting with environment variables. While `dotenv` is used for loading, `os.environ` is typically used to *access* these variables after they are loaded.

#### 3.2. Third-Party Imports

```python
# Third-party imports
from dotenv import load_dotenv
from fastapi import FastAPI, Depends
from fastapi.middleware.cors import CORSMiddleware
```

*   `from dotenv import load_dotenv`: Imports the `load_dotenv` function from the `dotenv` library. This function is used to load key-value pairs from a `.env` file into the application's environment variables.
*   `from fastapi import FastAPI, Depends`:
    *   `FastAPI`: Imports the main `FastAPI` class, which is used to create the web application instance.
    *   `Depends`: Imports the `Depends` class from FastAPI, a powerful feature for dependency injection. It's used here to inject authentication logic into route definitions.
*   `from fastapi.middleware.cors import CORSMiddleware`: Imports the `CORSMiddleware` class from FastAPI. This middleware is crucial for handling Cross-Origin Resource Sharing (CORS) policies, allowing web browsers to access resources from origins other than their own.

#### 3.3. Environment Variable Loading

```python
# Load environment variables from .env file
load_dotenv()
```

*   `load_dotenv()`: This function call executes immediately upon script startup. It scans the current directory and its parents for a `.env` file and loads any key-value pairs found within it as environment variables. This is a common practice for managing configuration settings (e.g., database credentials, API keys) securely and separately from the codebase.

#### 3.4. Internal Imports - Route Handlers and Authentication

```python
# Internal imports - Route handlers and authentication
from app.api.routes import search, auth
from app.api.routes.auth import verify_token
from app.api.middleware.admin_auth import verify_admin
from app.api.routes.search import router as search_router, public_router as search_public_router
from app.api.routes.player_dashboard import router as player_dashboard_router
from app.api.routes.report import router as report_router
from app.api.routes.game_dashboard import router as game_dashboard_router
from app.api.routes.admin import router as admin_router
from app.api.routes.login_issue import router as login_issue_router
```

These lines import various components (routers and authentication functions) from the application's internal `app.api` package structure. Each import brings in a specific router or utility function:

*   `from app.api.routes import search, auth`: Imports the `search` and `auth` modules from `app.api.routes`. These modules likely contain additional functions or sub-routers.
*   `from app.api.routes.auth import verify_token`: Imports the `verify_token` function specifically from the `auth` module. This function is expected to handle the verification of user authentication tokens (e.g., JWTs) and is used as a dependency for protected routes.
*   `from app.api.middleware.admin_auth import verify_admin`: Imports the `verify_admin` function from the `admin_auth` middleware module. This function is specifically designed to verify if an authenticated user also holds an administrator role, making it suitable for protecting admin-specific endpoints.
*   `from app.api.routes.search import router as search_router, public_router as search_public_router`: Imports two distinct `APIRouter` instances from the `search` module:
    *   `search_router`: Expected to contain search endpoints that require authentication.
    *   `search_public_router`: Expected to contain search endpoints that do not require authentication.
*   `from app.api.routes.player_dashboard import router as player_dashboard_router`: Imports the `APIRouter` instance for player dashboard functionalities.
*   `from app.api.routes.report import router as report_router`: Imports the `APIRouter` instance for report generation and access.
*   `from app.api.routes.game_dashboard import router as game_dashboard_router`: Imports the `APIRouter` instance for game dashboard functionalities.
*   `from app.api.routes.admin import router as admin_router`: Imports the `APIRouter` instance dedicated to administrative tasks.
*   `from app.api.routes.login_issue import router as login_issue_router`: Imports the `APIRouter` instance for reporting login-related issues.

#### 3.5. FastAPI Application Initialization

```python
# Initialize FastAPI application with metadata
app = FastAPI(title="R2 Search API", version="1.0")
```

*   `app = FastAPI(...)`: This line creates the main FastAPI application instance.
    *   `title="R2 Search API"`: Sets the title of the API, which will be displayed in the automatically generated interactive API documentation (e.g., Swagger UI).
    *   `version="1.0"`: Sets the version number of the API, also used in the documentation.

#### 3.6. CORS Configuration

```python
# Configure CORS (Cross-Origin Resource Sharing) allowed origins
# This is necessary to allow frontend applications to communicate with the API
origins = [
    "http://localhost:3000",      # Local development React server
    "http://127.0.0.1:3000"       # Alternative local development URL
]

# Add CORS middleware to allow cross-origin requests
# This configuration allows:
# - Requests from specified origins
# - Credentials in requests
# - All HTTP methods
# - All HTTP headers
app.add_middleware(
    CORSMiddleware,
    allow_origins=origins,
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)
```

This block configures and adds the `CORSMiddleware` to the FastAPI application.

*   `origins = [...]`: Defines a list of allowed origins. These are the URLs of frontend applications (or any other client) that are permitted to make requests to this API. In this case, `http://localhost:3000` and `http://127.0.0.1:3000` are specified, typically used during local development for a React (or similar) frontend.
*   `app.add_middleware(CORSMiddleware, ...)`: Integrates the CORS middleware into the application's request processing pipeline.
    *   `allow_origins=origins`: Specifies the list of origins that are permitted to make cross-origin requests. Requests from any other origin will be blocked by the middleware.
    *   `allow_credentials=True`: Indicates that cookies, authorization headers, or TLS client certificates can be included in cross-origin requests.
    *   `allow_methods=["*"]`: Allows all standard HTTP methods (GET, POST, PUT, DELETE, OPTIONS, etc.) for cross-origin requests.
    *   `allow_headers=["*"]`: Allows all HTTP headers to be sent in cross-origin requests.

#### 3.7. Debug/Testing Print Statement

```python
print("Testing ...")
```

*   `print("Testing ...")`: This is a simple print statement. It serves as a basic indicator that the script has started execution. In a production environment, this might be replaced by a more sophisticated logging mechanism or removed entirely.

#### 3.8. Route Registration

```python
# Register route handlers with their respective prefixes and tags
# Auth routes - Handle user authentication and authorization
app.include_router(auth.router, prefix="/auth", tags=["Auth"])

# Protected search routes - Require valid authentication token
app.include_router(search_router, prefix="/api", tags=["Search"], dependencies=[Depends(verify_token)])

# Admin routes - Protected with admin role check
app.include_router(admin_router, prefix="/api", tags=["Admin"], dependencies=[Depends(verify_admin)])

# Report routes - No authentication required
app.include_router(report_router, prefix="/api", tags=["Report"])

# Player dashboard routes - Protected routes
app.include_router(player_dashboard_router, prefix="/api", tags=["Player Dashboard"])

# Game dashboard routes - Protected routes
app.include_router(game_dashboard_router, prefix="/api", tags=["Game Dashboard"])

# Public search routes - No authentication required
app.include_router(search_public_router, prefix="/api", tags=["Search"])

# Login page issue reporting - No authentication required
app.include_router(login_issue_router, prefix="/api", tags=["Login Issues"])
```

These lines are crucial for integrating all the defined `APIRouter` instances into the main FastAPI application. Each `app.include_router()` call registers a set of endpoints under a specific URL prefix and assigns them to categories (tags) for documentation purposes.

*   `app.include_router(auth.router, prefix="/auth", tags=["Auth"])`:
    *   Includes the router from the `auth` module.
    *   `prefix="/auth"`: All endpoints defined within `auth.router` will be accessible under the `/auth` path (e.g., `/auth/login`, `/auth/register`).
    *   `tags=["Auth"]`: Groups these endpoints under the "Auth" category in the API documentation.
*   `app.include_router(search_router, prefix="/api", tags=["Search"], dependencies=[Depends(verify_token)])`:
    *   Includes the main (protected) search router.
    *   `prefix="/api"`: Endpoints will be under `/api` (e.g., `/api/search`).
    *   `tags=["Search"]`: Groups them under "Search".
    *   `dependencies=[Depends(verify_token)]`: This is a critical security measure. For *every* endpoint defined within `search_router`, the `verify_token` dependency function will be executed *before* the actual route handler. If `verify_token` raises an `HTTPException` (e.g., token missing or invalid), the route handler will not be called.
*   `app.include_router(admin_router, prefix="/api", tags=["Admin"], dependencies=[Depends(verify_admin)])`:
    *   Includes the admin router.
    *   `prefix="/api"`, `tags=["Admin"]`: Similar to above.
    *   `dependencies=[Depends(verify_admin)]`: Ensures that *only* users with administrative privileges (as determined by `verify_admin`) can access these routes. This adds an additional layer of authorization beyond basic token verification.
*   `app.include_router(report_router, prefix="/api", tags=["Report"])`:
    *   Includes the report router.
    *   `prefix="/api"`, `tags=["Report"]`: Standard prefix and tag.
    *   **Note**: No `dependencies` are specified here, indicating that routes within `report_router` do not require authentication at this global inclusion level. Individual routes within the `report_router` *could* still have their own dependencies.
*   `app.include_router(player_dashboard_router, prefix="/api", tags=["Player Dashboard"])`:
    *   Includes the player dashboard router.
    *   `prefix="/api"`, `tags=["Player Dashboard"]`: Standard prefix and tag.
    *   **Note**: Although the comment states "Protected routes", the `dependencies` parameter is *not* explicitly set here. This implies that the protection (e.g., `Depends(verify_token)`) is either applied individually to each route within the `player_dashboard_router` module or through another shared dependency injection mechanism within that module itself.
*   `app.include_router(game_dashboard_router, prefix="/api", tags=["Game Dashboard"])`:
    *   Includes the game dashboard router.
    *   `prefix="/api"`, `tags=["Game Dashboard"]`: Standard prefix and tag.
    *   **Note**: Similar to `player_dashboard_router`, the `dependencies` parameter is not explicitly set here, despite the comment "Protected routes". Protection is presumed to be handled within the `game_dashboard_router` module.
*   `app.include_router(search_public_router, prefix="/api", tags=["Search"])`:
    *   Includes the public search router.
    *   `prefix="/api"`, `tags=["Search"]`: Standard prefix and tag.
    *   **Note**: This router is explicitly designated for public access, so no authentication dependencies are expected or specified.
*   `app.include_router(login_issue_router, prefix="/api", tags=["Login Issues"])`:
    *   Includes the login issue reporting router.
    *   `prefix="/api"`, `tags=["Login Issues"]`: Standard prefix and tag.
    *   **Note**: No authentication dependencies are specified, as this feature is likely intended for unauthenticated users experiencing login problems.

#### 3.9. Root Endpoint

```python
# Define root endpoint that serves as a basic health check and welcome message
@app.get("/")
def home():
    return {"message": "Welcome to R2 API"}
```

*   `@app.get("/")`: This is a FastAPI decorator that registers the `home` function as a handler for HTTP GET requests to the root path (`/`) of the application.
*   `def home():`: Defines the `home` function, which is the actual endpoint handler.
*   `return {"message": "Welcome to R2 API"}`: When a GET request is made to `/`, this function will return a JSON response containing a simple welcome message. This often serves as a basic health check or a friendly entry point for API users.

---
