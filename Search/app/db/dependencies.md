# Documentation for `dependencies.py`

This document provides a comprehensive, line-by-line technical explanation of the provided Python script, which is designed to manage database connection dependencies for FastAPI applications.

---

## Technical Documentation: `app.db.dependencies`

### 1. High-Level Overview

The `app.db.dependencies` module serves as a central hub for managing database connection instances within a FastAPI application. Its primary purpose is to provide reusable and robust FastAPI dependency functions that ensure database connections (represented by various "adapter" classes) are properly opened, used, and then safely closed and returned to their respective connection pools.

It achieves this by:
*   Utilizing Python's `contextlib.contextmanager` to create a generic context manager (`get_db_adapter`) that handles the lifecycle of any database adapter inheriting from `PGSQLAdapter`.
*   Defining specific FastAPI `Depends` functions for each type of database adapter (e.g., `AuthAdapter`, `AdminAdapter`), which leverage the generic context manager.
*   Facilitating a clean and safe pattern for injecting database access logic into FastAPI route handlers, removing the burden of manual connection management from individual endpoints.

This design ensures that:
*   Connections are always returned to the pool, even if errors occur during request processing.
*   Code remains DRY (Don't Repeat Yourself) regarding connection handling.
*   The application benefits from connection pooling for efficiency.

### 2. Module Contents

```python
"""
Database dependency module

This module provides FastAPI dependency functions that handle database connection
management in a clean, safe way using context managers and dependency injection.

Usage in route files:
    from app.db.dependencies import get_auth_adapter

    @router.post("/endpoint")
    async def my_endpoint(request: MyRequest, auth_adapter: AuthAdapter = Depends(get_auth_adapter)):
        # Use auth_adapter without worrying about connection management
        result = auth_adapter.some_method()
        # No need to call auth_adapter.close() - handled automatically
        return result
"""
```
This is the module's docstring, which clearly outlines its purpose and provides a practical example of how the dependency functions should be used within a FastAPI route. It highlights the main benefits: automatic connection management and safe resource handling.

### 3. Imports

```python
from fastapi import Depends
from contextlib import contextmanager
from typing import Iterator, TypeVar, Type, Generic, Any
from typing import Generator
from fastapi import Depends
```
*   **`from fastapi import Depends`**: Imports the `Depends` utility from FastAPI, which is crucial for declaring dependencies in route handlers. `Depends` is used to inject resources or values provided by other functions into endpoint functions.
*   **`from contextlib import contextmanager`**: Imports the `contextmanager` decorator. This decorator is used to convert a simple generator function into a context manager, allowing it to be used with Python's `with` statement.
*   **`from typing import Iterator, TypeVar, Type, Generic, Any`**: Imports various type hinting utilities:
    *   `Iterator`: Used to specify that a function yields an iterable sequence of items, typically used for context managers that yield a single item.
    *   `TypeVar`: Used to declare a type variable, enabling the creation of generic types and functions.
    *   `Type`: Used to indicate that a parameter expects a class (type) rather than an instance of a class.
    *   `Generic`: Base class for generic types. (Note: `Generic` and `Any` are imported but not directly used in *this* module, suggesting they might be part of a standard `typing` import block in the project or remnants from earlier iterations).
*   **`from typing import Generator`**: Imports the `Generator` type, which is used to explicitly type hint generator functions. FastAPI `Depends` functions that manage resources often return `Generator`s to enable cleanup logic.
*   **`from fastapi import Depends`**: This is a duplicate import of `Depends`. It has no functional impact but could be consolidated.

```python
from app.db.pgsql_adapter import PGSQLAdapter
from app.db.auth_adapter import AuthAdapter
from app.db.admin_adapter import AdminAdapter
from app.db.playerDashboard import PlayerDashboardAdapter
from app.db.gameDashboard import GameDashboardAdapter
from app.db.connection import get_db_pool
from app.db.postgres_handler import PostgresHandler
```
These lines import custom database adapter classes and utility functions defined elsewhere within the `app.db` package. Each adapter class is expected to encapsulate specific database operations and inherit from a base `PGSQLAdapter` (or at least provide a `close()` method).
*   **`PGSQLAdapter`**: A base class for PostgreSQL adapters, likely providing common methods for database interaction and connection management.
*   **`AuthAdapter`**: An adapter specifically for authentication-related database operations.
*   **`AdminAdapter`**: An adapter for administrative database tasks.
*   **`PlayerDashboardAdapter`**: An adapter for operations related to player dashboards.
*   **`GameDashboardAdapter`**: An adapter for operations related to game dashboards.
*   **`get_db_pool`**: (Imported but not used in this specific module) This function presumably retrieves or initializes a PostgreSQL connection pool, which the adapters might use internally to get connections.
*   **`PostgresHandler`**: Another class for PostgreSQL interaction, potentially a more generic handler or an alternative to the `PGSQLAdapter` hierarchy.

### 4. Type Variables

```python
# Define a generic type for the adapter classes
AdapterT = TypeVar('AdapterT', bound=PGSQLAdapter)
```
*   **`AdapterT = TypeVar('AdapterT', bound=PGSQLAdapter)`**: This line defines a generic type variable named `AdapterT`.
    *   `TypeVar` allows for writing functions and classes that can work with different types while maintaining type safety.
    *   `bound=PGSQLAdapter` specifies an upper bound for `AdapterT`. This means that any type substituted for `AdapterT` must either be `PGSQLAdapter` itself or a subclass of `PGSQLAdapter`. This constraint is crucial because the `get_db_adapter` context manager assumes that the adapter class will have a `close()` method, which `PGSQLAdapter` is expected to provide.

### 5. `get_db_adapter` Context Manager

```python
@contextmanager
def get_db_adapter(adapter_class: Type[AdapterT]) -> Iterator[AdapterT]:
    """
    Context manager that creates a database adapter instance and ensures
    the connection is properly returned to the pool when done, even if
    exceptions occur.
    
    Args:
        adapter_class: The adapter class to instantiate
        
    Yields:
        An instance of the specified adapter class
    """
    print(f"Opening Adapter: {adapter_class}")

    adapter = adapter_class()
    try:
        yield adapter
    finally:
        # This ensures the connection is returned to the pool even if exceptions occur
        if adapter:
            adapter.close()
```
*   **`@contextmanager`**: This decorator transforms the `get_db_adapter` generator function into a context manager. This allows it to be used with a `with` statement, ensuring that setup (before `yield`) and teardown (after `yield`) logic are automatically executed.
*   **`def get_db_adapter(adapter_class: Type[AdapterT]) -> Iterator[AdapterT]:`**:
    *   **`adapter_class: Type[AdapterT]`**: This parameter expects a *class* (the `Type` hint) that is either `PGSQLAdapter` or one of its subclasses (due to `AdapterT`'s `bound`). This allows the context manager to be generic and instantiate any compatible adapter.
    *   **`-> Iterator[AdapterT]`**: The return type hint indicates that this function yields an object of type `AdapterT`. When used as a context manager, the value yielded by the function becomes the value assigned to the `as` variable in the `with` statement.
*   **`print(f"Opening Adapter: {adapter_class}")`**: A debugging or logging statement indicating which adapter class is currently being initialized. This is useful for tracing the lifecycle of database connections.
*   **`adapter = adapter_class()`**: An instance of the provided `adapter_class` is created. This assumes that the adapter classes have a no-argument constructor or that default values are sufficient for initialization.
*   **`try...yield adapter...finally:`**: This block defines the core logic of the context manager:
    *   **`yield adapter`**: The instantiated adapter object is yielded. Execution of `get_db_adapter` pauses here, and the adapter instance is made available to the code block following the `with ... as adapter:` statement. When that block finishes (either normally or due to an exception), control returns to this generator.
    *   **`finally:`**: This block ensures that the cleanup logic is always executed, regardless of whether the code within the `with` block completes successfully or raises an exception.
    *   **`if adapter: adapter.close()`**: Inside the `finally` block, the `close()` method of the adapter instance is called. This is critical for returning the database connection to its pool, preventing resource leaks and ensuring efficient connection reuse. The `if adapter:` check is a minor safeguard, though `adapter` should always be initialized if `adapter_class()` succeeded.

### 6. FastAPI Dependency Functions for Specific Adapters

These functions are designed to be used directly with FastAPI's `Depends`. They are generator functions that leverage the `get_db_adapter` context manager to provide a specific adapter instance and ensure its proper closure.

```python
def get_auth_adapter() -> AuthAdapter:
    """Get an AuthAdapter instance with proper connection management."""
    with get_db_adapter(AuthAdapter) as adapter:
        yield adapter
```
*   **`def get_auth_adapter() -> AuthAdapter:`**: Defines a dependency function for `AuthAdapter`. It returns an `AuthAdapter` instance.
*   **`with get_db_adapter(AuthAdapter) as adapter:`**: It uses the generic `get_db_adapter` context manager, passing `AuthAdapter` as the class to instantiate. The `AuthAdapter` instance is then available as `adapter`.
*   **`yield adapter`**: The `AuthAdapter` instance is yielded. FastAPI's `Depends` mechanism understands that if a generator function yields a value, that value should be injected into the dependent function. Crucially, FastAPI also knows to execute the `finally` block (or any code after the `yield`) of the generator when the request finishes or the dependency is no longer needed.

The following dependency functions follow the exact same pattern, differing only in the specific adapter class they instantiate and yield:

```python
def get_admin_adapter() -> AdminAdapter:
    """Get an AdminAdapter instance with proper connection management."""
    with get_db_adapter(AdminAdapter) as adapter:
        yield adapter

def get_player_dashboard_adapter() -> PlayerDashboardAdapter:
    """Get a PlayerDashboardAdapter instance with proper connection management."""
    with get_db_adapter(PlayerDashboardAdapter) as adapter:
        yield adapter

def get_game_dashboard_adapter() -> GameDashboardAdapter:
    """Get a GameDashboardAdapter instance with proper connection management."""
    with get_db_adapter(GameDashboardAdapter) as adapter:
        yield adapter

def get_pgsql_adapter() -> PGSQLAdapter:
    """Get a PGSQLAdapter instance with proper connection management."""
    with get_db_adapter(PGSQLAdapter) as adapter:
        yield adapter
```
These functions provide instances of `AdminAdapter`, `PlayerDashboardAdapter`, `GameDashboardAdapter`, and the base `PGSQLAdapter`, respectively, each with automatic connection management via `get_db_adapter`.

```python
def get_postgres_handler() -> PostgresHandler:
    """
    Creates a PostgresHandler and ensures it's properly closed after use.
    Generator = a function that can 'yield' values and clean up afterward
    """
    with get_db_adapter(PostgresHandler) as adapter:
        yield adapter
```
*   **`def get_postgres_handler() -> PostgresHandler:`**: Similar to the others, this function provides an instance of `PostgresHandler`.
*   **Docstring Note**: The docstring specifically mentions `Generator = a function that can 'yield' values and clean up afterward`. This is a helpful clarification reinforcing the role of generator functions in FastAPI dependencies for resource management.

### 7. Main Execution Flow (`__main__` block analysis)

This script does not contain a `if __name__ == "__main__":` block, meaning it is not intended to be run directly as a standalone script. Instead, it is designed to be imported and used as a module providing dependency functions within a larger FastAPI application.

### 8. Important Implementation Details, Dependencies, and Configuration

*   **FastAPI Dependency Injection**: The core principle is leveraging FastAPI's dependency injection system (`Depends`). This allows route handlers to declare their required database adapters as parameters, and FastAPI automatically provides and manages them.
*   **Context Managers for Resource Management**: The `contextlib.contextmanager` decorator combined with `try...finally` blocks is critical for ensuring that database connections are always released (via `adapter.close()`) back to their pools, even if errors occur during request processing. This prevents connection leaks and optimizes resource utilization.
*   **Connection Pooling**: Although not explicitly shown in this module, the `adapter.close()` method implies that the underlying database connections are managed by a connection pool (likely initialized via `app.db.connection.get_db_pool`). This is essential for performance in high-concurrency web applications.
*   **Extensibility**: Adding a new database adapter type simply involves:
    1.  Creating a new adapter class (e.g., `NewFeatureAdapter`) that inherits from `PGSQLAdapter` (or at least implements a `close()` method).
    2.  Importing the new adapter class into `app.db.dependencies`.
    3.  Creating a new dependency function (e.g., `get_new_feature_adapter`) that uses `get_db_adapter` with the new class.
*   **Logging**: The `print(f"Opening Adapter: {adapter_class}")` statement provides basic logging for adapter instantiation. In a production environment, this would typically be replaced with a more robust logging framework (e.g., Python's `logging` module).
*   **Error Handling**: The `try...finally` block within `get_db_adapter` ensures cleanup. Any exceptions occurring within the code block that uses the yielded adapter will propagate up the call stack, but the `finally` block will still execute to close the connection.
*   **Asynchronous Operations**: While FastAPI endpoints are often `async`, the `get_db_adapter` and the dependency functions themselves are synchronous generator functions. This is acceptable for many database drivers, especially if the `adapter.close()` call is quick. For truly asynchronous database operations, the adapters themselves (e.g., `PGSQLAdapter`) would need to use `asyncpg` or similar async drivers, and the context manager might need to be an `async` context manager (`@asynccontextmanager`) and the dependency functions `async def` generators that `yield await adapter`. This current setup implies the adapters handle their own (potentially synchronous) connection acquisition from an async-compatible pool, or are synchronous adapters.
