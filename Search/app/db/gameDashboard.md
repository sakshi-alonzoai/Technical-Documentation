# Documentation for `gameDashboard.py`

This document provides a comprehensive technical overview of the `GameDashboardAdapter` Python script. It details its purpose, structure, individual components (functions and classes), and overall operational flow, specifically focusing on its interaction with a PostgreSQL database to retrieve game-related data for a dashboard.

---

## Technical Documentation: `GameDashboardAdapter`

### 1. High-Level Overview

The `GameDashboardAdapter` class is a specialized database adapter designed to interact with a PostgreSQL database to fetch various types of game-related data for a dashboard. It extends `PGSQLAdapter`, suggesting it reuses common database connection and cursor management functionalities. Its primary role is to encapsulate the logic required to query specific game details, player statistics, team statistics, and rosters based on sport, match, and team identifiers. It leverages SQL templates for dynamic query generation and internal configuration mappings to present data consistently.

### 2. Dependencies and Imports

The script relies on several external and internal modules:

*   **`collections.defaultdict`**: Provides `defaultdict` which is a subclass of `dict` that calls a factory function to supply missing values. While imported, it is not explicitly used in the provided code snippet.
*   **`logging`**: Python's standard library for logging events. It's used extensively for debugging, informational messages, and error reporting.
*   **`app.data_config.mappings.mappings_handler.MappingsHandler`**: An internal module responsible for handling data mappings, likely for converting raw database column names to more user-friendly display names, or for defining categories of statistics.
*   **`app.constants.PostgreSQL`**: An internal module that likely contains constants related to PostgreSQL database configuration or operations. While imported, it's not directly referenced in the provided code snippet beyond the implicit use by `PGSQLAdapter`.
*   **`app.sql_templates.sql_template_loader.SQLTemplateLoader`**: An internal module used to load SQL query templates from specified file paths. This allows for cleaner, externalized SQL queries that can be formatted dynamically.
*   **`app.db.pgsql_adapter.PGSQLAdapter`**: The base class for PostgreSQL database interaction. `GameDashboardAdapter` inherits from this, implying it provides core functionalities like connection pooling and cursor management.

### 3. Logging Configuration

```python
logging.basicConfig(level=logging.INFO, format='%(asctime)s - %(levelname)s - %(message)s')
logger = logging.getLogger()
```

*   **`logging.basicConfig(...)`**: Configures the root logger.
    *   `level=logging.INFO`: Sets the minimum logging level to `INFO`. This means messages with severity `INFO`, `WARNING`, `ERROR`, and `CRITICAL` will be processed, while `DEBUG` messages will be ignored.
    *   `format='%(asctime)s - %(levelname)s - %(message)s'`: Defines the format of the log messages. Each log entry will include the timestamp, logging level (e.g., INFO, ERROR), and the actual message.
*   **`logger = logging.getLogger()`**: Retrieves the root logger instance, which is then used throughout the script for logging various events and messages.

### 4. Class `GameDashboardAdapter`

The `GameDashboardAdapter` class extends `PGSQLAdapter`, inheriting its database connection and cursor management capabilities.

#### 4.1. `__init__(self)`

```python
class GameDashboardAdapter(PGSQLAdapter):
    def __init__(self):
        super().__init__()
```

*   **Purpose**: Initializes an instance of the `GameDashboardAdapter`.
*   **Parameters**: None, implicitly `self` refers to the instance being created.
*   **Return Value**: None.
*   **Logic**:
    *   `super().__init__()`: Calls the constructor of the parent class, `PGSQLAdapter`. This ensures that the base database adapter is properly initialized, including establishing database connections or setting up connection pools as defined in `PGSQLAdapter`.

#### 4.2. `game_dashboard(self, sport_code, match_id, home_team_code)`

```python
    def game_dashboard(self, sport_code, match_id, home_team_code):
        with self.get_cursor() as cursor:
            try:
                query = SQLTemplateLoader(f'app/sql_templates/GameDashboard/{sport_code}').load_template(f"basic")
                query = query.format(match_id=match_id, home_team_code=home_team_code)
                logger.info(f"Executing SQL Query:\n{query}")

                cursor.execute(query)
                row = cursor.fetchone()
                logger.info(f"Game Dashboard Data: {row}")
                if not row:
                    return None

                column_names = [desc[0].lower() for desc in cursor.description]
                result = dict(zip(column_names, row))

                # Add categories from config
                categories = self.fetch_game_categories(sport_code)
                result['categories'] = list(categories.keys())

                logger.info(f"Game Dashboard Data:\n{result}")
                return result

            except Exception as e:
                logger.error(f"Database Query Error: {e}")
                raise
        # finally:
        #     self.cursor.close()
```

*   **Purpose**: Fetches basic game dashboard information for a specific match. This typically includes high-level game details and a list of available data categories.
*   **Parameters**:
    *   `sport_code` (str): A unique code identifying the sport (e.g., "MBB", "MFB"). Used to locate sport-specific SQL templates and data mappings.
    *   `match_id` (int): The unique identifier for the match.
    *   `home_team_code` (int): The unique identifier for the home team, used in the SQL query for filtering or specific logic.
*   **Return Value**:
    *   `dict`: A dictionary containing basic game information, with keys representing database column names (converted to lowercase) and an additional `categories` key listing available game categories.
    *   `None`: If no data is found for the given `match_id`.
*   **Logic**:
    1.  **Cursor Acquisition**: `with self.get_cursor() as cursor:` obtains a database cursor using a context manager, ensuring proper resource management.
    2.  **SQL Template Loading**: `SQLTemplateLoader(...)` loads the `basic` SQL template specific to the `sport_code` from `app/sql_templates/GameDashboard/{sport_code}/basic.sql`.
    3.  **Query Formatting**: The loaded `query` string is formatted using `match_id` and `home_team_code`. **Note**: Direct string formatting with `.format()` can be susceptible to SQL injection if `match_id` or `home_team_code` originate from untrusted user input. Parameterized queries are generally safer.
    4.  **Query Execution**: The formatted SQL query is logged (`logger.info`) and then executed using `cursor.execute(query)`.
    5.  **Fetch Result**: `cursor.fetchone()` retrieves the first row of the result set.
    6.  **Handle No Data**: If `row` is `None` (no data found), `None` is returned.
    7.  **Process Result**:
        *   `column_names = [desc[0].lower() for desc in cursor.description]`: Extracts column names from the cursor description and converts them to lowercase.
        *   `result = dict(zip(column_names, row))`: Combines column names with the fetched row data into a dictionary.
    8.  **Fetch Categories**: `self.fetch_game_categories(sport_code)` is called to retrieve a mapping of game categories for the given sport.
    9.  **Add Categories to Result**: The keys (category names) from the fetched categories are added as a list to the `result` dictionary under the key `'categories'`.
    10. **Log and Return**: The final `result` dictionary is logged and returned.
    11. **Error Handling**: A `try...except` block catches any `Exception` during the database operation, logs the error, and re-raises it. The commented-out `finally` block suggests an intention for cursor closing which is now handled by the `with` statement.

#### 4.3. `game_player_stats(self, sport_code, match_id, category, home_team_code)`

```python
    def game_player_stats(self, sport_code, match_id, category, home_team_code):
        with self.get_cursor() as cursor:
            try:
                query = SQLTemplateLoader(f'app/sql_templates/GameDashboard/{sport_code}').load_template(f"player_stats")
                categories = self.fetch_game_categories(sport_code)
                category_data = categories[category]
                stat_columns_with_aliases = []
                stat_columns_raw = []

                for sql_col, display_alias in category_data['stats'].items():
                    stat_columns_raw.append(sql_col)
                    stat_columns_with_aliases.append(f"{sql_col} AS {display_alias}")

                if sport_code in ("MBB", "WBB"):
                    sql_select_cols_raw = ", ".join(stat_columns_raw)
                    sql_select_cols_aliased = ", ".join(stat_columns_with_aliases)

                    query = query.format(
                        match_id=match_id,
                        sql_select_cols_raw=sql_select_cols_raw,
                        sql_select_cols_aliased=sql_select_cols_aliased,
                        home_team_code=home_team_code
                    )
                elif sport_code == "MFB":
                    applicable_positions = category_data['applicable_positions']
                    sql_select_cols_raw = ", ".join(stat_columns_raw)
                    sql_select_cols_aliased = ", ".join(stat_columns_with_aliases)
                    sql_positions_in_clause = ", ".join(f"'{pos}'" for pos in applicable_positions)

                    query = query.format(
                        match_id=match_id,
                        sql_positions_in_clause=sql_positions_in_clause,
                        sql_select_cols_raw=sql_select_cols_raw,
                        sql_select_cols_aliased=sql_select_cols_aliased,
                        home_team_code=home_team_code
                    )

                logger.info(f"Executing SQL Query:\n{query}")
                cursor.execute(query)
                rows = cursor.fetchall()

                if not rows:
                    return None

                column_names = [desc[0].lower() for desc in cursor.description]
                raw_player_stats = [dict(zip(column_names, row)) for row in rows]

                home_team_players = []
                opp_team_players = []
                home_team_name = None
                opp_team_name = None

                for player in raw_player_stats:
                    if int(player.get('playing_team_code')) == int(home_team_code) and home_team_name is None:
                        home_team_name = player.get('playing_team_name')
                    if int(player.get('playing_team_code')) != int(home_team_code) and opp_team_name is None:
                        opp_team_name = player.get('playing_team_name')

                    stats_dict = {}
                    for stat_name in category_data['stats'].values():
                        stat_name_lower = stat_name.lower()
                        if stat_name_lower in player:
                            value = player[stat_name_lower]
                            stats_dict[stat_name] = value

                    player_stats = {
                        'player_name': player.get('player_name', ''),
                        'position': player.get('position', ''),
                        'stats': stats_dict
                    }

                    if int(player.get('playing_team_code')) == int(home_team_code):
                        home_team_players.append(player_stats)
                    else:
                        opp_team_players.append(player_stats)

                if not home_team_name or not opp_team_name:
                    logger.error("Could not determine both home and opp team names")
                    return None

                result = {
                    'home_team_name': home_team_name,
                    'opp_team_name': opp_team_name,
                    'home_team_player_stats': home_team_players,
                    'opp_team_player_stats': opp_team_players
                }

                return result

            except Exception as e:
                logger.error(f"Database Query Error in game_player_stats: {e}")
                raise
        # finally:
        #     self.cursor.close()
```

*   **Purpose**: Retrieves detailed player statistics for a specific match and a given statistical `category`.
*   **Parameters**:
    *   `sport_code` (str): Code for the sport.
    *   `match_id` (int): Unique identifier for the match.
    *   `category` (str): The specific category of player statistics to fetch (e.g., "Scoring", "Passing").
    *   `home_team_code` (int): Identifier for the home team.
*   **Return Value**:
    *   `dict`: A dictionary containing home and opponent team names, and separate lists of player statistics for each team. Each player's stats are structured as `{'player_name': ..., 'position': ..., 'stats': {...}}`.
    *   `None`: If no player data is found or if home/opponent team names cannot be determined.
*   **Logic**:
    1.  **Cursor Acquisition**: Obtains a database cursor.
    2.  **SQL Template Loading**: Loads the `player_stats` SQL template for the specific `sport_code`.
    3.  **Fetch Categories & Data**: `self.fetch_game_categories(sport_code)` is called, and `category_data` for the requested `category` is extracted. This `category_data` contains mappings of SQL column names to display aliases and potentially `applicable_positions` for certain sports.
    4.  **Prepare Stat Columns**: It iterates through `category_data['stats']` to build two lists: `stat_columns_raw` (original SQL column names) and `stat_columns_with_aliases` (SQL column names aliased for display).
    5.  **Conditional Query Formatting**:
        *   **`MBB`, `WBB` (Men's/Women's Basketball)**: The query is formatted with `match_id`, raw stat columns, aliased stat columns, and `home_team_code`.
        *   **`MFB` (Men's Football)**: In addition to the above, it also formats an `sql_positions_in_clause` based on `applicable_positions` from `category_data`. This suggests filtering players by their positions for football statistics.
    6.  **Query Execution**: The formatted SQL query is logged and executed.
    7.  **Fetch Results**: `cursor.fetchall()` retrieves all matching rows.
    8.  **Handle No Data**: If `rows` is empty, `None` is returned.
    9.  **Process Raw Data**: `column_names` are extracted, and `raw_player_stats` is created as a list of dictionaries, where each dictionary represents a player's raw data.
    10. **Separate Teams and Format Stats**:
        *   Initializes empty lists for `home_team_players` and `opp_team_players`, and `None` for team names.
        *   It iterates through `raw_player_stats`:
            *   Determines `home_team_name` and `opp_team_name` based on `playing_team_code` comparison with `home_team_code`.
            *   For each player, `stats_dict` is built by mapping the `display_alias` (from `category_data['stats'].values()`) to the corresponding `player` data, if available.
            *   A `player_stats` dictionary is created containing `player_name`, `position`, and the formatted `stats_dict`.
            *   Players are appended to either `home_team_players` or `opp_team_players` based on `playing_team_code`.
    11. **Validate Team Names**: If `home_team_name` or `opp_team_name` could not be determined, an error is logged, and `None` is returned.
    12. **Construct Result**: A final `result` dictionary is created, grouping players by team with their names.
    13. **Log and Return**: The final `result` dictionary is logged and returned.
    14. **Error Handling**: Catches, logs, and re-raises any `Exception`.

#### 4.4. `game_team_stats(self, sport_code, match_id, home_team_code)`

```python
    def game_team_stats(self, sport_code, match_id, home_team_code):
        with self.get_cursor() as cursor:
            try:
                logger.info(f"Fetching game team stats for sport code: {sport_code}, match id: {match_id}")
                query = SQLTemplateLoader(f'app/sql_templates/GameDashboard/{sport_code}').load_template(f"team_stats")
                stats = self.fetch_stat_mapping(sport_code, "Team",return_full_name=True)
                sql_select_cols_raw = ", ".join(stats.keys())
                logger.info(f"SQL Select Columns Raw: {sql_select_cols_raw}")
                query = query.format(match_id=match_id, sql_select_cols_raw=sql_select_cols_raw, home_team_code=home_team_code)
                logger.info(f"Executing SQL Query:\n{query}")

                cursor.execute(query)
                rows = cursor.fetchall()
                if not rows:
                    return None

                # Process rows into the required format
                column_names = [desc[0].lower() for desc in cursor.description]
                raw_stats = [dict(zip(column_names, row)) for row in rows]

                # Separate home and away team stats
                home_team_stats = []
                opp_team_stats = []
                home_team_name = None
                opp_team_name = None

                # logger.info(f"Raw Stats: {raw_stats}")
                for stat_row in raw_stats:
                    if int(stat_row.get('team_code')) == int(home_team_code):
                        home_team_name = stat_row.get('team_name')
                        stats_dict = {stats[k]: float(v) for k, v in stat_row.items()
                                    if k not in ['team_name', 'team_code'] and isinstance(v, (int, float))}
                        home_team_stats.append({'stats': stats_dict})
                    else:
                        opp_team_name = stat_row.get('team_name')
                        stats_dict = {stats[k]: float(v) for k, v in stat_row.items()
                                    if k not in ['team_name', 'team_code'] and isinstance(v, (int, float))}
                        opp_team_stats.append({'stats': stats_dict})

                if not home_team_name or not opp_team_name:
                    logger.error("Could not determine both home and away team names")
                    return None

                result = {
                    'home_team_name': home_team_name,
                    'opp_team_name': opp_team_name,
                    'home_team_stats': home_team_stats,
                    'opp_team_stats': opp_team_stats
                }

                return result

            except Exception as e:
                logger.error(f"Database Query Error: {e}")
                raise
        # finally:
        #     self.cursor.close()
```

*   **Purpose**: Retrieves detailed team statistics for a specific match.
*   **Parameters**:
    *   `sport_code` (str): Code for the sport.
    *   `match_id` (int): Unique identifier for the match.
    *   `home_team_code` (int): Identifier for the home team.
*   **Return Value**:
    *   `dict`: A dictionary containing home and opponent team names, and separate lists of team statistics for each team. Each team's stats are structured as `{'stats': {...}}`.
    *   `None`: If no team data is found or if home/opponent team names cannot be determined.
*   **Logic**:
    1.  **Cursor Acquisition**: Obtains a database cursor.
    2.  **Log Info**: Logs that team stats fetching has started.
    3.  **SQL Template Loading**: Loads the `team_stats` SQL template for the specific `sport_code`.
    4.  **Fetch Stat Mapping**: `self.fetch_stat_mapping(sport_code, "Team", return_full_name=True)` is called to get a mapping of database column names to their full display names for "Team" level statistics.
    5.  **Prepare Stat Columns**: `sql_select_cols_raw` is created by joining the keys (database column names) from the fetched `stats` mapping.
    6.  **Query Formatting**: The query is formatted with `match_id`, the raw SQL select columns, and `home_team_code`.
    7.  **Query Execution**: The formatted SQL query is logged and executed.
    8.  **Fetch Results**: `cursor.fetchall()` retrieves all matching rows.
    9.  **Handle No Data**: If `rows` is empty, `None` is returned.
    10. **Process Raw Data**: `column_names` are extracted, and `raw_stats` is created as a list of dictionaries.
    11. **Separate Teams and Format Stats**:
        *   Initializes empty lists for `home_team_stats` and `opp_team_stats`, and `None` for team names.
        *   Iterates through each `stat_row` in `raw_stats`:
            *   Determines and sets `home_team_name` or `opp_team_name` based on `team_code` comparison with `home_team_code`.
            *   For each team, `stats_dict` is built. It iterates over the items in the `stat_row`. If a key is not 'team_name' or 'team_code' and its value is an `int` or `float`, it uses the `stats` mapping to convert the database column name (`k`) to its display name (`stats[k]`) and stores the value (cast to `float`).
            *   The formatted `stats_dict` is appended to the appropriate team's list wrapped in `{'stats': ...}`.
    12. **Validate Team Names**: If `home_team_name` or `opp_team_name` could not be determined, an error is logged, and `None` is returned.
    13. **Construct Result**: A final `result` dictionary is created, grouping team statistics by home/opponent with their names.
    14. **Log and Return**: The final `result` dictionary is logged and returned.
    15. **Error Handling**: Catches, logs, and re-raises any `Exception`.

#### 4.5. `game_rosters(self, sport_code, match_id, home_team_code, show_starters_only)`

```python
    def game_rosters(self,sport_code,match_id,home_team_code,show_starters_only):
        with self.get_cursor() as cursor:
            try:
                query = SQLTemplateLoader(f'app/sql_templates/GameDashboard/{sport_code}').load_template(f"rosters")
                query = query.format(match_id=match_id, show_starters_only=show_starters_only)
                logger.info(f"Executing SQL Query:\n{query}")

                cursor.execute(query)
                rows = cursor.fetchall()
                column_names = [desc[0].lower() for desc in cursor.description]
                rosters = [dict(zip(column_names, row)) for row in rows]
                home_team_rosters = []
                opp_team_rosters = []
                for roster in rosters:
                    if int(roster.get('playing_team_code')) == int(home_team_code):
                        home_team_rosters.append(roster)
                    else:
                        opp_team_rosters.append(roster)
                print("its executing here , roster")
                return {
                    'home_team_rosters': home_team_rosters,
                    'opp_team_rosters': opp_team_rosters
                }
            except Exception as e:
                logger.error(f"Database Query Error: {e}")
                raise
        # finally:
        #     cursor.close()
```

*   **Purpose**: Fetches the roster for a specific game, optionally filtering for starters only.
*   **Parameters**:
    *   `sport_code` (str): Code for the sport.
    *   `match_id` (int): Unique identifier for the match.
    *   `home_team_code` (int): Identifier for the home team.
    *   `show_starters_only` (bool/int): A flag (likely 0 or 1, or boolean) indicating whether to retrieve only starters (`True`/`1`) or all players (`False`/`0`). This is directly inserted into the SQL query.
*   **Return Value**:
    *   `dict`: A dictionary containing two lists: `home_team_rosters` and `opp_team_rosters`, each holding dictionaries representing player roster details.
*   **Logic**:
    1.  **Cursor Acquisition**: Obtains a database cursor.
    2.  **SQL Template Loading**: Loads the `rosters` SQL template for the specific `sport_code`.
    3.  **Query Formatting**: The query is formatted with `match_id` and `show_starters_only`.
    4.  **Query Execution**: The formatted SQL query is logged and executed.
    5.  **Fetch Results**: `cursor.fetchall()` retrieves all matching rows.
    6.  **Process Raw Data**: `column_names` are extracted, and `rosters` is created as a list of dictionaries.
    7.  **Separate Teams**:
        *   Initializes empty lists for `home_team_rosters` and `opp_team_rosters`.
        *   Iterates through `rosters`, appending each player's roster data to the appropriate team list based on `playing_team_code` compared with `home_team_code`.
    8.  **Print Debug Message**: `print("its executing here , roster")` is a debug print statement that should ideally be replaced with a `logger.debug` or `logger.info` call in a production environment.
    9.  **Construct and Return Result**: A dictionary containing `home_team_rosters` and `opp_team_rosters` is returned.
    10. **Error Handling**: Catches, logs, and re-raises any `Exception`.

### 5. Main Execution Flow (`if __name__ == "__main__":` block)

The provided script defines a class but does not include an `if __name__ == "__main__":` block. This implies that `GameDashboardAdapter` is intended to be imported as a module into other parts of the `app` (e.g., an API endpoint, a service layer) where its methods will be called to interact with the database.

### 6. Important Implementation Details and Dependencies

*   **Database Adapter Inheritance**: `GameDashboardAdapter` inherits from `PGSQLAdapter`. This means it implicitly relies on `PGSQLAdapter` to manage database connections, provide cursors (`get_cursor`), and potentially handle connection pooling or transaction management.
*   **SQL Template-Based Queries**: Queries are not hardcoded within the Python functions. Instead, they are loaded dynamically from `.sql` files using `SQLTemplateLoader`. This promotes separation of concerns, making SQL queries easier to manage, test, and update without modifying Python code. The structure `app/sql_templates/GameDashboard/{sport_code}/{template_name}.sql` is implied.
*   **Configuration-Driven Data Retrieval**: The methods `fetch_game_categories` and `fetch_stat_mapping` (likely from `MappingsHandler`) are crucial. They indicate that the structure and display of statistics (e.g., which columns to select, how to alias them, which categories exist) are driven by external configuration or data mappings, allowing for flexible and sport-specific data presentation without code changes.
*   **Dynamic Column Selection and Aliasing**: For player and team statistics, the script dynamically constructs the `SELECT` clause of the SQL query based on the configured stat mappings. This makes the queries highly adaptable to different statistical requirements.
*   **Data Transformation**: Raw results from the database (tuples) are consistently converted into lists of dictionaries, with column names extracted from `cursor.description` and converted to lowercase. This standardizes the output format.
*   **Separation of Home/Opponent Data**: All methods that return player or team-level data explicitly separate results into 'home' and 'opponent' categories based on the `home_team_code` provided.
*   **SQL Injection Vulnerability**: The use of `query.format(...)` to insert `match_id`, `home_team_code`, `sql_select_cols_raw`, `sql_select_cols_aliased`, `sql_positions_in_clause`, and `show_starters_only` directly into the SQL string can lead to SQL injection vulnerabilities if these values are not strictly controlled and sanitized. For production applications, it is strongly recommended to use parameterized queries (`cursor.execute(query, (param1, param2))`) where possible, especially for `match_id`, `home_team_code`, and `show_starters_only`. Dynamic column names and aliases often require string formatting, but careful validation of input for these elements is critical.
*   **Error Handling**: A `try...except Exception` block is consistently used across all methods to catch and log any database-related errors, ensuring that exceptions are not silently swallowed but are recorded for debugging. The `raise` statement then propagates the exception further up the call stack.
*   **Implicit `MappingsHandler` Methods**: The `game_dashboard`, `game_player_stats`, and `game_team_stats` methods call `self.fetch_game_categories(sport_code)` and `self.fetch_stat_mapping(sport_code, "Team", return_full_name=True)`. These methods are not defined within `GameDashboardAdapter` itself, suggesting they are either inherited from `PGSQLAdapter` or, more likely, are methods implemented in `MappingsHandler` and then accessed via the `self` object due to some composition or proxy pattern not shown in the provided code (e.g., `PGSQLAdapter` might instantiate `MappingsHandler` and expose its methods). Given the import `from app.data_config.mappings.mappings_handler import MappingsHandler`, it is probable that `PGSQLAdapter` (or an intermediary) manages an instance of `MappingsHandler` and exposes its relevant methods.
