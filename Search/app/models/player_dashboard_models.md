# Documentation for `player_dashboard_models.py`

This document provides a comprehensive technical overview and line-by-line documentation for the provided Python script.

---

## 1. High-Level Overview

This Python script defines a collection of Pydantic models that serve as data transfer objects (DTOs) for various API requests and responses related to player profiles, game logs, records, highs, and streaks in sports. These models leverage Pydantic's robust data validation and serialization capabilities to ensure that incoming and outgoing data conforms to predefined schemas.

The script is structured to define distinct models for each API endpoint's request and response payloads, ensuring clear separation of concerns and type-safety. It uses advanced typing features from Python's `typing` module, such as `Literal` for enumerated values, `Optional` for nullable fields, and `Union` for fields that can accept multiple types.

**Primary Purpose:** To provide a well-defined, validated, and self-documenting schema for interacting with a sports statistics API, making it easier for client applications to send correct data and parse expected responses.

## 2. Dependencies and Imports

The script relies on the following Python modules:

*   **`pydantic`**: A data validation and settings management library, used here for defining data models.
    *   `BaseModel`: The base class for creating data models.
    *   `conint`: (Not explicitly used, but imported. Likely intended for constrained integers.)
    *   `Field`: Used for adding extra validation or metadata to model fields (not explicitly used here but commonly used with `pydantic`).
    *   `field_validator`: A decorator for custom field validation methods.
*   **`typing`**: Provides support for type hints.
    *   `Optional`: Indicates that a value can be either the specified type or `None`.
    *   `Dict`: Type hint for dictionaries.
    *   `List`: Type hint for lists.
    *   `Union`: Indicates that a value can be one of several specified types.
    *   `Any`: Indicates an unconstrained type.
    *   `Literal`: Specifies that a field's value must be exactly one of the provided string literals.
*   **`datetime`**: Provides classes for working with dates and times.
    *   `date`: Represents a date (year, month, day).

```python
from pydantic import BaseModel, conint, Field,field_validator
from typing import Optional, Dict, List, Union, Any,Literal
from datetime import date
```
*   **`from pydantic import BaseModel, conint, Field,field_validator`**: Imports necessary classes and decorators from the Pydantic library. `BaseModel` is the foundation for all data models. `field_validator` is used to define custom validation logic for model fields. `conint` and `Field` are imported but not directly used in the provided code snippet.
*   **`from typing import Optional, Dict, List, Union, Any,Literal`**: Imports advanced type hints from Python's `typing` module. These are crucial for defining precise data structures and enabling Pydantic's validation.
*   **`from datetime import date`**: Imports the `date` class from the standard `datetime` module, used for representing date values in some models.

## 3. Detailed Explanation of Models

This section details each Pydantic model defined in the script.

### 3.1. `PlayerProfileRequest`

```python
class PlayerProfileRequest(BaseModel):
    """
    Represents a request for player profile information.
    
    Attributes:
        PlayerName (str): Name of the player
        SportCode (str): Sport identifier code (e.g. "MFB" for football, "MBB" for basketball)
        TeamCode (str): Team identifier code
    """
    Player_id : str
    SportCode : Literal["MFB", "MBB","WBB"]
```
*   **`class PlayerProfileRequest(BaseModel):`**: Defines a Pydantic model named `PlayerProfileRequest` which inherits from `BaseModel`. This class represents the expected structure of a request body when fetching a player's profile.
*   **Docstring**: Provides a brief description of the model's purpose and its attributes. Note that the docstring's `PlayerName` and `TeamCode` attributes are not actually present in the model's definition, indicating a potential discrepancy or an older version of the docstring.
*   **`Player_id : str`**: A required field representing the unique identifier of the player, expected to be a string.
*   **`SportCode : Literal["MFB", "MBB","WBB"]`**: A required field representing the sport code. `Literal` ensures that the value must be one of the specified strings: "MFB" (Men's Football), "MBB" (Men's Basketball), or "WBB" (Women's Basketball).

### 3.2. `PlayerProfileResponse`

```python
class PlayerProfileResponse(BaseModel):
    """
    Represents a response from a player profile request.
    
    Attributes:
        PlayerName (str): Name of the player
        SportCode (str): Sport identifier code (e.g. "MFB" for football, "MBB" for basketball)
        TeamCode (str): Team identifier code
    """
    PlayerName : str
    SportCode : Literal["MFB", "MBB","WBB"]
    TeamName : str
    TeamCode : Union [str,None]
    PlayerPosition : Union[str, None]
    stats : List[Dict[str, Union[str, int, float,Any, None]]]
    stat_mapping : Dict[str, List[str]]
```
*   **`class PlayerProfileResponse(BaseModel):`**: Defines a Pydantic model named `PlayerProfileResponse` for the response body of a player profile request.
*   **Docstring**: Describes the model's purpose and its attributes. Similar to `PlayerProfileRequest`, the docstring might be slightly outdated as it doesn't fully capture all defined attributes.
*   **`PlayerName : str`**: The name of the player, expected to be a string.
*   **`SportCode : Literal["MFB", "MBB","WBB"]`**: The sport code, constrained to "MFB", "MBB", or "WBB".
*   **`TeamName : str`**: The name of the player's team, expected to be a string.
*   **`TeamCode : Union [str,None]`**: The team identifier code. `Union[str, None]` indicates that it can be either a string or `None` (i.e., it's an optional string).
*   **`PlayerPosition : Union[str, None]`**: The player's position. It can be a string or `None`.
*   **`stats : List[Dict[str, Union[str, int, float,Any, None]]]`**: A list where each element is a dictionary. The keys of these dictionaries are strings, and their values can be strings, integers, floats, any type, or `None`. This flexible type hint allows for various statistical data.
*   **`stat_mapping : Dict[str, List[str]]`**: A dictionary where keys are strings (e.g., category names) and values are lists of strings (e.g., individual statistic names within that category). This likely helps in interpreting the `stats` data.

### 3.3. `PlayerGameLogsRequest`

```python
class PlayerGameLogsRequest(BaseModel):
    """
    Represents a request for player game logs.
    
    Attributes:
        PlayerName (str): Name of the player
        SportCode (str): Sport identifier code (e.g. "MFB" for football, "MBB" for basketball)
        TeamCode (str): Team identifier code
    """
    Player_id : str
    SportCode : Literal["MFB", "MBB","WBB"]
    Season : Optional[int]
```
*   **`class PlayerGameLogsRequest(BaseModel):`**: Defines a Pydantic model for requesting player game logs.
*   **Docstring**: Describes the model's purpose. The attributes in the docstring (`PlayerName`, `TeamCode`) do not match the actual model definition.
*   **`Player_id : str`**: The unique identifier of the player, required as a string.
*   **`SportCode : Literal["MFB", "MBB","WBB"]`**: The sport code, restricted to "MFB", "MBB", or "WBB".
*   **`Season : Optional[int]`**: An optional field representing the specific season (e.g., year) for which to retrieve game logs. It can be an integer or `None`.

### 3.4. `PlayerGameLogsResponse`

```python
class PlayerGameLogsResponse(BaseModel):
    """
    Represents a response from a player game logs request.
    
    Attributes:
        PlayerGameLogs (List[Dict[str, Union[str, int, float,Any, None]]]): List of player game logs
    """
    PlayerName : str
    SportCode : Literal["MFB", "MBB","WBB"]
    TeamName : str
    TeamCode : str
    Seasons : List[int]
    PlayerPosition : str
    stats : List[Dict[str, Union[str, int, float,Any, None]]]
    stat_mapping : Dict[str, List[str]]
```
*   **`class PlayerGameLogsResponse(BaseModel):`**: Defines a Pydantic model for the response body of a player game logs request.
*   **Docstring**: Describes the model's purpose and its primary attribute.
*   **`PlayerName : str`**: The name of the player.
*   **`SportCode : Literal["MFB", "MBB","WBB"]`**: The sport code, restricted as before.
*   **`TeamName : str`**: The name of the player's team.
*   **`TeamCode : str`**: The team identifier code.
*   **`Seasons : List[int]`**: A list of integers representing all seasons for which logs are available or returned.
*   **`PlayerPosition : str`**: The player's position.
*   **`stats : List[Dict[str, Union[str, int, float,Any, None]]]`**: A list of dictionaries, where each dictionary likely represents a single game log with various statistics. The type hints allow for flexible data types within these statistics.
*   **`stat_mapping : Dict[str, List[str]]`**: A dictionary providing mapping or categorization for the statistics, similar to `PlayerProfileResponse`.

### 3.5. `PlayerRecordsRequest`

```python
class PlayerRecordsRequest(BaseModel):
    """
    Represents a request for player records.
    
    Attributes:
        PlayerName (str): Name of the player
        SportCode (str): Sport identifier code (e.g. "MFB" for football, "MBB" for basketball)
        TeamCode (str): Team identifier code
        StatPeriod (str): Statistic period
        RankThreshold (Optional[int]): Minimum rank threshold
    """
    Player_id : str
    SportCode : Literal["MFB", "MBB","WBB"]
    TeamCode : str
    StatPeriod: Literal["game","career", "season"] = "game"
    RankThreshold : Optional[int] = 100
```
*   **`class PlayerRecordsRequest(BaseModel):`**: Defines a Pydantic model for requesting player records (e.g., personal bests or top performances).
*   **Docstring**: Describes the model's purpose. The attributes in the docstring (`PlayerName`) do not fully align with the actual model.
*   **`Player_id : str`**: The unique identifier of the player, required.
*   **`SportCode : Literal["MFB", "MBB","WBB"]`**: The sport code, restricted as before.
*   **`TeamCode : str`**: The team identifier code, required.
*   **`StatPeriod: Literal["game","career", "season"] = "game"`**: A required field specifying the period for statistics. It must be "game", "career", or "season". It has a default value of "game".
*   **`RankThreshold : Optional[int] = 100`**: An optional integer field specifying a minimum rank threshold. If provided, records with a rank below this threshold might be filtered. It has a default value of `100`.

### 3.6. `PlayerRecord`

```python
class PlayerRecord(BaseModel):
    """Represents a single player performance record for a specific game."""
    stat: str
    value: float
    season: Optional[int] = None
    game_date: Optional[str] = None
    opponent_team_name: Optional[str] = None
    rank: int
    next_rank_value: Optional[float] = None
```
*   **`class PlayerRecord(BaseModel):`**: Defines a Pydantic model representing a single item within a list of player records. This nested model provides a structured way to return detailed record information.
*   **Docstring**: Clearly states its purpose as representing "a single player performance record for a specific game."
*   **`stat: str`**: The name or identifier of the statistic (e.g., "Points", "Rebounds").
*   **`value: float`**: The numerical value of the statistic for this record.
*   **`season: Optional[int] = None`**: The season in which the record occurred, optional, with a default of `None`.
*   **`game_date: Optional[str] = None`**: The date of the game when the record was achieved, optional, with a default of `None`. Expected to be a string (e.g., "YYYY-MM-DD").
*   **`opponent_team_name: Optional[str] = None`**: The name of the opponent team during that game, optional, with a default of `None`.
*   **`rank: int`**: The rank of this record (e.g., "1" for the top record, "2" for the second, etc.).
*   **`next_rank_value: Optional[float] = None`**: The value of the next ranked record, optional, with a default of `None`. This could be useful for context (e.g., "current record is 30 points, next best is 28").

### 3.7. `PlayerRecordsResponse`

```python
class PlayerRecordsResponse(BaseModel):
    """
    Represents a response from a player records request.
    
    Attributes:
        PlayerRecords (List[Dict[str, Union[str, int, float,Any, None]]]): List of player records
    """
    PlayerRecords : List[PlayerRecord]
    # TotalCount : int
```
*   **`class PlayerRecordsResponse(BaseModel):`**: Defines a Pydantic model for the response body of a player records request.
*   **Docstring**: Describes the model's purpose. The attribute type hint in the docstring (`List[Dict[str, Union[...]]]`) is generic, while the actual model uses the more specific `List[PlayerRecord]`.
*   **`PlayerRecords : List[PlayerRecord]`**: A list of `PlayerRecord` objects, where each object details a specific player performance record. This demonstrates the use of nested Pydantic models for structured responses.
*   **`# TotalCount : int`**: A commented-out field, suggesting it was either considered for inclusion and then removed, or is a placeholder for future implementation.

### 3.8. `PlayerHighsRequest`

```python
class PlayerHighsRequest(BaseModel):
    """
    Represents a request for player highs.
    
    Attributes:
        PlayerName (str): Name of the player
        SportCode (str): Sport identifier code (e.g. "MFB" for football, "MBB" for basketball)
        StatPeriod (str): Statistic period
        Limit (Optional[int]): Maximum number of records to return
        Offset (Optional[int]): Number of records to skip
    """
    Player_id : str
    SportCode : Literal["MFB", "MBB","WBB"]
    StatPeriod: Literal["career", "season"]
```
*   **`class PlayerHighsRequest(BaseModel):`**: Defines a Pydantic model for requesting player highs (e.g., top scores in a game, season, or career).
*   **Docstring**: Describes the model's purpose. The attributes in the docstring (`PlayerName`, `Limit`, `Offset`) do not match the actual model definition.
*   **`Player_id : str`**: The unique identifier of the player, required.
*   **`SportCode : Literal["MFB", "MBB","WBB"]`**: The sport code, restricted as before.
*   **`StatPeriod: Literal["career", "season"]`**: A required field specifying the period for the highs. It must be "career" or "season".

### 3.9. `PlayerHigh`

```python
class PlayerHigh(BaseModel):
    """Represents a single player performance high for a specific game."""
    stat: str
    value: float
    season: int
    game_date: str
    opponent_team_name: str
    stat_rank: int
```
*   **`class PlayerHigh(BaseModel):`**: Defines a Pydantic model representing a single item within a list of player highs.
*   **Docstring**: States its purpose as representing "a single player performance high for a specific game."
*   **`stat: str`**: The name or identifier of the statistic.
*   **`value: float`**: The numerical value of the statistic for this high.
*   **`season: int`**: The season in which the high occurred.
*   **`game_date: str`**: The date of the game when the high was achieved, expected as a string.
*   **`opponent_team_name: str`**: The name of the opponent team during that game.
*   **`stat_rank: int`**: The rank of this high within its category.

### 3.10. `PlayerHighsResponse`

```python
class PlayerHighsResponse(BaseModel):
    """
    Represents a response from a player highs request.
    
    Attributes:
        PlayerHighs (List[Dict[str, Union[str, int, float,Any, None]]]): List of player highs
    """
    PlayerHighs : List[PlayerHigh]
    # TotalCount : int
```
*   **`class PlayerHighsResponse(BaseModel):`**: Defines a Pydantic model for the response body of a player highs request.
*   **Docstring**: Describes the model's purpose. Similar to `PlayerRecordsResponse`, the docstring's attribute type hint is generic, while the actual model uses the specific `List[PlayerHigh]`.
*   **`PlayerHighs : List[PlayerHigh]`**: A list of `PlayerHigh` objects, each detailing a specific player high performance.
*   **`# TotalCount : int`**: A commented-out field, similar to `PlayerRecordsResponse`.

### 3.11. `PlayerStreaksRequest`

```python
class PlayerStreaksRequest(BaseModel):
    """
    Represents a request for player stats.
    
    Attributes:
        PlayerId (str): Name of the player
        SportCode (str): Sport identifier code (e.g. "MFB" for football, "MBB" for basketball)
        TeamCode (str): Team identifier code
        stat (str): Statistic Name 
        stat_value (int): Statistic Value
        streak_length (int): Streak Length
    """
    Player_id : str
    SportCode : Literal["MFB", "MBB","WBB"]
    TeamCode : str
    stat : str
    stat_value : int
    streak_length : int
    start_date : Optional[str] = None
    end_date : Optional[str] = None
```
*   **`class PlayerStreaksRequest(BaseModel):`**: Defines a Pydantic model for requesting player streaks (e.g., consecutive games with a certain stat value).
*   **Docstring**: Describes the model's purpose. The `PlayerId` attribute in the docstring differs slightly from the `Player_id` in the model definition.
*   **`Player_id : str`**: The unique identifier of the player, required.
*   **`SportCode : Literal["MFB", "MBB","WBB"]`**: The sport code, restricted as before.
*   **`TeamCode : str`**: The team identifier code, required.
*   **`stat : str`**: The name of the statistic to check for streaks (e.g., "Points", "Assists").
*   **`stat_value : int`**: The threshold value for the statistic that defines a streak (e.g., `10` for "10+ points").
*   **`streak_length : int`**: The minimum length of the streak to be considered (e.g., `3` for a 3-game streak).
*   **`start_date : Optional[str] = None`**: An optional start date for the period to search for streaks, expected as a string (YYYY-MM-DD), with a default of `None`.
*   **`end_date : Optional[str] = None`**: An optional end date for the period to search for streaks, expected as a string (YYYY-MM-DD), with a default of `None`.

### 3.12. `PlayerStreaks`

```python
class PlayerStreaks(BaseModel):
    """
    Represents a response containing player statistics, including streak information.

    Attributes:
        streak_length (int): Length of the streak.
        start_game_date (date): Date of the first game in the streak (YYYY-MM-DD).
        start_opponent (str): Name of the opponent in the first game of the streak.
        end_game_date (date): Date of the last game in the streak (YYYY-MM-DD).
        end_opponent (str): Name of the opponent in the last game of the streak.
    """
    player_name: str
    team_name: str
    streak_length: int
    start_game_date: date
    start_opponent: str
    end_game_date: date
    end_opponent: str

    @field_validator('start_game_date', 'end_game_date')
    def parse_date(cls, value):
        if isinstance(value, str):
            return date.fromisoformat(value)
        return value
```
*   **`class PlayerStreaks(BaseModel):`**: Defines a Pydantic model representing a single detected streak for a player.
*   **Docstring**: Clearly describes the model's purpose and its attributes.
*   **`player_name: str`**: The name of the player.
*   **`team_name: str`**: The name of the player's team.
*   **`streak_length: int`**: The number of consecutive games in the streak.
*   **`start_game_date: date`**: The date of the first game in the streak. This field is explicitly typed as a `datetime.date` object.
*   **`start_opponent: str`**: The name of the opponent team in the first game of the streak.
*   **`end_game_date: date`**: The date of the last game in the streak. Also typed as a `datetime.date` object.
*   **`end_opponent: str`**: The name of the opponent team in the last game of the streak.
*   **`@field_validator('start_game_date', 'end_game_date')`**: This is a Pydantic decorator that applies the `parse_date` method as a validator to both `start_game_date` and `end_game_date` fields. It ensures that these fields are correctly converted to `datetime.date` objects.
*   **`def parse_date(cls, value):`**: This class method performs the custom validation.
    *   **`if isinstance(value, str):`**: Checks if the incoming `value` for the date field is a string.
    *   **`return date.fromisoformat(value)`**: If it's a string, it attempts to parse it using `date.fromisoformat()`. This method is designed to parse date strings in the `YYYY-MM-DD` format.
    *   **`return value`**: If the value is not a string (e.g., already a `date` object or `None`), it is returned as is, allowing Pydantic's default validation to handle it or assuming it's already in the correct format.

### 3.13. `PlayerStreaksResponse`

```python
class PlayerStreaksResponse(BaseModel):
    """
    Represents a response from a player streaks request.
    
    Attributes:
        PlayerStreaks (List[Dict[str, Union[str, int, float,Any, None]]]): List of player streaks
    """
    PlayerStreaks : List[PlayerStreaks]
    TotalCount : int
```
*   **`class PlayerStreaksResponse(BaseModel):`**: Defines a Pydantic model for the response body of a player streaks request.
*   **Docstring**: Describes the model's purpose. The attribute type hint in the docstring is generic, while the actual model uses the specific `List[PlayerStreaks]`.
*   **`PlayerStreaks : List[PlayerStreaks]`**: A list of `PlayerStreaks` objects, where each object details a detected streak.
*   **`TotalCount : int`**: The total number of streaks found, expected as an integer.

### 3.14. `MappingRequest`

```python
class MappingRequest(BaseModel):
    SportCode: Literal["MFB", "MBB", "WBB"]
    playerposition: str
```
*   **`class MappingRequest(BaseModel):`**: Defines a Pydantic model for a request related to mapping information, likely for player positions within specific sports.
*   **`SportCode: Literal["MFB", "MBB", "WBB"]`**: The sport code, restricted as before. Required.
*   **`playerposition: str`**: The player's position, expected as a string. Required.

## 4. Main Execution Flow

The provided script consists solely of Pydantic model definitions. There is no `if __name__ == "__main__":` block or any other executable code within this file. Therefore, this script is designed to be imported as a module into other Python applications (e.g., a FastAPI application, a data processing script) to provide the data structures for request and response validation and serialization.

## 5. Important Implementation Details, Dependencies, and Configuration

*   **Pydantic for Data Validation**: The core of this script is its reliance on Pydantic. Pydantic automatically validates the types and constraints of incoming data when an instance of a `BaseModel` is created. If data does not conform to the schema, Pydantic raises validation errors, ensuring data integrity.
*   **Type Hinting**: Extensive use of Python's type hints (`typing` module) makes the code self-documenting, improves readability, and enables static analysis tools (like MyPy) to catch potential type mismatches before runtime.
*   **`Literal` for Enumerated Values**: The `Literal` type is strategically used for `SportCode` and `StatPeriod` fields. This enforces that these fields can only accept a predefined set of string values, preventing invalid inputs and improving API robustness.
*   **`Optional` and `Union` for Nullability**: `Optional[Type]` (which is syntactic sugar for `Union[Type, None]`) is used to clearly mark fields that can be absent or explicitly `None`. This is crucial for handling flexible API payloads where some parameters might not always be required.
*   **Nested Models**: Models like `PlayerRecordsResponse`, `PlayerHighsResponse`, and `PlayerStreaks` demonstrate the use of nested Pydantic models. This allows for complex, structured data to be represented in a clear and validated manner, providing strong typing for elements within lists or sub-objects.
*   **Custom Field Validators**: The `parse_date` validator in `PlayerStreaks` shows how to implement custom logic for type conversion and validation. This is particularly useful when data might arrive in one format (e.g., a date string) but needs to be handled as a different type internally (e.g., a `datetime.date` object).
*   **Readability and Maintainability**: By separating concerns into distinct request and response models, and by leveraging Pydantic, the code base becomes highly readable, maintainable, and less prone to common data-related bugs. Any changes to the API schema can be quickly reflected and validated within these models.
