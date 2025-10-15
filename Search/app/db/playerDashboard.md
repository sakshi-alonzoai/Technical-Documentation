# Documentation for `playerDashboard.py`

This document provides a comprehensive technical overview of the `PlayerDashboardAdapter.py` Python script, detailing its purpose, architecture, and the functionality of each component.

---

## Technical Documentation: `PlayerDashboardAdapter.py`

### 1. High-Level Overview

The `PlayerDashboardAdapter.py` script defines a `PlayerDashboardAdapter` class, which extends a base `PGSQLAdapter` for PostgreSQL database interactions. Its primary purpose is to serve as a specialized data access layer for retrieving various player-centric statistics and records from a PostgreSQL database. It handles complex data retrieval tasks for player profiles, game logs, historical records, career highs, and performance streaks across different sports.

The adapter utilizes SQL templates for dynamic query generation, applies stat mapping to transform raw database column names into more user-friendly short names, and structures the retrieved data into a consistent dictionary format for consumption by upstream services or APIs. It abstracts away the complexities of database querying and data transformation, providing a clean interface for accessing player dashboard data.

### 2. Dependencies and Imports

The script relies on several internal and external modules for its functionality:

*   **`collections.defaultdict`**:
    *   **Purpose**: Used to create dictionaries with a default value for missing keys, particularly for the `stat_to_short_name` mapping, ensuring that if a stat name is not found in the mapping, it defaults to `None` or itself rather than raising a `KeyError`.
*   **`logging`**:
    *   **Purpose**: Standard Python library for logging events, errors, and debugging information to the console or other configured handlers.
*   **`app.data_config.mappings.mappings_handler.MappingsHandler`**:
    *   **Purpose**: An internal module responsible for handling data mappings, likely used to determine the correct database table name based on sport, entity type, and stat period.
    *   **Assumed usage**: An instance of this class (`self.mappings_handler`) is expected to be available through inheritance or composition in `PGSQLAdapter`.
*   **`app.constants.PostgreSQL`**:
    *   **Purpose**: An internal module that likely contains constants related to PostgreSQL database configuration or specific database behaviors. While imported, it's not directly used in the provided code snippet.
*   **`app.sql_templates.sql_template_loader.SQLTemplateLoader`**:
    *   **Purpose**: An internal utility class designed to load SQL query templates from specified file paths. This allows for parameterizing SQL queries without hardcoding them directly into the Python script.
*   **`app.db.pgsql_adapter.PGSQLAdapter`**:
    *   **Purpose**: The base class from which `PlayerDashboardAdapter` inherits. It's expected to provide fundamental PostgreSQL database interaction capabilities, such as connection management (`get_cursor()`), and potentially methods for fetching stat mappings (`fetch_stat_to_short_name`, `fetch_stat_mapping_R2`, `get_stat_mapping_by_position`).

### 3. Logging Configuration

```python
logging.basicConfig(level=logging.INFO, format='%(asctime)s - %(levelname)s - %(message)s')
logger = logging.getLogger()
```

*   **`logging.basicConfig(...)`**: Configures the root logger.
    *   `level=logging.INFO`: Sets the minimum logging level to `INFO`. This means messages with `INFO`, `WARNING`, `ERROR`, and `CRITICAL` severity will be processed, while `DEBUG` messages will be ignored.
    *   `format='%(asctime)s - %(levelname)s - %(message)s'`: Defines the format for log messages, including timestamp, log level, and the log message itself.
*   **`logger = logging.getLogger()`**: Retrieves the root logger instance, which will be used throughout the script for logging various events and errors.

### 4. Class: `PlayerDashboardAdapter`

```python
class PlayerDashboardAdapter(PGSQLAdapter):
    def __init__(self):
        super().__init__()
```

*   **`class PlayerDashboardAdapter(PGSQLAdapter):`**: Defines a class named `PlayerDashboardAdapter` that inherits from `PGSQLAdapter`. This inheritance implies that `PlayerDashboardAdapter` will have access to all methods and properties of `PGSQLAdapter`, such as database connection handling.
*   **`def __init__(self):`**: The constructor for the `PlayerDashboardAdapter` class.
    *   **`super().__init__()`**: Calls the constructor of the parent class (`PGSQLAdapter`). This is crucial for initializing any database connection pools, mapping handlers (`self.mappings_handler`), or other base functionalities defined in `PGSQLAdapter`.

#### 4.1. Method: `execute_player_profile`

```python
    def execute_player_profile(self, sport_code, player_id, entity="Player", stat_period="season"):
        """
        Executes a SQL query for player profile.
        
        Args:
            sport_code (str): Sport code ('MFB' or 'MBB' or 'WBB').
            entity (str): Entity type ('Player' or 'Team').
            player_id (str): Player identifier code.
            stat_period (str): The period for which stats are being retrieved (default: "season").
        
        Returns:
            dict: Player profile data, or None if no rows are found.
        """
        with self.get_cursor() as cursor:
            try:
                # Get table name
                table_name = self.mappings_handler.get_table_name(sport_code, entity, stat_period)
                
                # Load SQL query template
                query = SQLTemplateLoader(f'app/sql_templates/Profile/{sport_code}/{entity}').load_template(
                    f"{stat_period}")
                
                # Format query with parameters
                query = query.format(player_id=player_id, table_name=table_name)
                print("table_name is i guess {table_name}") # Debug print
                logger.info(f"Executing SQL Query:\n{query}")

                # Execute query and fetch results
                cursor.execute(query)
                rows = cursor.fetchall()

                if not rows:
                    return None # Return None if no data is found

                # Fetch stat short names and prepare for column mapping
                stat_to_short_name = self.fetch_stat_to_short_name(sport_code, entity)
                stat_to_short_name = defaultdict(lambda: None, stat_to_short_name)
                column_names = [stat_to_short_name[desc[0]] or desc[0] for desc in cursor.description]
                
                # Convert rows to list of dictionaries
                player_data = [dict(zip(column_names, row)) for row in rows]
                first_record = player_data[0]
                logger.info(f"Record keys from DB: {list(first_record.keys())}")

                # Extract common player/team information
                player_name = first_record.get("player_name", "")
                team_name = first_record.get("team_name", "")
                team_code = first_record.get("team_code", "")
                player_position = first_record.get("position", "")

                # Fetch and transform stat mapping
                stat_mapping = self.fetch_stat_mapping_R2(sport_code, entity) # Original mapping
                stat_mapping = self.get_stat_mapping_by_position(sport_code, entity, player_position) # Position-specific mapping

                # Collect all stat names and apply short names for MBB/WBB
                stats_names = [stat for sublist in stat_mapping.values() for stat in sublist]
                if sport_code in ("MBB", "WBB"):
                    transformed_stat_mapping = {}
                    for category, stats in stat_mapping.items():
                        transformed_stat_mapping[category] = [stat_to_short_name[stat] or stat for stat in stats]
                    stat_mapping = transformed_stat_mapping
                    stats_names = [stat_to_short_name[stat] or stat for stat in stats_names]

                stats_names.append("season") # Add 'season' to stats to keep
                logger.info(f"Stat names used for filtering: {stats_names}")

                # Filter player data based on desired stat names
                filtered_player_data = []
                for record in player_data:
                    filtered_record = {k: v for k, v in record.items() if k in stats_names}
                    filtered_player_data.append(filtered_record)
                    logger.info(f"Filtered player data sample: {filtered_player_data[0]}")

                # Return structured player profile data
                return {
                    "PlayerName": player_name,
                    "SportCode": sport_code,
                    "TeamName": team_name,
                    "TeamCode": str(team_code),
                    "PlayerPosition": player_position,
                    "stats": filtered_player_data,
                    "stat_mapping": stat_mapping
                }

            except Exception as e:
                logger.error(f"Database Query Error: {e}")
                raise
```

*   **Purpose**: Retrieves comprehensive profile statistics for a given player, typically aggregated by season. It fetches raw data, transforms column names, filters relevant statistics, and organizes them into a structured dictionary.
*   **Parameters**:
    *   `sport_code` (str): Unique code for the sport (e.g., 'MFB' for Men's Football, 'MBB' for Men's Basketball, 'WBB' for Women's Basketball).
    *   `player_id` (str): Unique identifier for the player.
    *   `entity` (str, optional): The type of entity, defaults to "Player".
    *   `stat_period` (str, optional): The aggregation period for statistics, defaults to "season".
*   **Returns**:
    *   `dict`: A dictionary containing player's name, sport code, team details, position, a list of filtered statistics, and a categorized stat mapping. Returns `None` if no player data is found.
*   **Implementation Logic**:
    1.  **Context Manager**: `with self.get_cursor() as cursor:` ensures that a database cursor is acquired and properly released (or rolled back/committed by the `PGSQLAdapter`) after the operation, even if errors occur.
    2.  **Table Name Retrieval**: `self.mappings_handler.get_table_name()` determines the correct database table based on `sport_code`, `entity`, and `stat_period`.
    3.  **SQL Template Loading**: `SQLTemplateLoader` loads a SQL query template from a path constructed using `sport_code`, `entity`, and `stat_period`.
    4.  **Query Formatting**: The loaded query string is formatted using `player_id` and the `table_name` to create a complete SQL statement. A `print` statement aids debugging, and `logger.info` logs the final query.
    5.  **Query Execution**: `cursor.execute(query)` sends the SQL query to the database, and `cursor.fetchall()` retrieves all resulting rows.
    6.  **No Data Check**: If `rows` is empty, the function returns `None`.
    7.  **Stat Name Mapping**:
        *   `self.fetch_stat_to_short_name()` retrieves a mapping from raw database column names to user-friendly short names.
        *   `defaultdict(lambda: None, stat_to_short_name)` is used to create a dictionary where accessing a non-existent key returns `None` instead of raising a `KeyError`.
        *   `column_names = [stat_to_short_name[desc[0]] or desc[0] for desc in cursor.description]` constructs a list of column names for the result set. It attempts to use the short name; otherwise, it falls back to the raw database column name.
    8.  **Data Structuring**: `player_data = [dict(zip(column_names, row)) for row in rows]` converts the list of tuples (`rows`) into a list of dictionaries, where each dictionary represents a player record with user-friendly keys.
    9.  **Extracting Player Info**: Common details like player name, team name, team code, and position are extracted from the `first_record`.
    10. **Stat Mapping for Display**:
        *   `self.fetch_stat_mapping_R2()` (likely a generic stat mapping) and `self.get_stat_mapping_by_position()` (more specific, overriding if necessary) are called to retrieve structured mappings of stats (e.g., grouped by categories like "Offense", "Defense").
        *   For "MBB" and "WBB" sports, the `stat_mapping` and individual `stats_names` are further transformed using `stat_to_short_name` to ensure consistency in display names.
    11. **Filtering Stats**: The `stats_names` list (including "season") is used to filter `player_data`, ensuring only relevant statistics are included in the final output.
    12. **Return Value**: A comprehensive dictionary is returned, containing core player information, the filtered list of stats, and the transformed stat mapping.
    13. **Error Handling**: A `try...except` block catches `Exception` during database operations, logs the error, and re-raises it to propagate the issue.

#### 4.2. Method: `execute_player_game_logs`

```python
    def execute_player_game_logs(self, sport_code, player_id, season=None, entity="Player", stat_period="game"):
        """
        Executes a SQL query for player profile. # NOTE: Documentation here seems copied from execute_player_profile, should be updated.
        Executes a SQL query for player game logs.

        Args:
            sport_code (str): Sport code ('MFB' or 'MBB' or 'WBB').
            player_id (str): Player identifier code.
            season (str, optional): The specific season to retrieve game logs for. If None, the most recent season is used.
            entity (str, optional): Entity type ('Player' or 'Team'), defaults to "Player".
            stat_period (str, optional): The period for which stats are being retrieved, defaults to "game".

        Returns:
            dict: Player game log data, or None if no rows are found.
        """
        with self.get_cursor() as cursor:
            try:
                # Get table name
                table_name = self.mappings_handler.get_table_name(sport_code, entity, stat_period)
                
                # Query to fetch available seasons for the player
                sql_query_seasons = f"""
                    SELECT DISTINCT season
                    FROM {table_name}
                    WHERE player_id = '{player_id}'
                    ORDER BY season DESC;
                """
                cursor.execute(sql_query_seasons)
                seasons = cursor.fetchall()
                seasons = [season[0] for season in seasons]
                
                # Determine the season to query
                if season:
                    pass # Use provided season
                else:
                    season = seasons[0] # Use the most recent season if not specified

                # Load SQL query template for game logs
                query = SQLTemplateLoader(f'app/sql_templates/Profile/{sport_code}/{entity}').load_template(
                    f"{stat_period}")
                logger.info(f"Executing SQL Query: for {sport_code}/{entity}: \n{query}")

                # Execute query for game logs of the selected season
                cursor.execute(query.format(player_id=player_id, season=f"'{season}'", table_name=table_name))
                rows = cursor.fetchall()

                if not rows:
                    return None # Return None if no data is found

                # Fetch stat short names and prepare for column mapping
                stat_to_short_name = self.fetch_stat_to_short_name(sport_code, entity)
                stat_to_short_name = defaultdict(lambda: None, stat_to_short_name)
                column_names = [stat_to_short_name[desc[0]] or desc[0] for desc in cursor.description]
                
                # Convert rows to list of dictionaries
                player_data = [dict(zip(column_names, row)) for row in rows]

                # Extract common player/team information
                first_record = player_data[0]
                player_name = first_record.get("player_name", "")
                team_name = first_record.get("team_name", "")
                team_code = first_record.get("team_code", "")
                player_position = first_record.get("position", "")

                # Fetch and transform stat mapping
                stat_mapping = self.get_stat_mapping_by_position(sport_code, entity, player_position)

                # Collect all stat names and apply short names for MBB/WBB (same logic as execute_player_profile)
                stats_names = [stat for sublist in stat_mapping.values() for stat in sublist]
                if sport_code in ("MBB", "WBB"):
                    print("Raw stat_mapping before short name transform (MBB/WBB):", stat_mapping) # Debug print
                    transformed_stat_mapping = {}
                    for category, stats in stat_mapping.items():
                        transformed_stat_mapping[category] = [stat_to_short_name[stat] or stat for stat in stats]
                    stat_mapping = transformed_stat_mapping
                    stats_names = [stat_to_short_name[stat] or stat for stat in stats_names]
                    print("Transformed stats_names:", stats_names) # Debug print

                stats_names.append("season") # Add essential display columns
                stats_names.append("opponent_team_name")
                stats_names.append("game_date")

                # Filter player data based on desired stat names
                filtered_player_data = []
                for record in player_data:
                    filtered_record = {k: v for k, v in record.items() if k in stats_names}
                    filtered_player_data.append(filtered_record)

                # Return structured game log data
                return {
                    "PlayerName": player_name,
                    "SportCode": sport_code,
                    "TeamName": team_name,
                    "TeamCode": str(team_code),
                    "PlayerPosition": player_position,
                    "Seasons": seasons, # List of all seasons available for the player
                    "stats": filtered_player_data,
                    "stat_mapping": stat_mapping
                }

            except Exception as e:
                logger.error(f"Database Query Error: {e}")
                raise
```

*   **Purpose**: Retrieves a player's game-by-game statistics for a specified season (or the most recent season if none is provided). It also lists all seasons for which game log data is available for the player.
*   **Parameters**:
    *   `sport_code` (str): Sport code (e.g., 'MFB', 'MBB', 'WBB').
    *   `player_id` (str): Player identifier code.
    *   `season` (str, optional): Specific season to query. If `None`, the most recent season is used.
    *   `entity` (str, optional): Entity type, defaults to "Player".
    *   `stat_period` (str, optional): Stat aggregation period, defaults to "game".
*   **Returns**:
    *   `dict`: A dictionary containing player's name, sport code, team details, position, a list of available seasons, a list of filtered game logs, and a categorized stat mapping. Returns `None` if no game log data is found.
*   **Implementation Logic**:
    1.  **Context Manager**: Uses `with self.get_cursor() as cursor:` for cursor management.
    2.  **Table Name Retrieval**: `self.mappings_handler.get_table_name()` determines the appropriate table.
    3.  **Fetch Available Seasons**: A direct SQL query is executed to find all distinct seasons for the given `player_id` in descending order. This allows the API consumer to know which seasons are available.
    4.  **Determine Target Season**: If `season` is provided, it's used; otherwise, the first (most recent) season from the fetched list is selected.
    5.  **SQL Template Loading**: Loads the game log specific SQL template from `app/sql_templates/Profile/{sport_code}/{entity}`.
    6.  **Query Execution**: The query is formatted with `player_id`, the determined `season`, and `table_name`, then executed.
    7.  **No Data Check**: Returns `None` if `rows` is empty.
    8.  **Stat Name Mapping & Data Structuring**: Identical logic to `execute_player_profile` for fetching `stat_to_short_name` and converting raw `rows` into `player_data` (list of dictionaries).
    9.  **Extracting Player Info**: Extracts common player/team details.
    10. **Stat Mapping for Display**: Retrieves and potentially transforms `stat_mapping` based on `player_position` and `sport_code` (for MBB/WBB, using short names).
    11. **Filtering Stats**: `stats_names` is built from `stat_mapping` (and transformed for MBB/WBB), then augmented with additional display columns crucial for game logs: "season", "opponent\_team\_name", and "game\_date". `player_data` is then filtered.
    12. **Return Value**: A dictionary containing player information, the list of all available `Seasons`, the `filtered_player_data` (game logs), and `stat_mapping`.
    13. **Error Handling**: Standard `try...except` block for logging and re-raising exceptions.

#### 4.3. Method: `execute_player_records`

```python
    def execute_player_records(self, sport_code, entity, player_id, teamcode, stat_period, rank_threshold):
        """
        Executes a SQL query to retrieve player records and their rankings.

        This function fetches player records from the database based on the provided parameters.
        It first validates and retrieves the correct team code for the player, then executes
        a template-based SQL query to get the player's records and their rankings within
        their team. The results are transformed to include short names for statistics.

        Args:
            sport_code (str): The code representing the sport (e.g., 'MBB', 'MFB')
            entity (str): The type of entity (e.g., 'Player', 'Team')
            player_id (str): The unique identifier for the player
            teamcode (str): The team code (will be validated/overridden if incorrect or empty)
            stat_period (str): The period for which stats are being retrieved (e.g., 'season', 'career')
            rank_threshold (int): The maximum rank to include in results (e.g., 10 for top 10 records)

        Returns:
            dict: A dictionary containing:
                - PlayerRecords (list): List of dictionaries containing player records with stats and rankings
                - TotalCount (int): Total number of *filtered* records found

        Raises:
            ValueError: If no team code is found for the given player_id
            Exception: For any database query errors or other unexpected errors
        """
        with self.get_cursor() as cursor:
            try:
                # Get the correct table name based on sport code
                table_name = self.mappings_handler.get_table_name(sport_code, entity, stat_period)

                # Safely fetch team code (in case input is blank or wrong)
                team_code_sql = f"""
                    SELECT team_code FROM {table_name} WHERE player_id = '{player_id}' LIMIT 1
                """
                cursor.execute(team_code_sql)
                team_codes = cursor.fetchall()
                if not team_codes:
                    raise ValueError(f"No team_code found for player_id = {player_id}")
                teamcode = team_codes[0][0] # Override input teamcode with validated one

                # Load the SQL template for the correct sport/entity/stat_period
                query = SQLTemplateLoader(f'app/sql_templates/Records/{sport_code}/{entity}').load_template(stat_period)
                logger.info(f"Team Code is {teamcode}")
                query = query.format(player_id=player_id, team_code=f"{teamcode}", rank_threshold=rank_threshold)
                logger.info(f"Executing SQL Query:\n{query}")

                cursor.execute(query)
                rows = cursor.fetchall()

                stat_to_short_name = self.fetch_stat_to_short_name(sport_code, entity)
                column_names = [desc[0] for desc in cursor.description]
                player_records = [dict(zip(column_names, row)) for row in rows]
                
                # Filter and transform records based on stat_to_short_name
                filtered_records = []
                for row in player_records:
                    raw_stat = row['stat'].lower()
                    short_name = stat_to_short_name.get(raw_stat)
                    if short_name: # Only include stats that have a defined short name
                        row['stat'] = short_name
                        filtered_records.append(row)

                total_count = len(filtered_records) # Total count of filtered records
                return {"PlayerRecords": filtered_records, "TotalCount": total_count}

            except Exception as e:
                logger.error(f"Database Query Error: {e}")
                raise
```

*   **Purpose**: Retrieves player records (e.g., top-X performances in various stats) and their rankings, typically within their team. It ensures the correct `team_code` is used and transforms stat names for display.
*   **Parameters**:
    *   `sport_code` (str): Sport code.
    *   `entity` (str): Entity type.
    *   `player_id` (str): Player identifier.
    *   `teamcode` (str): Team code (will be validated/overridden).
    *   `stat_period` (str): Stat aggregation period (e.g., 'career', 'season').
    *   `rank_threshold` (int): The maximum rank to include (e.g., 5 for top 5).
*   **Returns**:
    *   `dict`: Contains `PlayerRecords` (list of dictionaries with transformed stat names and rankings) and `TotalCount` (the number of filtered records).
*   **Implementation Logic**:
    1.  **Context Manager**: Uses `with self.get_cursor() as cursor:`.
    2.  **Table Name Retrieval**: `self.mappings_handler.get_table_name()` to get the base table.
    3.  **Team Code Validation/Retrieval**:
        *   Executes a direct SQL query to fetch the `team_code` for the given `player_id` from the determined `table_name`. This is crucial to ensure consistency and correctness, overriding the `teamcode` parameter if necessary.
        *   Raises a `ValueError` if no `team_code` is found.
    4.  **SQL Template Loading**: Loads the records-specific SQL template from `app/sql_templates/Records/{sport_code}/{entity}`.
    5.  **Query Formatting**: Formats the query with `player_id`, the validated `teamcode`, and `rank_threshold`. Logs the formatted query.
    6.  **Query Execution**: Executes the query and fetches results.
    7.  **Stat Name Mapping & Data Structuring**:
        *   Fetches `stat_to_short_name`.
        *   Converts raw `rows` to `player_records` (list of dictionaries).
        *   The original `total_count` from the query result (if available) is extracted.
    8.  **Filtering and Transformation**: Iterates through `player_records`. For each record, it attempts to get a `short_name` for the `stat`. If a `short_name` exists, it updates the `stat` key in the record with the `short_name` and adds the record to `filtered_records`. This ensures only stats with defined short names are included and displayed.
    9.  **Recalculate Total Count**: `total_count` is updated to reflect the `len(filtered_records)`.
    10. **Return Value**: A dictionary containing `filtered_records` and `TotalCount`.
    11. **Error Handling**: Standard `try...except` block.

#### 4.4. Method: `execute_player_highs`

```python
    def execute_player_highs(self, sport_code, entity, player_id, stat_period):
        with self.get_cursor() as cursor:
            try:
                query = SQLTemplateLoader(f'app/sql_templates/Highs/{sport_code}/{entity}').load_template(
                    f"{stat_period}_high")
                query = query.format(player_id=player_id)

                logger.info(f"Executing SQL Query:\n{query}")

                cursor.execute(query)
                rows = cursor.fetchall()
                column_names = [desc[0].lower() for desc in cursor.description] # Lowercase column names
                player_highs = [dict(zip(column_names, row)) for row in rows]
                
                stat_to_short_name = self.fetch_stat_to_short_name(sport_code, entity)

                filtered_highs = []
                for row in player_highs:
                    raw_stat = row['stat'].lower()
                    short_name = stat_to_short_name.get(raw_stat)
                    if short_name:
                        row['stat'] = short_name
                        filtered_highs.append(row)

                total_count = len(filtered_highs) # Total count of filtered highs
                return {"PlayerHighs": filtered_highs, "TotalCount": total_count}

            except Exception as e:
                logger.error(f"Database Query Error: {e}")
                raise
```

*   **Purpose**: Retrieves career or season highs for a player in various statistical categories.
*   **Parameters**:
    *   `sport_code` (str): Sport code.
    *   `entity` (str): Entity type.
    *   `player_id` (str): Player identifier.
    *   `stat_period` (str): The period for which highs are being retrieved (e.g., 'season', 'career').
*   **Returns**:
    *   `dict`: A dictionary containing `PlayerHighs` (list of dictionaries with transformed stat names) and `TotalCount` (the number of filtered highs).
*   **Implementation Logic**:
    1.  **Context Manager**: Uses `with self.get_cursor() as cursor:`.
    2.  **SQL Template Loading**: Loads the highs-specific SQL template from `app/sql_templates/Highs/{sport_code}/{entity}`, appending `_high` to the `stat_period`.
    3.  **Query Formatting**: Formats the query with `player_id`. Logs the query.
    4.  **Query Execution**: Executes the query and fetches results.
    5.  **Data Structuring**:
        *   `column_names = [desc[0].lower() for desc in cursor.description]` converts column descriptions to lowercase before zipping, ensuring consistent dictionary keys.
        *   Converts `rows` to `player_highs` (list of dictionaries).
    6.  **Stat Name Mapping & Filtering**:
        *   Fetches `stat_to_short_name`.
        *   Iterates through `player_highs`, applies the short name transformation to the `'stat'` key, and includes only those records where a short name is found, similar to `execute_player_records`.
    7.  **Total Count**: The `total_count` is updated to the number of `filtered_highs`.
    8.  **Return Value**: A dictionary with `filtered_highs` and `TotalCount`.
    9.  **Error Handling**: Standard `try...except` block.

#### 4.5. Method: `get_player_streaks`

```python
    def get_player_streaks(self, sport_code, entity, player_id, teamcode, stat, stat_value, streak_length, start_date, end_date):
        with self.get_cursor() as cursor:
            try:
                query = SQLTemplateLoader(f'app/sql_templates/Streaks/{sport_code}/{entity}').load_template(f"streaks")
                query = query.format(player_id=f"'{player_id}'", team_code=f"'{teamcode}'", stat=f"{stat.lower()}",
                                     stat_value=stat_value, streak_length=streak_length, start_date=start_date, end_date=end_date)
                logger.info(f"Executing SQL Query:\n{query}")

                cursor.execute(query)
                rows = cursor.fetchall()
                if not rows:
                    return {"PlayerStreaks": [], "TotalCount": 0}
                column_names = [desc[0].lower() for desc in cursor.description]
                player_streaks = [dict(zip(column_names, row)) for row in rows]
                print("player_streaks are", player_streaks) # Debug print

                # The commented out section suggests an intention to apply stat_to_short_name
                # stat_to_short_name = self.fetch_stat_to_short_name(sport_code, entity)
                # for i in range(len(player_streaks)):
                #     player_streaks[i]['stat'] = stat_to_short_name.get(player_streaks[i]['stat'].lower(),
                #                                                      player_streaks[i]['stat'])

                return {"PlayerStreaks": player_streaks, "TotalCount": player_streaks[0]['total_count']}
            except Exception as e:
                logger.error(f"Database Query Error: {e}")
                raise
```

*   **Purpose**: Retrieves performance streaks for a player based on a specific statistic, value, and date range.
*   **Parameters**:
    *   `sport_code` (str): Sport code.
    *   `entity` (str): Entity type.
    *   `player_id` (str): Player identifier.
    *   `teamcode` (str): Team code.
    *   `stat` (str): The specific statistic to check for a streak (e.g., 'points', 'rebounds').
    *   `stat_value`: The threshold or exact value for the stat to qualify for the streak.
    *   `streak_length` (int): The minimum length of the streak.
    *   `start_date` (str): Start date for the streak search (e.g., 'YYYY-MM-DD').
    *   `end_date` (str): End date for the streak search (e.g., 'YYYY-MM-DD').
*   **Returns**:
    *   `dict`: A dictionary containing `PlayerStreaks` (list of dictionaries describing the streaks) and `TotalCount` (the total number of streaks found). Returns `{"PlayerStreaks": [], "TotalCount": 0}` if no streaks are found.
*   **Implementation Logic**:
    1.  **Context Manager**: Uses `with self.get_cursor() as cursor:`.
    2.  **SQL Template Loading**: Loads the streak-specific SQL template from `app/sql_templates/Streaks/{sport_code}/{entity}`.
    3.  **Query Formatting**: Formats the query with `player_id`, `team_code`, `stat` (converted to lowercase), `stat_value`, `streak_length`, `start_date`, and `end_date`. Note the use of `f"'{value}'"` for string parameters to ensure they are properly quoted in the SQL. Logs the query.
    4.  **Query Execution**: Executes the query and fetches results.
    5.  **No Data Check**: If `rows` is empty, returns an empty list and 0 count.
    6.  **Data Structuring**:
        *   `column_names = [desc[0].lower() for desc in cursor.description]` converts column descriptions to lowercase.
        *   Converts `rows` to `player_streaks` (list of dictionaries).
        *   A `print` statement shows the raw fetched streaks for debugging.
    7.  **Stat Name Transformation (Commented Out)**: The commented-out section indicates that, similar to other methods, there was an intention to transform the `stat` field within the streak records using `stat_to_short_name`. This functionality is currently inactive.
    8.  **Return Value**: A dictionary containing `player_streaks` and `total_count` (extracted from the first record, assuming the SQL query provides this).
    9.  **Error Handling**: Standard `try...except` block.

---
