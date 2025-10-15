# Documentation for `search.py`

This document provides a comprehensive technical breakdown of the `search_api` Python module. It details the module's purpose, its components, functions, and API endpoints, including their functionalities, parameters, and operational logic.

---

## `search_api` Module Documentation

### Overview

The `search_api` module is a core component of the R2 system, designed to expose API endpoints for searching, retrieving, and exporting sports-related data. It leverages FastAPI for building robust web APIs and integrates with various internal services for natural language processing (NLP), SQL generation, and PostgreSQL database interactions. The module supports authenticated search requests, public name suggestion features, and the logging of export operations.

Key functionalities include:
*   Converting natural language queries into structured database queries (AQL and SQL).
*   Executing these queries against a PostgreSQL database.
*   Providing auto-complete suggestions for player and team names.
*   Exporting query results to CSV format with customizable column ordering.
*   Logging user export activities.
*   Retrieving a user's recent search queries.

### Dependencies and Imports

The module relies on several external libraries and internal application components:

*   **`fastapi`**: The web framework for building the API.
    *   `APIRouter`: To organize API endpoints into modular units.
    *   `HTTPException`: For raising standardized HTTP errors.
    *   `Depends`: For dependency injection, managing resource lifecycles (e.g., database connections) and authentication.
*   **`app.models.search_models`**: Custom Pydantic models defining the structure of request and response bodies.
    *   `SearchRequest`: Model for incoming search queries.
    *   `SearchResponse`: Model for search results.
    *   `ExportRequest`: Model for data export requests.
    *   `ExportLogRequest`: Model for logging export activities.
    *   `RecentQuery`: Model for representing a recent user query.
*   **`fastapi.responses.StreamingResponse`**: Used for sending large data (like CSV files) as a stream.
*   **`app.api.routes.auth.verify_token`**: A custom dependency function responsible for authenticating users via JWT tokens.
*   **`app.services.nlp_processor.NlpProcessor`**: A service that processes natural language input to convert it into an Abstract Query Language (AQL).
*   **`app.services.sql_builder.SQLBuilder`**: A service that translates AQL into executable SQL queries.
*   **`app.db.pgsql_adapter.PGSQLAdapter`**: A database adapter specifically for interacting with PostgreSQL.
*   **`app.db.dependencies.get_pgsql_adapter`**: A FastAPI dependency provider for `PGSQLAdapter` instances.
*   **`app.db.dependencies.get_postgres_handler`**: A FastAPI dependency provider for `PostgresHandler` instances.
*   **`app.llms.get_llm_connector`**: A function to retrieve a Language Model (LLM) connector instance based on specified LLM and model.
*   **`logging`**: Python's standard library for logging events and errors.
*   **`time`**: Used for measuring execution times of operations.
*   **`io`**: Provides tools for working with I/O streams, particularly `StringIO` for in-memory text streams, used in CSV export.
*   **`app.constants`**: A module defining various constant values used across the application.
    *   `PlayerClass`, `Positions`, `SortingFields`, `QueryTypes`, `Qualifiers`, `PostgreSQL`: Enums or classes holding predefined values for categories, fields, and query parameters.
*   **`app.services.stat_matcher.StatMatcher`, `STModelChoice`**: Components for identifying and matching statistical terms within natural language queries.
*   **`pydantic.BaseModel`**: The base class for defining data models with validation.
*   **`typing.List`**: Type hinting for list objects.
*   **`app.data_config.mappings.mappings_handler.MappingsHandler`**: A service to manage and provide data mappings (e.g., between UI labels and database column names, or for conferences).
*   **`pandas as pd`**: A powerful data manipulation library, used here for creating DataFrames and exporting to CSV.
*   **`decouple.config`**: A library for handling configuration variables (though `config` itself isn't directly used after import in this specific file, it indicates environment variable usage elsewhere).
*   **`app.db.fuzzy_logic.FuzzyMatch`**: A utility for performing fuzzy matching on names (e.g., player names, team names, conference names) to correct typos or variations.
*   **`app.db.postgres_handler.PostgresHandler`**: A specific handler for PostgreSQL operations, possibly for simpler or more direct queries than `PGSQLAdapter`.

### Global Variables and Initialization

The module initializes several key components at startup:

*   **`router = APIRouter()`**:
    *   Initializes the main FastAPI router. This router will be used to register all authenticated API endpoints within this module.
*   **`sql_builder = SQLBuilder()`**:
    *   Creates an instance of the `SQLBuilder` class. This instance is responsible for constructing SQL queries from the AQL representation.
*   **`mappings_handler = MappingsHandler()`**:
    *   Creates an instance of the `MappingsHandler` class. This component provides various data mappings, such as those between user-friendly labels and database fields, or lists of conferences.
*   **`team_start_time = time.time()` / `teams_end_time = time.time()`**:
    *   Variables used to measure the time taken to load team data.
*   **`TEAMS = postgres_handler.get_all_teams()`**:
    *   **Note**: This line is commented out in the provided code snippet but indicates the intended way to load all team names using `postgres_handler`. In `get_pos_class_team_values`, `postgres_handler.get_all_teams()` is called directly.
*   **`print(f"Loaded teams in {teams_end_time - team_start_time} seconds")`**:
    *   Prints the duration of team data loading to the console.
*   **`s = time.time()` / `e = time.time()`**:
    *   Variables used to measure the time taken to load statistical matchers.
*   **`STAT_MATCHERS = StatMatcher.get_all_stat_matchers()`**:
    *   Loads all predefined statistical matchers at application startup. These matchers are crucial for the `NlpProcessor` to understand and interpret statistical terms in natural language queries.
*   **`print(f"Time taken to load stat_matchers: {e-s}")`**:
    *   Prints the duration of statistical matcher loading to the console.
*   **`logging.basicConfig(level=logging.INFO)`**:
    *   Configures the basic logging system, setting the default logging level to `INFO`.
*   **`logger = logging.getLogger(__name__)`**:
    *   Creates a logger instance specifically for this module, using the module's name.
*   **`COLUMN_ORDER_MAP`**:
    *   A dictionary defining the fixed order of columns for CSV exports. The keys are tuples of `(sport_code, entity)`, and values are lists of desired column names. This ensures consistent formatting for downloaded data.

### Pydantic Models

#### `NameSuggestionsRequest(BaseModel)`

*   **Purpose**: Defines the structure for requests made to the name suggestion endpoint.
*   **Attributes**:
    *   `SportCode` (`str`): The identifier for the sport (e.g., "MFB" for Men's Football).
    *   `Entity` (`str`): The type of entity for which suggestions are requested (e.g., "Player" or "Team").
    *   `Prefix` (`str`): The partial name string to be matched for suggestions (e.g., "Tom" for "Tom Brady").

#### `NameSuggestionsResponse(BaseModel)`

*   **Purpose**: Defines the structure for responses from the name suggestion endpoint.
*   **Attributes**:
    *   `Suggestions` (`List[str]`): A list of matching name strings.

### Functions

#### `get_pos_class_team_values(sport_code: str, qualifiers: dict, postgres_handler: PostgresHandler = Depends(get_postgres_handler))`

*   **Purpose**: This function dynamically fetches and filters valid options for UI elements such as positions, player classes, teams, and conferences. These options are tailored based on the provided `sport_code` and any already existing `qualifiers` in the query, ensuring that users are only presented with relevant choices.
*   **Parameters**:
    *   `sport_code` (`str`): A string identifying the sport (e.g., "MFB", "MBB", "WBB").
    *   `qualifiers` (`dict`): A dictionary containing existing search qualifiers (e.g., `{'POSITION': 'QB'}`). If a qualifier is already set, its corresponding dropdown option list will be `None` to indicate it cannot be further filtered by the user.
    *   `postgres_handler` (`PostgresHandler`, injected via `Depends(get_postgres_handler)`): An instance of the `PostgresHandler` for database interactions, specifically to fetch all team names.
*   **Return Values (`tuple`)**:
    *   `positions` (`list` or `None`): A list of available positions for the sport. `None` if `SortingFields.POSITION` is already in `qualifiers`.
    *   `player_classes` (`list` or `None`): A list of available player class years. `None` if `SortingFields.PLAYERCLASS` is already in `qualifiers`.
    *   `teams` (`list`): A list of all available team names. `None` if `SortingFields.OPPONENTTEAMNAME` is already in `qualifiers`.
    *   `conferences` (`dict` or `None`): A dictionary of conferences for the sport. If `Qualifiers.OPPONENT_CONFERENCE_NAME` is in `qualifiers`, it's filtered to only the specified conference.
*   **Implementation Logic**:
    1.  **Positions Initialization**: It first initializes the `positions` list based on the `sport_code`. For "MFB" (football), it includes a comprehensive list of football positions. For "MBB" or "WBB" (basketball), it lists common basketball positions. For other sport codes, it defaults to an empty list.
    2.  **Player Classes Initialization**: Similarly, `player_classes` are initialized. For "MFB", it includes football-specific class years. For "MBB" or "WBB", it lists common undergraduate class years. Defaults to an empty list otherwise.
    3.  **Conferences Initialization**: If the `sport_code` is for football or basketball, it fetches all relevant conferences using `mappings_handler.get_conferences(sport_code)`.
    4.  **Qualifier Filtering**:
        *   It checks if `SortingFields.POSITION.value` is present in `qualifiers`. If true, `positions` is set to `None` as the position is already specified in the query.
        *   It checks if `SortingFields.PLAYERCLASS.value` is present in `qualifiers`. If true, `player_classes` is set to `None`.
    5.  **Teams Fetching**: It fetches all available team names by calling `postgres_handler.get_all_teams()`.
    6.  **Opponent Team Filtering**: If `SortingFields.OPPONENTTEAMNAME.value` is in `qualifiers`, `teams` is set to `None`.
    7.  **Conference Filtering by Qualifier**: If `Qualifiers.OPPONENT_CONFERENCE_NAME.value` is in `qualifiers`, it filters the `conferences` dictionary to include only the conference specified in the qualifier.
    8.  Finally, it returns the calculated `positions`, `player_classes`, `TEAMS`, and `conferences`.

### FastAPI Endpoints

#### `POST /search` - `search_players`

*   **HTTP Method**: `POST`
*   **Path**: `/search`
*   **Response Model**: `SearchResponse`
*   **Description**: This is a protected endpoint that allows authenticated users to execute complex search queries on sports data using natural language. It handles the full pipeline from NLP to SQL execution and result formatting.
*   **Dependencies**:
    *   `username: str = Depends(verify_token)`: Authenticates the user and provides their username from the JWT token.
    *   `db_adapter: PGSQLAdapter = Depends(get_pgsql_adapter)`: Injects an instance of `PGSQLAdapter` for database interactions.
    *   `postgres_handler: PostgresHandler = Depends(get_postgres_handler)`: Injects an instance of `PostgresHandler` for specific PostgreSQL operations (e.g., AQL caching, team lookups).
*   **Request Body (`SearchRequest`)**:
    *   Contains the natural language `BasicQuery`, `SportCode`, `Entity`, `TeamCode`, pagination (`PageSize`, `PageNumber`), sorting (`SortBy`, `SortOrder`), `Filters`, `Llm`, `LlmModel`, `UseRAG`, and an optional pre-generated `aql_output`.
*   **Implementation Logic**:
    1.  **Logging**: Logs the incoming search request and the authenticated `username`.
    2.  **Filter Pre-processing**:
        *   Modifies `GAME_TYPE` filters: If both "Regular" and "Post Season" are selected, the filter is removed. If only "Regular" is selected, "Regular" is replaced with "NULL". If only "Post Season" is selected, "Post Season" is replaced with "NOT NULL". This logic seems to convert UI-friendly values to SQL-friendly expressions.
        *   Modifies `PERIOD_TYPE` filters: If both "Regular" and "Overtime" are selected, the filter is removed. Otherwise, the selected period type is converted to uppercase.
    3.  **AQL Generation / Cache Lookup**:
        *   Initializes the `llm_connector` (based on `request.Llm` and `request.LlmModel`) and `NlpProcessor` (with `STAT_MATCHERS`).
        *   Records `start_time` for inference timing.
        *   Checks if `request.aql_output` is already provided. If so, it uses that directly.
        *   Otherwise, it attempts to retrieve the AQL from a cache (`postgres_handler.get_cached_aql`) using the `username` and `BasicQuery`.
        *   If not found in cache, it calls `nlp_processor.convert_to_aql` to generate AQL from the natural language query, considering `SportCode`, `Entity`, `TeamCode`, and `UseRAG`.
        *   If AQL was newly generated, it's saved to the cache (`postgres_handler.cache_aql`).
    4.  **Fuzzy Matching Qualifiers**:
        *   An instance of `FuzzyMatch` is created.
        *   For various qualifiers like `TEAM_NAME`, `OPPONENT_TEAM_NAME`, `OPPONENT_CONFERENCE_NAME`, `TEAM_CONFERENCE_NAME`, and `PLAYER_NAME` within the generated `aql_output['qualifiers']`, fuzzy matching is applied using the `FuzzyMatch` service to ensure that names match known entries in the database.
    5.  **Team Code Inference**:
        *   If `TEAM_NAME` is not explicitly present in the AQL qualifiers, it attempts to infer the `team_name` from `request.TeamCode` by querying the `postgres_handler`. If a team name is found, it's added to the AQL qualifiers.
        *   If `TEAM_NAME` is in qualifiers and its value is "ALL", this qualifier is removed, implying a search across all teams.
        *   The `fuzzy_match` instance is closed.
    6.  **AQL-Only Response**: If `request.AQLOnly` is `True`, the function immediately returns a `SearchResponse` containing only the generated AQL, skipping SQL generation and execution.
    7.  **Qualifiers Validation**: Ensures `aql_output['qualifiers']` is a valid dictionary, initializing it if missing.
    8.  **Sort By Mapping**:
        *   Normalizes the `request.SortBy` string.
        *   Attempts to map `sort_by` to a database column name. It first checks `metadata_mapping` from `mappings_handler`.
        *   If no match in metadata, it falls back to `stat_mappings` for the specific `SportCode` and `Entity`. If a match is found (e.g., `short_label` matches `sort_by`), `sort_by` is updated to the corresponding `stat` field.
    9.  **SQL Generation**: `sql_builder.convert_aql_to_sql` is called to generate the final `sql_query`, `selected_columns`, and `stat_columns` based on the AQL, sport code, entity, filters, query type, pagination, and sorting.
    10. **SQL Execution**:
        *   Records `start_time_q` for query execution timing.
        *   `db_adapter.execute_query` executes the generated `sql_query` and fetches results along with the total count.
        *   Records `end_time_q`.
    11. **Filter Options for UI**:
        *   Determines available `game_type` and `period_type` filter options for the UI based on the `stat_period` in the AQL output.
    12. **Dynamic UI Values**: Calls `get_pos_class_team_values` (described above) to fetch available `positions`, `player_classes`, `teams`, and `conferences` for dynamic UI dropdowns, considering existing `qualifiers`.
    13. **Response**: Constructs and returns a `SearchResponse` object containing the `total_count`, `results`, `aql_output`, `sql_query`, various timing metrics, and UI filter options.
    14. **Error Handling**: Catches any `Exception` during the process, logs it, and raises an `HTTPException` with a 500 status code.

#### `POST /export` - `export_data`

*   **HTTP Method**: `POST`
*   **Path**: `/export`
*   **Description**: A protected endpoint to export search results (provided as a SQL query) into a CSV file.
*   **Dependencies**:
    *   `username: str = Depends(verify_token)`: Authenticates the user.
    *   `postgres_handler: PostgresHandler = Depends(get_postgres_handler)`: Injected but not directly used in the current body of the function.
*   **Request Body (`ExportRequest`)**:
    *   Contains the `sql` query string, `sport_code`, `entity`, and `stat_columns` (list of column names corresponding to stats).
*   **Implementation Logic**:
    1.  **DB Adapter**: A new `PGSQLAdapter` instance is created for this specific export operation.
    2.  **Execute Query**: `db_adapter.execute_query` is called to execute the provided `request.sql` query. It retrieves all results without pagination.
    3.  **Data Transformation**: Iterates through the raw results. For each result, it combines the `metadata` dictionary and `stats` dictionary into a single flat dictionary (`res`), which is then appended to `all_res`.
    4.  **DataFrame Creation**: A `pandas.DataFrame` is created from the `all_res` list of dictionaries.
    5.  **Column Ordering**:
        *   It constructs a `key` tuple `(request.sport_code, request.entity)`.
        *   If this key exists in `COLUMN_ORDER_MAP`, it retrieves the `desired_columns` list.
        *   It then filters `desired_columns` to only include those that actually exist in the `df.columns` to prevent errors.
        *   The DataFrame `df` is reordered to match the `ordered_columns`.
    6.  **CSV Stream Generation**: An `io.StringIO` object is created to serve as an in-memory text buffer. The DataFrame is then written to this buffer as a CSV string, with `index=False` to prevent Pandas from writing the DataFrame index to the CSV.
    7.  **Streaming Response**: A `StreamingResponse` is created. It takes an iterator of the `StringIO` content, sets `media_type="text/csv"`, and adds a `Content-Disposition` header to prompt the browser to download the file named `export.csv`.
    8.  **Close DB**: The `db_adapter` connection is closed.
    9.  **Return**: Returns the `StreamingResponse` and the `total_count`.

#### `POST /name-suggestions` - `post_name_suggestions`

*   **HTTP Method**: `POST`
*   **Path**: `/name-suggestions`
*   **Response Model**: `NameSuggestionsResponse`
*   **Description**: This is a public endpoint that provides auto-complete suggestions for player or team names based on a given prefix. It performs case-insensitive prefix matching.
*   **Dependencies**:
    *   `postgres_handler: PostgresHandler = Depends(get_postgres_handler)`: Injects an instance of `PostgresHandler` for fetching name suggestions.
*   **Request Body (`NameSuggestionsRequest`)**:
    *   Contains `SportCode`, `Entity` (either "Player" or "Team"), and `Prefix`.
*   **Implementation Logic**:
    1.  **Logging**: Logs the incoming suggestion request.
    2.  **Entity-Based Search**:
        *   If `request.Entity` is "Player": Calls `postgres_handler.search_players_by_prefix(prefix)` to fetch player names starting with the `prefix`. The results are converted to a `set` to ensure uniqueness.
        *   If `request.Entity` is "Team": Calls `postgres_handler.search_teams_by_prefix(prefix)` to fetch team names starting with the `prefix`. Results are processed to extract team names and added to a `set` for uniqueness.
    3.  **No Suggestions Found**: If the `suggestions` set remains empty after the search, an `HTTPException` with status code 404 is raised.
    4.  **Response**: Returns a `NameSuggestionsResponse` object containing the list of unique `Suggestions`.
    5.  **Error Handling**: Catches `HTTPException` (re-raises it) and other `Exception` types (logs and raises `HTTPException` with a 500 status code).

#### `POST /export-log` - `export_log`

*   **HTTP Method**: `POST`
*   **Path**: `/export-log`
*   **Description**: A protected endpoint to log user export activities. It also allows for re-ordering the data (provided in the request body) according to a predefined column order before logging the request and returning the ordered data.
*   **Dependencies**:
    *   `username: str = Depends(verify_token)`: Authenticates the user.
    *   `db_adapter: PGSQLAdapter = Depends(get_pgsql_adapter)`: Injects an instance of `PGSQLAdapter` for logging.
*   **Request Body (`ExportLogRequest`)**:
    *   Contains `SportCode`, `Entity`, `sql` (the query that was exported), `RowCount`, and `data` (the actual data exported as a list of dictionaries).
*   **Implementation Logic**:
    1.  **Logging**: Logs the incoming export log request.
    2.  **Data Column Ordering**:
        *   Initializes `ordered_data` with the `request.data`.
        *   Forms a `key` `(request.SportCode, request.Entity)`.
        *   If this key exists in `COLUMN_ORDER_MAP` and `request.data` is not empty:
            *   Creates a Pandas DataFrame `df` from `request.data`.
            *   Creates a case-insensitive mapping of DataFrame columns for flexible matching.
            *   Constructs `ordered_columns` based on `COLUMN_ORDER_MAP`, ensuring only existing columns are included and preserving their original casing.
            *   Adds any remaining columns from the original DataFrame that were not specified in `COLUMN_ORDER_MAP` to the end.
            *   Reorders the DataFrame columns and converts it back to a list of dictionaries (`df.to_dict('records')`) for `ordered_data`.
    3.  **Log Export Event**: Calls `db_adapter.execute_export_log` to record the export event, including the `username`, `sport_code`, `entity`, `query_input` (the SQL), and `row_count`.
    4.  **Response**: Returns a dictionary containing the `export_log` result (from `db_adapter`) and the `ordered_data`.
    5.  **Error Handling**: Catches any `Exception`, logs it, and raises an `HTTPException` with a 500 status code.

#### `GET /recent-queries` - `get_recent_queries`

*   **HTTP Method**: `GET`
*   **Path**: `/recent-queries`
*   **Response Model**: `List[RecentQuery]`
*   **Description**: A protected endpoint to retrieve a list of the authenticated user's most recent search queries for a specific sport and entity.
*   **Dependencies**:
    *   `username: str = Depends(verify_token)`: Authenticates the user.
    *   `db_adapter: PGSQLAdapter = Depends(get_pgsql_adapter)`: Injects an instance of `PGSQLAdapter` for fetching queries.
*   **Query Parameters**:
    *   `sportCode` (`str`): The sport code to filter recent queries by.
    *   `entity` (`str`): The entity type (e.g., "Player", "Team") to filter recent queries by.
    *   `limit` (`int`, default: 10): The maximum number of recent queries to return.
*   **Implementation Logic**:
    1.  **Logging**: Logs the request for recent queries.
    2.  **Fetch Queries**: Calls `db_adapter.get_recent_queries` with the `username`, `limit`, `sportCode`, and `entity` to retrieve the relevant queries from the database.
    3.  **Response**: Returns the `queries` list, which is a list of `RecentQuery` objects.
    4.  **Error Handling**: Catches any `Exception`, logs it, and raises an `HTTPException` with a 500 status code.

---
