# Documentation for `search_models.py`

This document provides comprehensive technical documentation for the given Python script, detailing its purpose, structure, and the functionality of each defined component.

---

## Technical Documentation: Data Models for API Interaction

### 1. High-Level Overview

This Python script primarily defines a set of Pydantic `BaseModel` classes. These models serve as structured data schemas for various API requests and responses within a larger application, likely related to sports analytics or a similar domain. Pydantic is used for data validation, serialization, and deserialization, ensuring that data conforming to these models adheres to predefined types, constraints, and structures.

The models cover:
*   **Search Request**: Defining parameters for querying structured data.
*   **Export Request**: Specifying details for exporting generated SQL data.
*   **Recent Query**: Representing previously executed or example queries.
*   **Export Log Request**: Structuring data for logging export operations.
*   **Conference Data**: A helper model for conference-related information.
*   **Search Response**: The structure for the results returned from a search query, including data, metadata, and performance metrics.

The script leverages Pydantic's `Field` and `conint` for advanced validation and default value specification, along with standard Python `typing` hints for clarity and static analysis. It also imports custom Enum types (`LLMConnectors`, `LLMModels`) for type-safe configuration of Language Model (LLM) parameters.

### 2. Dependencies and Imports

The script relies on the following modules and custom types:

*   **`pydantic`**:
    *   `BaseModel`: The base class for creating data models.
    *   `conint`: A constrained integer type used for validation (e.g., `ge=1`, `le=100`).
    *   `Field`: A function used to provide extra validation, documentation, and default values for model attributes.
*   **`typing`**:
    *   `Optional`: Indicates that a value can be `None`.
    *   `Dict`: Type hint for dictionary.
    *   `List`: Type hint for list.
    *   `Union`: Indicates that a value can be one of several types.
*   **`datetime`**: Imported but not directly used within the provided snippet, typically used for handling date and time objects.
*   **`app.llms`**:
    *   `LLMConnectors`: An `Enum` (enumerated type) representing different Large Language Model connectors (e.g., OpenAI, Cohere). This is an application-specific import, implying the presence of an `llms.py` module in the `app` package.
    *   `LLMModels`: An `Enum` representing specific Large Language Model types (e.g., GPT-Mini, GPT-4). This is also an application-specific import.

### 3. Class Definitions and Functionality

This section details each `BaseModel` class, explaining its purpose, attributes, and any specific validation rules or default values.

#### 3.1. `SearchRequest`

**Purpose**:
`SearchRequest` represents the data model for an incoming search query. It defines all the parameters a client can provide to initiate a structured data retrieval request. This model is crucial for validating user input for search operations, ensuring data integrity and consistency before processing.

**Attributes**:

*   **`BasicQuery`** (`str`):
    *   **Description**: The primary textual search query provided by the user. This is typically the natural language input to be processed by the system.
*   **`SportCode`** (`str`):
    *   **Description**: A unique code identifying the specific sport to search within (e.g., "MFB" for Men's Football, "MBB" for Men's Basketball).
*   **`TeamCode`** (`Optional[str]`, default: `None`):
    *   **Description**: An optional identifier code for a specific team. If provided, the search will be filtered to data associated with this team.
*   **`AQLOnly`** (`Optional[bool]`, default: `False`):
    *   **Description**: A boolean flag. If `True`, the system will only generate and return the internal AQL (Analytical Query Language) representation of the query without executing it against the database. Useful for debugging or understanding query translation.
*   **`Entity`** (`str`):
    *   **Description**: The type of entity the search is focused on (e.g., "Player", "Team", "Game"). This guides the data retrieval logic to the correct dataset.
*   **`TimePeriod`** (`Optional[str]`, default: `"Game"`):
    *   **Description**: Specifies the time granularity for statistical aggregation (e.g., "Game", "Season", "Career"). Defaults to "Game" if not specified.
*   **`GamePeriod`** (`Optional[str]`, default: `None`):
    *   **Description**: A specific period within a game (e.g., "1st Quarter", "Half") to filter data by.
*   **`PageNumber`** (`conint(ge=1)`, default: `1`):
    *   **Description**: The current page number for paginating search results.
    *   **Validation**: Must be an integer greater than or equal to 1.
*   **`PageSize`** (`conint(ge=1, le=100)`, default: `10`):
    *   **Description**: The number of search results to return per page.
    *   **Validation**: Must be an integer between 1 and 100, inclusive.
*   **`Filters`** (`Dict[str, Union[str, List[str]]]`, default: `{}`):
    *   **Description**: A dictionary of additional filtering criteria to apply to the search results. Keys represent filter categories, and values can be a single string or a list of strings.
*   **`SortBy`** (`Optional[str]`, default: `None`):
    *   **Description**: The name of the field by which the results should be sorted.
*   **`SortOrder`** (`Optional[str]`, default: `None`):
    *   **Description**: The direction of sorting.
    *   **Validation**: If provided, must match the pattern `^(ASC|DESC)$` (i.e., either "ASC" for ascending or "DESC" for descending).
*   **`Llm`** (`LLMConnectors`, default: `LLMConnectors.OPENAI`):
    *   **Description**: Specifies which Language Model connector (e.g., OpenAI, Azure) to use for processing the query. Defaults to `OPENAI`.
*   **`LlmModel`** (`LLMModels`, default: `LLMModels.GPT_MINI`):
    *   **Description**: Specifies the particular language model (e.g., GPT-Mini, GPT-4) to be used. Defaults to `GPT_MINI`.
*   **`UseRAG`** (`bool`, default: `True`):
    *   **Description**: A boolean flag indicating whether to employ Retrieval-Augmented Generation (RAG) to shortlist relevant statistics before sending them to the LLM. This can improve the LLM's performance and accuracy by providing more focused context.
*   **`aql_output`** (`Optional[dict]`, default: `None`):
    *   **Description**: An optional field intended to hold the generated AQL query and its components internally after initial processing. This field is likely used for internal state management or passing information between processing steps rather than as direct user input.

#### 3.2. `ExportRequest`

**Purpose**:
`ExportRequest` defines the schema for requests aimed at exporting data, typically generated SQL query results. It encapsulates the necessary information to perform an export operation.

**Attributes**:

*   **`sql`** (`str`):
    *   **Description**: The SQL query string whose results are to be exported.
*   **`sport_code`** (`str`):
    *   **Description**: The code identifying the sport relevant to the exported data.
*   **`entity`** (`str`):
    *   **Description**: The type of entity (e.g., "Player", "Team") that the exported data pertains to.
*   **`stat_columns`** (`Optional[List[str]]`, default: `[]`):
    *   **Description**: An optional list of specific column names to include in the export. If empty or not provided, all available columns might be exported.
*   **`limit`** (`Optional[int]`, default: `0`):
    *   **Description**: An optional integer specifying the maximum number of rows to export. A value of `0` typically indicates no limit.
*   **`offset`** (`Optional[int]`, default: `0`):
    *   **Description**: An optional integer specifying the starting offset for the exported rows. A value of `0` indicates starting from the first row.

#### 3.3. `RecentQuery`

**Purpose**:
`RecentQuery` models a recent or example query, providing a structured way to store and retrieve query strings, potentially for display in a user interface (e.g., "Recent Searches" or "Popular Examples").

**Attributes**:

*   **`id`** (`int`):
    *   **Description**: A unique integer identifier for the query.
*   **`query`** (`str`):
    *   **Description**: The actual query string.
*   **`example`** (`Optional[bool]`, default: `False`):
    *   **Description**: A boolean flag indicating whether this query is an example query (`True`) or a user's recent query (`False`).

#### 3.4. `ExportLogRequest`

**Purpose**:
`ExportLogRequest` is designed for logging the details of data export operations. It captures key metrics about an export, which can be useful for auditing, monitoring, and debugging.

**Attributes**:

*   **`sql`** (`str`):
    *   **Description**: The exact SQL query that was executed and its results exported.
*   **`SportCode`** (`str`):
    *   **Description**: The code identifying the sport associated with the exported data (e.g., "MFB", "MBB").
*   **`Entity`** (`str`):
    *   **Description**: The type of entity whose data was exported (e.g., "Player", "Team").
*   **`RowCount`** (`int`):
    *   **Description**: The total number of rows that were successfully exported in the operation.

#### 3.5. `ConferenceData`

**Purpose**:
`ConferenceData` is a helper model used within `SearchResponse` to structure information about a sports conference, including the teams associated with it.

**Attributes**:

*   **`conference_short_name`** (`str`):
    *   **Description**: The short or abbreviated name of the conference (e.g., "SEC", "Big Ten").
*   **`teams_associated_with_conference`** (`List[str]`):
    *   **Description**: A list of names of teams that are part of this conference.

#### 3.6. `SearchResponse`

**Purpose**:
`SearchResponse` defines the comprehensive structure for the response returned by a search query. It includes the actual results, various metadata about the query, performance metrics, and optional aggregate information.

**Attributes**:

*   **`count`** (`int`):
    *   **Description**: The total number of results found that match the search criteria. This may be greater than the number of results returned in the `results` list if pagination is applied.
*   **`results`** (`List[dict]`):
    *   **Description**: A list of dictionaries, where each dictionary represents a single search result (e.g., a player's statistics, a team's record).
*   **`aql_output`** (`dict`):
    *   **Description**: A dictionary containing the generated AQL query and its intermediate components or parsed structure. This provides insight into how the natural language query was interpreted.
*   **`sql_query`** (`str`):
    *   **Description**: The final SQL query string that was generated from the AQL and executed against the database to retrieve the results.
*   **`inference_time`** (`float`):
    *   **Description**: The time, in seconds, taken for the system to infer and generate the AQL and SQL queries from the `BasicQuery`.
*   **`query_execution_time`** (`float`):
    *   **Description**: The total time, in seconds, spent executing the generated query against the underlying data store, encompassing database communication and result retrieval.
*   **`mongo_execution_time`** (`float`):
    *   **Description**: The time, in seconds, specifically spent executing queries against a MongoDB database, if used as part of the data retrieval process.
*   **`positions`** (`Optional[List[str]]`, default: `None`):
    *   **Description**: An optional list of distinct player positions (e.g., "QB", "WR") found within the returned `results`.
*   **`classes`** (`Optional[List[str]]`, default: `None`):
    *   **Description**: An optional list of distinct player classes or academic years (e.g., "Freshman", "Senior") found within the returned `results`.
*   **`teams`** (`Optional[List[str]]`, default: `None`):
    *   **Description**: An optional list of distinct team names found within the returned `results`.
*   **`conferences`** (`Optional[Dict[str, ConferenceData]]`, default: `None`):
    *   **Description**: An optional dictionary where keys are conference names (e.g., "SEC") and values are `ConferenceData` objects, providing details about conferences found in the results and their associated teams.
*   **`game_type`** (`Optional[List[str]]`, default: `None`):
    *   **Description**: An optional list of distinct game types (e.g., "Regular Season", "Playoff") found within the returned `results`.
*   **`period_type`** (`Optional[List[str]]`, default: `None`):
    *   **Description**: An optional list of distinct period types (e.g., "Quarter", "Half", "Overtime") found within the returned `results`.

### 4. Main Execution Flow (`if __name__ == "__main__":`)

The provided Python script defines only data models (Pydantic classes). It does not contain a `if __name__ == "__main__":` block or any executable code that runs when the script is invoked directly. Therefore, there is no main execution flow to document within this specific file. The classes defined here are intended to be imported and used by other parts of the application (e.g., FastAPI route handlers, service layers) for data validation and structuring.
