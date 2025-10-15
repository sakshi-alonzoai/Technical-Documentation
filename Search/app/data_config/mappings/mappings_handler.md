# Documentation for `mappings_handler.py`

This document provides comprehensive technical documentation for the `MappingsHandler` Python class, detailing its purpose, implementation, and usage.

---

## Technical Documentation: `MappingsHandler`

### 1. High-Level Overview

The `MappingsHandler` class is designed to centralize and manage the retrieval of various configuration mappings required by an application. These mappings typically include database table names, field specifications for queries, filter criteria, position mappings, and more. All mappings are stored in structured JSON files within a designated `data_config/mappings` directory.

The primary goal of this class is to provide an efficient and consistent interface for accessing these configuration data points. It preloads frequently used static mappings into memory during initialization to minimize disk I/O and improve performance for subsequent requests. For dynamic or less frequently accessed mappings (like `stat_mapping` or `conference_mapping`), it loads them on demand.

### 2. Dependencies

The script relies on the following standard Python modules and a custom constant definition:

*   **`os`**: Provides a way of using operating system dependent functionality, such as navigating file paths.
*   **`json`**: Enables working with JSON data, specifically parsing JSON files into Python dictionaries.
*   **`app.constants.MappingsHandlerTerms`**: A custom enumeration (likely an `Enum` class) that defines constant strings representing directory names, file names, and other terms used for constructing file paths and accessing mapping data.

### 3. Class: `MappingsHandler`

The `MappingsHandler` class encapsulates all logic related to locating, loading, and providing access to application mappings.

```python
class MappingsHandler:
    """
    Handles fetching of different mappings (table names, field mappings, etc.)
    from JSON files stored in the `data_config/mappings` directory.
    """
```

**Class Docstring**: This docstring clearly states the class's purpose: to fetch various mappings from JSON files located in the `data_config/mappings` directory.

#### 3.1. `__init__` Method

The constructor of the `MappingsHandler` class is responsible for initializing directory paths and preloading core mapping files into memory.

```python
    def __init__(self):
        """
        Initializes the MappingsHandler, preloading mappings from JSON files 
        to avoid repeated file I/O operations.
        """
        self.base_dir = os.path.dirname(os.path.dirname(os.path.dirname(__file__))) 
        self.mappings_dir = os.path.join(self.base_dir, MappingsHandlerTerms.DATA_CONFIG.value, MappingsHandlerTerms.MAPPINGS_ROOT.value)
        self.stat_mapping_dir = os.path.join(self.mappings_dir, MappingsHandlerTerms.STAT_MAPPING_FOLDER.value)  
        self.conference_mapping_dir = os.path.join(self.mappings_dir, MappingsHandlerTerms.CONFERENCE_MAPPING_FOLDER.value)  
        self.query_samples_dir = os.path.join(self.base_dir, MappingsHandlerTerms.DATA_CONFIG.value, MappingsHandlerTerms.QUERY_SAMPLES.value)  

        # Preload mappings
        self.table_mapping = self._load_json(MappingsHandlerTerms.TABLE_MAPPING.value)
        self.fields_mapping = self._load_json(MappingsHandlerTerms.FIELDS_MAPPING.value)
        self.filter_mappings = self._load_json(MappingsHandlerTerms.FILTER_MAPPINGS.value)
        self.position_mappings = self._load_json(MappingsHandlerTerms.POSITION_MAPPINGS.value)
        self.metadata_mappings = self._load_json(MappingsHandlerTerms.METADATA_MAPPINGS.value)
        self.division_mappings = self._load_json(MappingsHandlerTerms.DIVISION_MAPPINGS.value)
```

**Method Docstring**: Explains that the initialization preloads mappings to enhance performance by reducing repeated file I/O.

**Implementation Details**:

*   `self.base_dir = os.path.dirname(os.path.dirname(os.path.dirname(__file__)))`: This line calculates the project's base directory.
    *   `__file__` refers to the path of the current script (`MappingsHandler.py`).
    *   `os.path.dirname(__file__)` gets the directory containing the current script (e.g., `.../app/`).
    *   `os.path.dirname(...)` again moves up one level (e.g., `.../`).
    *   `os.path.dirname(...)` a third time moves up to the root of the project structure, assuming the script is nested at least three levels deep from the root (e.g., `project_root/app/module/MappingsHandler.py`). This establishes a stable reference point for all other paths.
*   `self.mappings_dir = os.path.join(self.base_dir, MappingsHandlerTerms.DATA_CONFIG.value, MappingsHandlerTerms.MAPPINGS_ROOT.value)`: Constructs the path to the main directory where static mapping JSON files are expected. It uses `os.path.join` for platform-independent path construction. `MappingsHandlerTerms.DATA_CONFIG.value` likely refers to a folder like `data_config`, and `MappingsHandlerTerms.MAPPINGS_ROOT.value` to a subfolder like `mappings`.
*   `self.stat_mapping_dir = os.path.join(self.mappings_dir, MappingsHandlerTerms.STAT_MAPPING_FOLDER.value)`: Defines the path to a specific subdirectory for statistical mappings, e.g., `data_config/mappings/stat_mappings`.
*   `self.conference_mapping_dir = os.path.join(self.mappings_dir, MappingsHandlerTerms.CONFERENCE_MAPPING_FOLDER.value)`: Defines the path to a specific subdirectory for conference mappings, e.g., `data_config/mappings/conference_mappings`.
*   `self.query_samples_dir = os.path.join(self.base_dir, MappingsHandlerTerms.DATA_CONFIG.value, MappingsHandlerTerms.QUERY_SAMPLES.value)`: Defines the path to the directory containing query samples, e.g., `data_config/query_samples`. Note that this path starts from `self.base_dir` and includes `DATA_CONFIG` and `QUERY_SAMPLES`, implying query samples are stored parallel to the `mappings` root within `data_config`.

**Preloading Mappings**:
The subsequent lines preload several key mapping files by calling the private `_load_json` method:
*   `self.table_mapping`: Stores mappings for database table names.
*   `self.fields_mapping`: Stores mappings for fields required in SQL queries.
*   `self.filter_mappings`: Stores mappings related to data filtering options.
*   `self.position_mappings`: Stores mappings for player positions.
*   `self.metadata_mappings`: Stores general metadata mappings.
*   `self.division_mappings`: Stores mappings related to divisions.

These preloaded attributes are made available as instance variables, `self.table_mapping`, `self.fields_mapping`, etc., for quick access by other methods.

#### 3.2. `_load_json` Method (Private)

A private helper method for loading JSON files from disk.

```python
    def _load_json(self, file_name: str) -> dict:
        """
        Private method to load a JSON file from a given file path.

        Args:
            file_name (str): The name of the JSON file.

        Returns:
            dict: Parsed JSON data as a dictionary.

        Raises:
            RuntimeError: If the file is not found or cannot be loaded.
        """
        file_path = os.path.join(self.mappings_dir, file_name) 
        try:
            with open(file_path, "r", encoding="utf-8") as f:
                return json.load(f)
        except FileNotFoundError:
            raise RuntimeError(f"File not found: {file_path}")
        except json.JSONDecodeError:
            raise RuntimeError(f"Error parsing JSON in {file_path}. Ensure it contains valid JSON.")
```

**Method Docstring**: Explains its private nature, purpose (load JSON), arguments, return value, and potential exceptions.

**Parameters**:
*   `file_name` (`str`): The name of the JSON file to load (e.g., `"table_mapping.json"`).

**Returns**:
*   `dict`: A dictionary representing the parsed JSON data.

**Raises**:
*   `RuntimeError`: Raised if the specified `file_name` cannot be found or if the content of the file is not valid JSON.

**Implementation Details**:
*   `file_path = os.path.join(self.mappings_dir, file_name)`: Constructs the full path to the JSON file by joining the base `mappings_dir` with the provided `file_name`.
*   **Error Handling (`try...except`)**:
    *   `with open(file_path, "r", encoding="utf-8") as f:`: Attempts to open the file in read mode (`"r"`) with UTF-8 encoding, ensuring proper character handling. The `with` statement guarantees the file is properly closed even if errors occur.
    *   `return json.load(f)`: Parses the JSON content from the opened file and returns it as a Python dictionary.
    *   `except FileNotFoundError:`: Catches the error if the file specified by `file_path` does not exist, raising a `RuntimeError` with a descriptive message.
    *   `except json.JSONDecodeError:`: Catches errors that occur if the file content is not valid JSON, raising a `RuntimeError` to indicate a parsing issue.

#### 3.3. `get_table_name` Method

Retrieves a database table name based on specific criteria.

```python
    def get_table_name(self, sport_code: str, entity: str, stat_period: str) -> str:
        """
        Fetches the correct table name based on sport_code, entity, and stat_period.

        Args:
            sport_code (str): The sport identifier (e.g., "MFB").
            entity (str): The entity type (e.g., "player", "team").
            stat_period (str): The statistical period (e.g., "game", "season").

        Returns:
            str: The corresponding table name, or None if not found.
        """
        return self.table_mapping.get(sport_code, {}).get(entity, {}).get(stat_period, None)
```

**Method Docstring**: Explains its purpose, arguments, and return value.

**Parameters**:
*   `sport_code` (`str`): A code identifying the sport (e.g., "MFB" for Men's Football).
*   `entity` (`str`): The type of entity (e.g., "player", "team").
*   `stat_period` (`str`): The period for which statistics are relevant (e.g., "game", "season").

**Returns**:
*   `str`: The name of the database table, or `None` if no matching table name is found for the given criteria.

**Implementation Details**:
*   `return self.table_mapping.get(sport_code, {}).get(entity, {}).get(stat_period, None)`: This line safely navigates a potentially nested dictionary structure (`self.table_mapping`).
    *   `self.table_mapping.get(sport_code, {})`: Tries to get the dictionary for the `sport_code`. If `sport_code` is not found, it returns an empty dictionary `{}` instead of raising a `KeyError`.
    *   `.get(entity, {})`: From the result of the previous call, it tries to get the dictionary for `entity`. If not found, it returns `{}`.
    *   `.get(stat_period, None)`: Finally, from the result of the previous call, it tries to get the value for `stat_period`. If found, it returns the table name (a string); otherwise, it returns `None`. This chain of `.get()` calls ensures graceful handling of missing keys at any level.

#### 3.4. `get_fields_for_query` Method

Retrieves a list of field names required for a specific SQL query.

```python
    def get_fields_for_query(self, sport_code : str, entity: str, stat_period: str, query_type: str) -> list:
        """
        Fetches the list of fields required for an SQL query based on entity, stat_period, and query_type.

        Args:
            entity (str): The entity type (e.g., "player", "team").
            stat_period (str): The statistical period (e.g., "game", "season").
            query_type (str): The type of query (e.g., "summary", "detailed").

        Returns:
            list: A list of field names, or an empty list if not found.
        """
        # return self.fields_mapping.get(entity, {}).get(stat_period, {}).get(query_type, [])
        return self.fields_mapping.get(sport_code, {}).get(entity, {}).get(stat_period, {}).get(query_type, [])
```

**Method Docstring**: Explains its purpose, arguments, and return value. Note the docstring mentions `entity`, `stat_period`, `query_type`, but the implementation also uses `sport_code`. The docstring should ideally be updated to reflect the `sport_code` parameter.

**Parameters**:
*   `sport_code` (`str`): A code identifying the sport. (e.g., "MFB")
*   `entity` (`str`): The type of entity (e.g., "player", "team").
*   `stat_period` (`str`): The statistical period (e.g., "game", "season").
*   `query_type` (`str`): The specific type of query (e.g., "summary", "detailed").

**Returns**:
*   `list`: A list of strings, where each string is a field name. Returns an empty list `[]` if no matching fields are found.

**Implementation Details**:
*   `return self.fields_mapping.get(sport_code, {}).get(entity, {}).get(stat_period, {}).get(query_type, [])`: Similar to `get_table_name`, this line uses nested `.get()` calls to safely retrieve the list of fields from `self.fields_mapping`. It navigates through `sport_code`, `entity`, `stat_period`, and `query_type` keys. If any key is not found, it defaults to an empty dictionary, eventually returning an empty list `[]` if the final `query_type` key is not found. The commented-out line shows a previous version that did not include `sport_code` as a key.

#### 3.5. `load_filter_mappings` Method

Returns the preloaded filter mappings.

```python
    def load_filter_mappings(self) -> dict:
        """
        Returns preloaded filter mappings.

        Returns:
            dict: A dictionary containing filter mappings.
        """
        return self.filter_mappings
```

**Method Docstring**: Explains its purpose and return value.

**Parameters**: None.

**Returns**:
*   `dict`: The preloaded dictionary containing filter mappings.

**Implementation Details**:
*   `return self.filter_mappings`: Directly returns the `self.filter_mappings` attribute that was preloaded during initialization.

#### 3.6. `get_stat_mapping` Method

Retrieves statistical mappings for a given sport and entity. This method loads the mapping dynamically from a file.

```python
    def get_stat_mapping(self, sport_code: str, entity: str) -> dict:
        """
        Retrieves stat mappings for a given sport and entity (Player/Team).

        Args:
            sport_code (str): The sport identifier (e.g., "MFB", "MBB").
            entity (str): The entity type ("Player", "Team").

        Returns:
            dict: Stat mapping dictionary with category field included.
        """
        file_path = os.path.join(self.stat_mapping_dir, sport_code, f"{entity}.json")
        try:
            with open(file_path, "r", encoding="utf-8") as f:
                data = json.load(f)
                for entry in data:
                    # Ensure category and sub_category are properly set
                    entry["category"] = entry.get("category", "Unknown")
                    entry["sub_category"] = entry.get("sub_category", "")
                return data
        except FileNotFoundError:
            logger.error(f"Stat mapping file not found: {file_path}")
            return []
        except json.JSONDecodeError:
            logger.error(f"Error parsing JSON in {file_path}")
            return []
```

**Method Docstring**: Explains its purpose, arguments, and return value.

**Parameters**:
*   `sport_code` (`str`): The sport identifier.
*   `entity` (`str`): The entity type (e.g., "Player", "Team").

**Returns**:
*   `dict`: A dictionary (or list of dictionaries if the JSON root is a list) containing the stat mappings. If the file is not found or parsing fails, an empty list `[]` is returned.

**Implementation Details**:
*   `file_path = os.path.join(self.stat_mapping_dir, sport_code, f"{entity}.json")`: Constructs the full path to the specific stat mapping JSON file. The path is typically `.../data_config/mappings/stat_mappings/<sport_code>/<entity>.json`.
*   **Error Handling (`try...except`)**:
    *   `with open(file_path, "r", encoding="utf-8") as f:`: Attempts to open the file.
    *   `data = json.load(f)`: Loads the JSON content.
    *   `for entry in data:`: Iterates through each entry in the loaded `data` (assuming `data` is a list of dictionaries).
    *   `entry["category"] = entry.get("category", "Unknown")`: Ensures that each entry has a "category" key, defaulting to "Unknown" if not present.
    *   `entry["sub_category"] = entry.get("sub_category", "")`: Ensures that each entry has a "sub_category" key, defaulting to an empty string if not present.
    *   `return data`: Returns the processed `data`.
    *   `except FileNotFoundError:`: Catches if the file does not exist.
        *   `logger.error(...)`: Logs an error message. **Note**: The `logger` object is not defined within the provided script snippet. This implies a dependency on a global or previously imported logger instance (e.g., from Python's `logging` module), which is missing here.
        *   `return []`: Returns an empty list upon `FileNotFoundError`.
    *   `except json.JSONDecodeError:`: Catches if the file contains invalid JSON.
        *   `logger.error(...)`: Logs an error message.
        *   `return []`: Returns an empty list upon `json.JSONDecodeError`.

#### 3.7. `get_conferences` Method

Retrieves conference mappings for a given sport. This method loads the mapping dynamically from a file.

```python
    def get_conferences(self, sport_code: str) -> dict:
        """
        Retrieves conference mappings for a given sport and entity (Player/Team).

        Args:
            sport_code (str): The sport identifier (e.g., "MFB", "MBB").

        Returns:
            dict: Conference mapping dictionary.
        """
        file_path = os.path.join(self.conference_mapping_dir, sport_code, f"conference_mapping.json")
        try:
            with open(file_path, "r", encoding="utf-8") as f:
                return json.load(f)
        except FileNotFoundError:
            return {}
```

**Method Docstring**: Explains its purpose, arguments, and return value. The reference to "entity (Player/Team)" in the docstring seems to be a leftover from a copy-paste and is not relevant for conference mappings which are sport-specific.

**Parameters**:
*   `sport_code` (`str`): The sport identifier.

**Returns**:
*   `dict`: A dictionary containing the conference mappings. Returns an empty dictionary `{}` if the file is not found.

**Implementation Details**:
*   `file_path = os.path.join(self.conference_mapping_dir, sport_code, f"conference_mapping.json")`: Constructs the full path to the conference mapping JSON file, e.g., `.../data_config/mappings/conference_mappings/<sport_code>/conference_mapping.json`.
*   **Error Handling (`try...except`)**:
    *   `with open(file_path, "r", encoding="utf-8") as f:`: Attempts to open the file.
    *   `return json.load(f)`: Loads and returns the JSON content.
    *   `except FileNotFoundError:`: If the file does not exist, it gracefully returns an empty dictionary `{}`. This is a softer error handling than `_load_json` or `get_stat_mapping`, implying that missing conference data might be acceptable in some contexts.

#### 3.8. `get_query_samples` Method

Retrieves example queries for a specific sport and entity.

```python
    def get_query_samples(self, sport_code: str, entity: str) -> list:
        """
        Retrieves example queries for a given sport code and entity.

        Args:
            sport_code (str): The sport identifier (e.g., "MFB", "MBB").
            entity (str): The entity type ("Player", "Team").

        Returns:
            list: A list of example queries.
        """
        file_path = os.path.join(self.query_samples_dir, sport_code, f"{entity.title()}.json")
        print(f"[DEBUG] Trying to load: {file_path}")

        if not os.path.exists(file_path):
            print(f"[ERROR] File not found: {file_path}")
            return []

        try:
            with open(file_path, "r", encoding="utf-8") as f:
                data = json.load(f)
                if isinstance(data, dict):
                    return data.get("query_samples", [])
                elif isinstance(data, list):
                    return data  # ← fallback if file is a plain list
        except Exception as e:
            print(f"[ERROR] Failed to load query samples: {e}")
        return []
```

**Method Docstring**: Explains its purpose, arguments, and return value.

**Parameters**:
*   `sport_code` (`str`): The sport identifier.
*   `entity` (`str`): The entity type (e.g., "Player", "Team"). The `.title()` method is used to ensure proper casing for the filename.

**Returns**:
*   `list`: A list of example queries (strings or dicts representing queries). Returns an empty list `[]` if the file is not found or an error occurs during loading/parsing.

**Implementation Details**:
*   `file_path = os.path.join(self.query_samples_dir, sport_code, f"{entity.title()}.json")`: Constructs the file path. For example, `.../data_config/query_samples/MFB/Player.json`.
*   `print(f"[DEBUG] Trying to load: {file_path}")`: A debug print statement indicating the file being loaded.
*   `if not os.path.exists(file_path):`: Checks if the file actually exists on the filesystem.
    *   `print(f"[ERROR] File not found: {file_path}")`: If the file doesn't exist, an error message is printed.
    *   `return []`: An empty list is returned.
*   **Error Handling (`try...except`)**:
    *   `with open(file_path, "r", encoding="utf-8") as f:`: Attempts to open the file.
    *   `data = json.load(f)`: Loads the JSON content.
    *   `if isinstance(data, dict): return data.get("query_samples", [])`: If the loaded JSON is a dictionary, it attempts to extract a list associated with the key "query_samples". If this key is missing, it returns an empty list.
    *   `elif isinstance(data, list): return data`: If the loaded JSON is directly a list, it returns that list. This provides flexibility for the structure of `query_samples` JSON files.
    *   `except Exception as e:`: Catches any other exceptions during file operations or JSON parsing.
        *   `print(f"[ERROR] Failed to load query samples: {e}")`: Prints a generic error message. **Note**: Similar to `get_stat_mapping`, using `print` for errors is generally less robust than a proper logging framework.
    *   `return []`: Returns an empty list if any exception occurs.

#### 3.9. `get_pos_mapping` Method

Retrieves position mappings for a specific sport.

```python
    def get_pos_mapping(self, sport_code: str) -> dict:
        """
        Retrieves position mappings for a given sport.

        Args:
            sport_code (str): The sport identifier (e.g., "MFB", "MBB").

        Returns:
            dict: Position mapping dictionary for the given sport.
        """
        return self.position_mappings.get(sport_code, {})
```

**Method Docstring**: Explains its purpose, arguments, and return value.

**Parameters**:
*   `sport_code` (`str`): The sport identifier.

**Returns**:
*   `dict`: A dictionary containing position mappings for the specified sport. Returns an empty dictionary `{}` if `sport_code` is not found in the preloaded `position_mappings`.

**Implementation Details**:
*   `return self.position_mappings.get(sport_code, {})`: Accesses the preloaded `self.position_mappings` dictionary and retrieves the sub-dictionary corresponding to the `sport_code`. It uses `get()` with an empty dictionary as a default to prevent `KeyError` if the `sport_code` is not found.

#### 3.10. `get_division_mapping` Method

Retrieves division mappings for a specific sport.

```python
    def get_division_mapping(self, sport_code: str) -> dict:
        return self.division_mappings.get(sport_code, {})
```

**Method Docstring**: Missing. Based on the implementation, it should be similar to `get_pos_mapping`.

**Parameters**:
*   `sport_code` (`str`): The sport identifier.

**Returns**:
*   `dict`: A dictionary containing division mappings for the specified sport. Returns an empty dictionary `{}` if `sport_code` is not found.

**Implementation Details**:
*   `return self.division_mappings.get(sport_code, {})`: Accesses the preloaded `self.division_mappings` dictionary and retrieves the sub-dictionary corresponding to the `sport_code`. It uses `get()` with an empty dictionary as a default to prevent `KeyError` if the `sport_code` is not found.

#### 3.11. `get_metadata_mapping` Method

Returns the preloaded metadata mappings.

```python
    def get_metadata_mapping(self) -> dict:
        """
        Returns metadata mappings.

        Returns:
            dict: A dictionary containing metadata mappings.
        """
        return self.metadata_mappings
```

**Method Docstring**: Explains its purpose and return value.

**Parameters**: None.

**Returns**:
*   `dict`: The preloaded dictionary containing metadata mappings.

**Implementation Details**:
*   `return self.metadata_mappings`: Directly returns the `self.metadata_mappings` attribute that was preloaded during initialization.

### 4. Main Execution Flow

The provided Python script defines a class `MappingsHandler`. It does not contain an `if __name__ == "__main__":` block, meaning it is intended to be imported as a module into other scripts rather than executed directly. When imported, the `MappingsHandler` class definition becomes available for instantiation.

### 5. Important Implementation Details and Considerations

*   **Path Management**: The use of `os.path.dirname(__file__)` multiple times ensures that the base directory calculation is robust, regardless of where the `MappingsHandler.py` file is executed from, as long as its relative position within the project structure remains consistent. `os.path.join` is used throughout for platform-independent path construction.
*   **Constants**: The reliance on `MappingsHandlerTerms` (presumably an `Enum`) for directory and file names promotes consistency, reduces magic strings, and makes the code easier to maintain and understand.
*   **Preloading vs. On-Demand Loading**: Static, frequently accessed mappings (`table_mapping`, `fields_mapping`, `filter_mappings`, `position_mappings`, `metadata_mappings`, `division_mappings`) are preloaded in `__init__` to optimize performance by minimizing disk I/O. More dynamic or less frequently used mappings (`stat_mapping`, `conference_mapping`, `query_samples`) are loaded on demand when their respective getter methods are called. This balances memory usage and performance.
*   **Robust Dictionary Access**: Methods like `get_table_name` and `get_fields_for_query` utilize nested `.get(key, default_value)` calls. This is a best practice for accessing potentially missing keys in nested dictionaries, preventing `KeyError` exceptions and providing sensible default fallback values (e.g., `None` or `[]`).
*   **Error Handling**:
    *   `_load_json` provides robust error handling for `FileNotFoundError` and `json.JSONDecodeError`, raising `RuntimeError` which is appropriate for critical setup failures.
    *   `get_stat_mapping`, `get_conferences`, and `get_query_samples` implement specific error handling for `FileNotFoundError` and `json.JSONDecodeError` (or general `Exception` for `get_query_samples`). They generally return empty lists or dictionaries on error, allowing the calling code to proceed without crashing, albeit with missing data.
    *   **Logging**: The `get_stat_mapping` method attempts to use a `logger` object for error reporting, but `logger` is not defined in the provided snippet. This is a critical missing dependency that would cause a `NameError` if this method were called. It should either be imported or instantiated within the class.
    *   **`print` for Errors**: In `get_query_samples`, `print` statements are used for debug and error reporting. While functional for basic debugging, it is generally recommended to use a proper logging framework (like Python's `logging` module) for production applications, as it provides more flexibility for configuring log levels, destinations, and formats.
*   **JSON Structure Flexibility**: `get_query_samples` intelligently handles two potential JSON structures for query samples (either a dictionary with a "query_samples" key or a direct list), making it more adaptable to different data conventions.
*   **Data Transformation**: `get_stat_mapping` includes logic to ensure that `category` and `sub_category` fields exist for each entry in the stat mapping data, providing default values if they are missing. This helps maintain data consistency for downstream consumers.
