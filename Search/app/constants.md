# Documentation for `constants.py`

This document provides a comprehensive technical overview and line-by-line explanation of the provided Python script.

---

## Technical Documentation: `constants.py`

This Python script (`constants.py`, based on typical naming conventions for such a file) serves as a central repository for defining application-wide constants, mappings, and enumerations. Its primary purpose is to enhance code readability, maintainability, and consistency by consolidating various fixed values, such as database table names, query qualifiers, sorting fields, user authentication fields, and configuration file paths, into well-structured Python objects.

The script extensively utilizes Python's `enum.Enum` class to create distinct sets of related constants, providing type safety and preventing errors that can arise from using magic strings throughout an application. This structure is particularly beneficial for applications interacting with databases, external APIs (like LLMs), and complex data filtering/sorting logic.

### Dependencies

*   `enum.Enum`: From Python's standard library, used for creating enumerable sets of symbolic names.

### `QUALIFIERS_MAPPING` Dictionary

```python
QUALIFIERS_MAPPING = {
    "playerName" : "player_name",
    "playerClass": "player_class",
    "teamName" : "team_name",
    "teamConferenceName": "conference_name",
    "opponentTeamName": "opponent_team_name",
    "opponentConferenceName": "opponent_conference_name",
    "gameType": "g_round_description",
    "position" : "position",
    "season" : "season",
    "teamCode" : "team_code"
}
```

*   **Type**: `dict`
*   **Purpose**: This dictionary defines a mapping between user-facing or API-facing qualifier names (keys) and their corresponding database column names (values). It acts as a translation layer, allowing the application to present standardized, often camelCase, names externally while using the snake_case names typically found in a relational database for internal queries.
*   **Structure**:
    *   Keys (e.g., `"playerName"`, `"playerClass"`): Represent the canonical names used in the application's logic or external interfaces (e.g., API requests).
    *   Values (e.g., `"player_name"`, `"player_class"`): Represent the actual column names in the underlying database tables that these qualifiers correspond to.

### `Qualifiers` Enum

```python
class Qualifiers(Enum):
    """Enumeration of valid qualifier fields used for filtering queries.
    
    These qualifiers are used to filter and constrain database queries based on 
    player, team, and game attributes.
    """
    PLAYER_NAME = "playerName"
    PLAYER_CLASS = "playerClass" 
    TEAM_NAME = "teamName"
    TEAM_CONFERENCE_NAME = "teamConferenceName"
    OPPONENT_TEAM_NAME = "opponentTeamName"
    POSITION = "position"
    OPPONENT_CONFERENCE_NAME = "opponentConferenceName"
    GAME_TYPE = "gameType"
    SEASON = "season"
    TEAM_CODE = "teamCode"
```

*   **Type**: `enum.Enum`
*   **Purpose**: This enumeration provides a standardized, type-safe list of all valid fields that can be used as *qualifiers* to filter data in database queries. These are the application-level (API-facing) names, consistent with the keys defined in `QUALIFIERS_MAPPING`. By using this enum, developers can avoid typos when specifying filter criteria.
*   **Members**: Each member represents a specific attribute that can be used for filtering:
    *   `PLAYER_NAME`: Filters by the player's name.
    *   `PLAYER_CLASS`: Filters by the player's academic class (e.g., Freshman, Senior).
    *   `TEAM_NAME`: Filters by the team's name.
    *   `TEAM_CONFERENCE_NAME`: Filters by the team's conference.
    *   `OPPONENT_TEAM_NAME`: Filters by the opponent team's name.
    *   `POSITION`: Filters by the player's position.
    *   `OPPONENT_CONFERENCE_NAME`: Filters by the opponent's conference.
    *   `GAME_TYPE`: Filters by the type of game (e.g., regular season, playoff).
    *   `SEASON`: Filters by the specific season.
    *   `TEAM_CODE`: Filters by a unique team identifier.

### `SortingFields` Enum

```python
class SortingFields(Enum):
    """Enumeration of valid fields that can be used for sorting query results.
    
    These fields represent the available options for ordering and sorting the 
    data returned from database queries.
    """
    PLAYER_NAME = "playerName"
    TEAM_NAME = "teamname"
    SEASON = "season"
    POSITION = "position"
    CONDITIONS = "conditions"
    QUALIFIERS = "qualifiers"
    GAMEDATE = "game_date"
    STREAKLENGTH = "streak_length"
    PLAYERCLASS = "playerClass"
    OPPONENTTEAMNAME = "opponentTeamName"
    QUERYTYPE = "query_type"
    PERIODS = "periods"
    PERIOD_TYPE = "period_type"
```

*   **Type**: `enum.Enum`
*   **Purpose**: Defines the set of fields that can be used to sort the results of data queries. This ensures that sorting operations are performed on recognized and supported attributes, improving data retrieval consistency.
*   **Members**:
    *   `PLAYER_NAME`, `TEAM_NAME`, `SEASON`, `POSITION`, `PLAYERCLASS`, `OPPONENTTEAMNAME`: Common attributes for sorting players, teams, or game data.
    *   `CONDITIONS`: Potentially a field related to game conditions or specific criteria.
    *   `QUALIFIERS`: Refers to sorting by the qualifier fields themselves (less common for direct sorting, possibly indicating an internal identifier or grouping).
    *   `GAMEDATE`: Sorts by the date of the game.
    *   `STREAKLENGTH`: Sorts by the length of a statistical streak.
    *   `QUERYTYPE`: Sorts by the type of query that generated the data.
    *   `PERIODS`, `PERIOD_TYPE`: Relates to sorting by game periods or types of periods.

### `DatabaseTables` Enum

```python
class DatabaseTables(Enum):
    """Enumeration of database table names used in the application.
    
    Defines the canonical names of tables storing player and team statistics.
    """
    PLAYER_STATS = "player_game_statistics"
    TEAM_STATS = "team_game_statistics"
```

*   **Type**: `enum.Enum`
*   **Purpose**: Stores the canonical names of the primary database tables used by the application, primarily for storing game-level statistics. Using an enum here centralizes table names, making database interactions less error-prone and easier to manage if table names change.
*   **Members**:
    *   `PLAYER_STATS`: The table storing individual player game statistics.
    *   `TEAM_STATS`: The table storing team-level game statistics.

### `SortOrders` Enum

```python
class SortOrders(Enum):
    """Enumeration of valid sort orders for query results.
    
    Defines ascending and descending sort order options.
    """
    ASC = "ASC"
    DESC = "DESC"
```

*   **Type**: `enum.Enum`
*   **Purpose**: Specifies the standard options for ordering query results: ascending or descending. This enum ensures consistency when constructing SQL queries or other data ordering operations.
*   **Members**:
    *   `ASC`: Ascending order.
    *   `DESC`: Descending order.

### `QueryTypes` Enum

```python
class QueryTypes(Enum):
    """Enumeration of supported query types for statistical analysis.
    
    Defines the different types of statistical queries that can be performed:
    - basic: Standard statistical queries
    - min: Minimum value queries
    - max: Maximum value queries
    - ltw: Last time when queries
    - streak: Streak analysis queries
    """
    BASIC = "basic"
    MIN = "min"
    MAX = "max"
    LTW = "ltw"
    STREAK = "streak"
```

*   **Type**: `enum.Enum`
*   **Purpose**: Categorizes the different types of statistical analysis queries supported by the application. This helps in dispatching requests to appropriate data processing logic.
*   **Members**:
    *   `BASIC`: Represents standard, straightforward statistical queries.
    *   `MIN`: Queries designed to find minimum values for a statistic.
    *   `MAX`: Queries designed to find maximum values for a statistic.
    *   `LTW`: "Last Time When" queries, designed to find the most recent occurrence of a specific event or statistical threshold.
    *   `STREAK`: Queries for analyzing streaks (e.g., winning streaks, scoring streaks).

### `PlayerClass` Enum

```python
class PlayerClass(Enum):
    """Enumeration of valid NCAA player classification years.
    
    Defines the standard academic classifications for college athletes.
    """
    FRESHMAN = 'FRESHMAN'
    JUNIOR = 'JUNIOR'
    SENIOR = 'SENIOR'
    SOPHOMORE = 'SOPHOMORE'
```

*   **Type**: `enum.Enum`
*   **Purpose**: Provides a standardized list of NCAA (National Collegiate Athletic Association) academic classifications for college athletes. This is useful for filtering or categorizing players.
*   **Members**:
    *   `FRESHMAN`, `JUNIOR`, `SENIOR`, `SOPHOMORE`: Standard academic years for undergraduate students.

### `Positions` Enum

```python
class Positions(Enum):
    """Enumeration of valid player positions across supported sports.
    
    Includes positions for:
    - Football (FB, QB, WR, etc.)
    - Men's Basketball (C, F, G)
    """
    # Football positions
    CB = 'CB'  # Cornerback
    DB = 'DB'  # Defensive Back
    DE = 'DE'  # Defensive End
    DL = 'DL'  # Defensive Line
    DT = 'DT'  # Defensive Tackle
    FB = 'FB'  # Fullback
    K = 'K'    # Kicker
    LB = 'LB'  # Linebacker
    LS = 'LS'  # Long Snapper
    OC = 'OC'  # Center
    OG = 'OG'  # Offensive Guard
    OL = 'OL'  # Offensive Line
    OT = 'OT'  # Offensive Tackle
    P = 'P'    # Punter
    QB = 'QB'  # Quarterback
    RB = 'RB'  # Running Back
    S = 'S'    # Safety
    TE = 'TE'  # Tight End
    WR = 'WR'  # Wide Receiver

    # Men's Basketball positions
    C = 'C'    # Center
    F = 'F'    # Forward
    G = 'G'    # Guard
```

*   **Type**: `enum.Enum`
*   **Purpose**: Lists a comprehensive set of valid player positions across different sports supported by the application. This ensures consistent representation of player positions, which is critical for filtering, analysis, and data entry.
*   **Members**: Includes a detailed list of positions, separated by sport for clarity:
    *   **Football Positions**: `CB` (Cornerback), `DB` (Defensive Back), `DE` (Defensive End), `DL` (Defensive Line), `DT` (Defensive Tackle), `FB` (Fullback), `K` (Kicker), `LB` (Linebacker), `LS` (Long Snapper), `OC` (Center), `OG` (Offensive Guard), `OL` (Offensive Line), `OT` (Offensive Tackle), `P` (Punter), `QB` (Quarterback), `RB` (Running Back), `S` (Safety), `TE` (Tight End), `WR` (Wide Receiver).
    *   **Men's Basketball Positions**: `C` (Center), `F` (Forward), `G` (Guard).

### `UserDataAuthentication` Enum

```python
class UserDataAuthentication(Enum):
    """Enumeration of authentication and user data fields.
    
    Defines the fields used for user authentication, session management,
    and user profile data.
    """
    EMAIL = "email"
    TEAM = "team"
    SPORTS = "sports"
    USERNAME = "username"
    ISACTIVE = "isActive"
    ISADMIN = "isAdmin"
    ISAPPROVED = "isApproved"
    HASHEDPWD = "hashedPwd"
    USERID = "userid"
    ACCESSTOKEN = "access_token"
    REFRESHTOKEN = "refresh_token"
    TOKENTYPE = "token_type"
    BEARER = "bearer"
    TERMSAGREED = "termsAgreed"
```

*   **Type**: `enum.Enum`
*   **Purpose**: Defines the various fields and terms associated with user management, authentication, and authorization within the application. This centralizes names for user attributes, database columns, and token types.
*   **Members**:
    *   `EMAIL`, `TEAM`, `SPORTS`, `USERNAME`: Standard user profile attributes.
    *   `ISACTIVE`, `ISADMIN`, `ISAPPROVED`: Boolean flags for user status and roles.
    *   `HASHEDPWD`: Field for storing hashed passwords.
    *   `USERID`: Unique identifier for a user.
    *   `ACCESSTOKEN`, `REFRESHTOKEN`, `TOKENTYPE`, `BEARER`: Fields and types related to OAuth2/JWT token-based authentication.
    *   `TERMSAGREED`: Indicates whether the user has agreed to terms and conditions.

### `MappingsHandlerTerms` Enum

```python
class MappingsHandlerTerms(Enum):
    """Enumeration of mapping configuration file names and paths.
    
    Defines the standard file names and paths for various mapping configurations
    used throughout the application.
    """
    TABLE_MAPPING = "table_mapping.json"
    FIELDS_MAPPING = "fields_mapping.json"
    FILTER_MAPPINGS = "filter_mappings.json"
    POSITION_MAPPINGS = "pos_mapping.json"
    METADATA_MAPPINGS = "metadata_mapping.json"
    QUERY_SAMPLES = "query_samples"
    STAT_MAPPING_FOLDER = "stat_mapping"
    CONFERENCE_MAPPING_FOLDER = "conference_mapping"
    DIVISION_MAPPINGS = "division_mapping.json"
    DATA_CONFIG = "data_config"
    MAPPINGS_ROOT = "mappings"
```

*   **Type**: `enum.Enum`
*   **Purpose**: Provides a centralized list of file names, folder names, and root paths for various configuration and mapping files used by the application. This standardizes file access and helps prevent hardcoding paths.
*   **Members**:
    *   `TABLE_MAPPING`, `FIELDS_MAPPING`, `FILTER_MAPPINGS`, `POSITION_MAPPINGS`, `METADATA_MAPPINGS`, `DIVISION_MAPPINGS`: Specific JSON file names for different types of data mappings.
    *   `QUERY_SAMPLES`: Likely a file or directory name for example queries.
    *   `STAT_MAPPING_FOLDER`, `CONFERENCE_MAPPING_FOLDER`: Names of directories containing further mapping configurations.
    *   `DATA_CONFIG`: Refers to data configuration files or folders.
    *   `MAPPINGS_ROOT`: The root directory where mapping files are located.

### `General` Enum

```python
class General(Enum):
    """Enumeration of general purpose constants used across the application.
    
    Defines common terms and fields used in various parts of the application.
    """
    STAT = "stat"
    DESCRIPTION = "description"
    STAT_PERIOD = "stat_period"
```

*   **Type**: `enum.Enum`
*   **Purpose**: Defines common, general-purpose terms or field names that are used across different modules or components of the application, ensuring consistency in naming conventions.
*   **Members**:
    *   `STAT`: Refers to a specific statistic.
    *   `DESCRIPTION`: Refers to a descriptive text.
    *   `STAT_PERIOD`: Refers to the time period over which a statistic is measured.

### `LLMModels` Enum

```python
class LLMModels(Enum):
    """Enumeration of supported Language Learning Models and their configurations.
    
    Defines available LLM models and their associated API keys across different providers:
    - OpenAI models (GPT-4)
    - Google models (Gemini)
    - Meta models (Llama)
    - Other providers (Groq, Gemma)
    """
    BASE = "base"
    GPT4 = "gpt-4o"
    GPT_MINI = "gpt-4o-mini"
    OPENAI_API_KEY = "OPENAI_API_KEY"
    GEMINI_FLASH = "gemini-2.0-flash"      
    GEMINI_FLASH_LITE = "gemini-2.0-flash-lite"
    GEMINI_API_KEY = "GEMINI_API_KEY"
    LLAMA_1 = "llama-3.2-1b-preview"
    LLAMA_8 = "llama3-8b-8192"
    LLAMA_70 = "llama-3.3-70b-versatile"
    GROQ_API_KEY = "GROQ_API_KEY"
    GEMMA = "gemma2-9b-it"
    GPT_O4_MINI = "o4-mini-2025-04-16"
    GPT4_1_MINI = "gpt-4.1-mini"
    GPT4_1 = "gpt-4.1"
    # GEMINI_2_5_FLASH = "gemini-2.5-flash-preview-05-20" # Commented out, but indicates a previously considered model.
    GEMINI_2_5_FLASH_LITE = "gemini-2.5-flash-lite-preview-06-17"
    GEMINI_2_5_FLASH="gemini-2.5-flash"
```

*   **Type**: `enum.Enum`
*   **Purpose**: Centralizes the names and identifiers for various Language Learning Models (LLMs) and their corresponding API key environment variable names. This enum is crucial for any part of the application that interacts with AI models, enabling easy switching between models and providers.
*   **Members**:
    *   `BASE`: A generic "base" identifier, possibly for a default model or a configuration root.
    *   **OpenAI Models**: `GPT4`, `GPT_MINI` (referring to `gpt-4o` and `gpt-4o-mini` respectively), `GPT_O4_MINI`, `GPT4_1_MINI`, `GPT4_1` (indicating specific versions or aliases). `OPENAI_API_KEY` specifies the environment variable name for the API key.
    *   **Google Gemini Models**: `GEMINI_FLASH`, `GEMINI_FLASH_LITE`, `GEMINI_2_5_FLASH_LITE`, `GEMINI_2_5_FLASH` (referring to various versions of Gemini models, particularly flash versions for speed). `GEMINI_API_KEY` specifies the environment variable name for the API key.
    *   **Meta Llama Models**: `LLAMA_1`, `LLAMA_8`, `LLAMA_70` (referring to different parameter counts or versions of Llama models).
    *   **Other Models/Providers**: `GROQ_API_KEY` (indicating a key for Groq's inference platform), `GEMMA` (referring to Google's Gemma model).
*   **Note**: The commented-out `GEMINI_2_5_FLASH` entry suggests an iterative development process where models are added, removed, or updated.

### `PostgreSQL` Enum

```python
class PostgreSQL(Enum):
    """Enumeration of PostgreSQL database configuration environment variables.
    
    Defines the expected environment variables needed for PostgreSQL database connection.
    """
    PG_DB = "PG_DB"           # Database name
    PG_USER = "PG_USER"       # Database user
    PG_PASSWORD = "PG_PASSWORD"# Database password
    PG_HOST = "PG_HOST"       # Database host
    PG_PORT = "PG_PORT"       # Database port
```

*   **Type**: `enum.Enum`
*   **Purpose**: Defines the standard environment variable names expected for configuring a PostgreSQL database connection. This promotes secure and flexible configuration by discouraging hardcoding sensitive database credentials directly into the application code.
*   **Members**:
    *   `PG_DB`: Environment variable for the PostgreSQL database name.
    *   `PG_USER`: Environment variable for the database username.
    *   `PG_PASSWORD`: Environment variable for the database password.
    *   `PG_HOST`: Environment variable for the database host address.
    *   `PG_PORT`: Environment variable for the database port number.

### Main Execution Flow

This script does not contain a `__main__` block and is not intended for direct execution. Its sole purpose is to define constants and enumerations for import and use by other modules within a larger Python application. It acts as a configuration and definition file.
