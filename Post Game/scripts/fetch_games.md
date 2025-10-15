# Documentation for `fetch_games.py`

# Technical Documentation: Python Script for Fetching and Batching Game Data

## Overview

This Python script fetches game data from a PostgreSQL database and processes it for parallel processing.  It supports three sports: Men's Football (MFB), Men's Basketball (MBB), and Women's Basketball (WBB). The script retrieves completed games, handles potential team conflicts, and outputs the data in either JSON or CSV format.  It uses a connection pool for efficient database interaction and includes robust error handling.


## Modules and Dependencies

The script relies on several Python modules:

* `json`: For JSON serialization and deserialization.
* `collections.defaultdict`: For creating dictionaries with default values.
* `datetime`, `timedelta`, `date`: For date and time manipulation.
* `enum`: For defining enumerated types.
* `pathlib.Path`: For working with file paths.
* `typing`: For type hinting.
* `psycopg2`, `psycopg2.extras`: For PostgreSQL database interaction (including using `RealDictCursor`).
* `typer`: For creating a command-line interface.


## Classes

### `SportType` (Enum)

Defines enumerated types for the supported sports:  `MFB`, `MBB`, `WBB`.

### `OutputFormat` (Enum)

Defines enumerated types for the supported output formats: `csv`, `json`.

### `DatabaseConnectionError` (Exception)

A custom exception class raised when there's an error connecting to the database. It includes the original exception for debugging purposes.

### `QueryExecutionError` (Exception)

A custom exception class raised when there's an error executing a SQL query. It includes the SQL statement, parameters, and original exception for better error analysis.


## Functions

### `get_basketball_date_range(base_date_str: str) -> List[date]`

This function takes a base date string (YYYY-MM-DD) and returns a list of three dates: the base date, the previous day, and the day before that. This accounts for potential timezone differences in basketball game times.

### `collect_games_backward_football(cursor, start_date: str) -> List[Dict]`

Retrieves completed football games from the database, working backward from the `start_date` until an empty date is encountered.  It employs a safety limit of 30 days to prevent excessively long queries. The query uses common table expressions (CTEs) for better readability and performance.  It returns a list of dictionaries, each representing a game.

### `collect_games_basketball_window(cursor, start_date: str, sport: SportType) -> List[Dict]`

Retrieves completed basketball games for a three-day window (base date and two preceding days) using a single query.  The specific table queried depends on the `sport` parameter.  It returns a list of dictionaries, each representing a game.


### `build_conflict_graph(games: List[Dict]) -> Dict[int, Set[int]]`

Constructs a conflict graph where nodes represent games, and an edge connects two games if they share a team. This graph is used for batch assignment to avoid scheduling conflicts.

### `games_share_team(game1: Dict, game2: Dict) -> bool`

Checks if two games share any teams.

### `assign_games_to_batches(games: List[Dict], conflicts: Dict[int, Set[int]], max_batches: int) -> Dict[int, List[Dict]]`

Assigns games to batches, aiming for a balanced distribution while avoiding conflicts based on the conflict graph.  It prioritizes smaller batches first for better balance.


### `can_add_to_batch(game_idx: int, batch_num: int, conflicts: Dict[int, Set[int]], game_to_batch: Dict[int, int]) -> bool`

Checks if adding a specific game to a batch would create a conflict.

### `fetch_games_data(date_str: str, sport: SportType, num_batches: int = 3, db_config: Optional[Dict[str, Union[str, int]]] = None, connection_pool=None, config: Optional[Dict] = None) -> Dict[str, Union[Dict, List]]`

This is the core function of the script. It handles database connection, fetches game data based on the sport and date, builds the conflict graph, assigns games to batches, and returns the results with metadata.  It supports various ways to specify database credentials.

### `fetch_game_by_match_id(g_match_id: int, sport: SportType, db_config: Optional[Dict[str, Union[str, int]]] = None, connection_pool=None, config: Optional[Dict] = None) -> List[Dict[str, Union[str, int]]]`

Fetches data for a specific game given its match ID. It supports various ways to specify database connection details, similar to `fetch_games_data`.


### `format_output(data: Union[List[Dict[str, Union[str, int]]], Dict[str, Union[Dict, List]]], output_format: OutputFormat) -> str`

Formats the game data into either JSON or CSV string format.  Note that CSV is only supported for single-match queries (not batched data).


## Command-Line Interface (CLI)

The script uses `typer` to create a command-line interface. The available options are:

* `--date` or `-d`: The base date (YYYY-MM-DD) for date-based queries.
* `--match-id` or `-m`: The specific match ID for single-match queries.
* `--sport` or `-s`:  The sport type (MFB, MBB, WBB).
* `--batches` or `-b`: The number of batches (1-20). Default is 3.
* `--format` or `-f`: The output format (json or csv). Default is json.


## Error Handling

The script includes comprehensive error handling for database connection issues, query execution failures, invalid input, and configuration problems.  Custom exception classes (`DatabaseConnectionError`, `QueryExecutionError`) provide detailed error information.


## Database Configuration

The script attempts to load database credentials from several sources in this order of priority:
1. Explicit `connection_pool` (for connection pooling)
2. Explicit `config` dictionary passed to the function
3. `db_config` dictionary (for backward compatibility)
4. `environment-variables.json` file in the same directory as the script.  

The `environment-variables.json` file should contain the following keys: `PG_HOST`, `PG_PORT`, `PG_DATABASE`, `PG_USERNAME`, `PG_PASSWORD`.  If none of these methods work, it will raise an error.  Providing a `connection_pool` is recommended for production deployments for efficiency and resilience.


## Usage Example

To fetch data for basketball games on 2024-03-08, outputting as JSON:

```bash
python your_script_name.py --date 2024-03-08 --sport MBB
```

To fetch data for a specific match ID (12345) in football format as CSV:

```bash
python your_script_name.py --match-id 12345 --sport MFB --format csv
```

Remember to replace `your_script_name.py` with the actual name of your Python file.

