# Documentation for `sql_builder.py`

This document provides a comprehensive technical overview of the `SQLBuilder` Python class and its associated components. It details the script's purpose, its internal mechanisms, and how it interacts with other modules and databases to dynamically convert AQL (Athlyte Query Language) queries into executable SQL.

---

## Technical Documentation: `SQLBuilder`

### 1. High-Level Overview

The `SQLBuilder` class is designed to act as an intermediary layer, translating a parsed Athlyte Query Language (AQL) output into a corresponding SQL query. Its primary function is to construct dynamic SQL statements for querying a PostgreSQL database based on user-specified sports, entities (player or team), statistical periods, and various query types (e.g., basic, min, max, last time when, streak). It leverages mapping configurations, SQL templates, and dynamic filtering/sorting logic to generate efficient and accurate database queries.

The class abstracts away the complexities of SQL generation, allowing the application to work with a higher-level AQL representation. It handles tasks such as determining the correct database table, selecting appropriate columns, building complex `WHERE` clauses, and applying sorting and pagination.

### 2. Dependencies and Imports

This script relies on several external and internal Python libraries and modules:

*   **`json`**: Standard library for working with JSON data. (Not directly used in the provided code snippet, but often useful for data serialization/deserialization).
*   **`os`**: Standard library for interacting with the operating system, e.g., environment variables, file paths. (Not directly used in the provided code snippet).
*   **`sys`**: Standard library for system-specific parameters and functions. (Not directly used in the provided code snippet).
*   **`psycopg2`**: PostgreSQL adapter for Python. Used by `PostgresHandler` for database connectivity.
*   **`re`**: Standard library for regular expressions, used for pattern matching and extraction (e.g., stat column names).
*   **`pymongo`**: Python driver for MongoDB. Although `_load_stat_mapping` originally contained MongoDB logic, it is currently commented out and not actively used.
*   **`decouple.config`**: Used for managing environment variables (e.g., database connection strings) from `.env` files or system environment.
*   **`app.data_config.mappings.mappings_handler`**: An internal module responsible for handling various data mappings, such as table names, field mappings, and filter configurations, based on sport, entity, and stat period.
*   **`app.db.pgsql_adapter`**: An internal module likely providing a base adapter for PostgreSQL database interactions. (Not directly instantiated in this class).
*   **`app.sql_templates.sql_template_loader`**: An internal module for loading pre-defined SQL query templates, which the `SQLBuilder` class then formats with dynamic values.
*   **`app.constants`**: An internal module defining various constants used throughout the application, such as `QueryTypes`, `SortingFields`, `SortOrders`, and `QUALIFIERS_MAPPING`.
*   **`app.db.postgres_handler`**: An internal module providing an interface for executing queries and fetching data from a PostgreSQL database.

### 3. Class `SQLBuilder`

This class is the core component responsible for dynamic SQL generation.

```python
class SQLBuilder:
    """
    Converts AQL (Athlyte Query Language) into SQL dynamically.
    """
```

**Docstring**: Explains the primary purpose of the class: converting AQL into SQL dynamically.

#### 3.1. `__init__(self)`

```python
    def __init__(self):
        self.mappings_handler = MappingsHandler()
        self.sql_loader = SQLTemplateLoader()
        self.stat_mapping = self._load_stat_mapping()
```

**Purpose**: The constructor initializes the `SQLBuilder` instance by setting up necessary helper objects and loading initial data.

**Implementation Logic**:
*   `self.mappings_handler = MappingsHandler()`: An instance of `MappingsHandler` is created. This object is crucial for retrieving database table names, column mappings, and filter definitions based on various query parameters.
*   `self.sql_loader = SQLTemplateLoader()`: An instance of `SQLTemplateLoader` is created. This object is responsible for loading pre-defined SQL query templates from external files.
*   `self.stat_mapping = self._load_stat_mapping()`: The `_load_stat_mapping` private method is called to fetch and store the mapping between "GeniusLabel" (a universal stat identifier) and "AthlyteLabel" (the corresponding short label used in Athlyte) from the database. This mapping is stored in `self.stat_mapping`.

#### 3.2. `_load_stat_mapping(self)`

```python
    def _load_stat_mapping(self):
        """Fetches GeniusLabel → AthlyteShortLabel mapping from PostgreSQL."""
        # Comment out the MongoDB code
        # try:
        #     collection = db['stat_mapping']
        #     return {
        #         entry['GeniusLabel'].lower(): entry['AthlyteShortLabel']
        #         for entry in collection.find()
        #     }
        # except Exception as e:
        #     print(f"Error fetching stat mapping: {e}")
        #     return {}
        try:
            postgres_handler = PostgresHandler()
            query = 'SELECT "GeniusLabel", "AthlyteLabel" FROM stat_mapping;'
            rows = postgres_handler.fetch_all(query)
            print(f"stat_mapping fetched from Postgres")
            postgres_handler.close()

            return {
                row[0].lower(): row[1]
                for row in rows
            }
        except Exception as e:
            print(f"Error fetching stat mapping from Postgres: {e}")
            return {}
```

**Purpose**: This private method is responsible for retrieving a mapping of statistical labels from the database. Specifically, it maps a generic "GeniusLabel" (likely from an external data source) to an "AthlyteLabel" (the label used within the Athlyte system).

**Implementation Logic**:
*   **Commented-out MongoDB Code**: The code block starting with `try: collection = db['stat_mapping']...` indicates that an earlier version of this method used MongoDB to fetch stat mappings. This code is now commented out, signifying a shift to PostgreSQL.
*   **PostgreSQL Fetching**:
    *   `postgres_handler = PostgresHandler()`: An instance of `PostgresHandler` is created to interact with the PostgreSQL database.
    *   `query = 'SELECT "GeniusLabel", "AthlyteLabel" FROM stat_mapping;'`: A SQL query is constructed to select `GeniusLabel` and `AthlyteLabel` columns from the `stat_mapping` table.
    *   `rows = postgres_handler.fetch_all(query)`: The `fetch_all` method of `PostgresHandler` is called to execute the query and retrieve all matching rows.
    *   `print(f"stat_mapping fetched from Postgres")`: A confirmation message is printed to the console.
    *   `postgres_handler.close()`: The database connection opened by `postgres_handler` is explicitly closed to release resources.
    *   `return {row[0].lower(): row[1] for row in rows}`: The fetched rows are transformed into a dictionary. The `GeniusLabel` (first element `row[0]`) is converted to lowercase and used as the key, and the `AthlyteLabel` (second element `row[1]`) is used as the value.
*   **Error Handling**: A `try-except` block catches any `Exception` that occurs during the database interaction. If an error occurs, an error message is printed, and an empty dictionary `{}` is returned, ensuring the application doesn't crash but operates without stat mappings.

**Returns**:
*   `dict`: A dictionary mapping lowercase `GeniusLabel` to `AthlyteLabel`.
*   `{}`: An empty dictionary if an error occurs during fetching.

#### 3.3. `convert_aql_to_sql(self, aql_output, sport_code, entity, filters, query_type, team_code, limit=10, offset=0, sort_by=None, sort_order=None)`

```python
    def convert_aql_to_sql(self, aql_output, sport_code, entity,filters, query_type, team_code, limit=10, offset=0, sort_by=None, sort_order=None):
        """
        Converts AQL (Athlyte Query Language) query to SQL based on query type.

        Args:
            aql_output (dict): The parsed AQL query output
            sport_code (str): Code identifying the sport (e.g. 'MFB', 'MBB')
            entity (str): Type of entity ('player' or 'team')
            filters (dict): Query filters to apply
            query_type (str): Type of query to execute
            team_code (str): Team identifier code
            limit (int, optional): Number of results to return. Defaults to 10
            offset (int, optional): Number of results to skip. Defaults to 0
            sort_by (str, optional): Column to sort results by. Defaults to None
            sort_order (str, optional): Sort direction ('ASC' or 'DESC'). Defaults to None

        Returns:
            tuple: Contains generated SQL query, count query, selected columns and stat columns

        Raises:
            ValueError: If no table mapping is found for the given parameters
        """
        stat_period = aql_output["stat_period"].lower()
        table_name = self.mappings_handler.get_table_name(sport_code, entity, stat_period)
        if not table_name:
            raise ValueError(f"No table mapping found for sport_code={sport_code}, entity={entity}, stat_period={stat_period}")
        
        selected_columns = self._get_selected_columns(sport_code ,aql_output, entity, stat_period, query_type)
        print("selected columns are:", selected_columns)

        conditions = aql_output.get(SortingFields.CONDITIONS.value, "").strip()
        print("Conditions are:", conditions)
        qualifiers = aql_output.get(SortingFields.QUALIFIERS.value, {})
        periods = aql_output.get(SortingFields.PERIODS.value, {})
        period_type = aql_output.get(SortingFields.PERIOD_TYPE.value, {})
        # if periods is not None:
        #     selected_columns += ["period_number", "period_type"]

        where_clause = self._build_where_clause(conditions, qualifiers, filters, query_type, periods, period_type)

        query_parsers = {
            QueryTypes.BASIC.value: self._parse_basic_query,
            QueryTypes.MIN.value: self._parse_min_query,
            QueryTypes.MAX.value: self._parse_max_query,
            QueryTypes.LTW.value: self._parse_ltw_query,
            QueryTypes.STREAK.value: self._parse_streak_query
        }
        
        if query_type.lower() not in query_parsers:
            raise ValueError(f"Query type '{query_type}' not implemented.")
        
        parser = query_parsers[query_type.lower()]
        return parser(table_name, selected_columns, where_clause, aql_output, limit, offset, sort_by, sort_order, entity, team_code, sport_code)
```

**Purpose**: This is the main public method of the `SQLBuilder` class. It orchestrates the entire AQL-to-SQL conversion process, taking the parsed AQL output and other query parameters to produce a complete SQL query.

**Parameters**:
*   `aql_output` (`dict`): A dictionary containing the parsed Athlyte Query Language (AQL) output, including elements like `stat_period`, `conditions`, `qualifiers`, `periods`, `period_type`, etc.
*   `sport_code` (`str`): A string code representing the sport (e.g., 'MFB' for Men's Football, 'MBB' for Men's Basketball).
*   `entity` (`str`): The type of entity for the query, either 'player' or 'team'.
*   `filters` (`dict`): A dictionary of additional filters to apply to the query.
*   `query_type` (`str`): The specific type of query to execute (e.g., 'basic', 'min', 'max', 'ltw', 'streak'). These correspond to `QueryTypes` constants.
*   `team_code` (`str`): An identifier for the team relevant to the query.
*   `limit` (`int`, optional): The maximum number of results to return. Defaults to `10`.
*   `offset` (`int`, optional): The number of results to skip from the beginning. Defaults to `0`.
*   `sort_by` (`str`, optional): The column name by which to sort the results. Defaults to `None`.
*   `sort_order` (`str`, optional): The direction of sorting, either 'ASC' (ascending) or 'DESC' (descending). Defaults to `None`.

**Returns**:
*   `tuple`: A tuple containing `(generated_sql_query, selected_columns, stat_columns_used)`.

**Raises**:
*   `ValueError`: If no database table mapping is found for the given `sport_code`, `entity`, and `stat_period`, or if the `query_type` is not recognized.

**Implementation Logic**:
1.  **Extract `stat_period`**: `stat_period` is extracted from `aql_output` and converted to lowercase.
2.  **Get Table Name**: `self.mappings_handler.get_table_name()` is called to determine the appropriate database table name based on `sport_code`, `entity`, and `stat_period`.
3.  **Validate Table Name**: If `table_name` is `None` (no mapping found), a `ValueError` is raised.
4.  **Get Selected Columns**: `self._get_selected_columns()` is called to determine the list of columns to be selected in the SQL query. This method considers the `sport_code`, `aql_output`, `entity`, `stat_period`, and `query_type`.
5.  **Extract AQL Components**: `conditions`, `qualifiers`, `periods`, and `period_type` are extracted from `aql_output`. `conditions` is stripped of leading/trailing whitespace.
6.  **Build WHERE Clause**: `self._build_where_clause()` is called to construct the `WHERE` clause of the SQL query by combining `conditions`, `qualifiers`, `filters`, `query_type`, `periods`, and `period_type`.
7.  **Query Type Dispatch**:
    *   A dictionary `query_parsers` maps `QueryTypes` (from `app.constants`) to their corresponding private parsing methods (`_parse_basic_query`, `_parse_min_query`, etc.).
    *   The `query_type` is checked against the keys in `query_parsers`. If it's not found, a `ValueError` is raised.
    *   The appropriate parser function (`parser`) is retrieved from the `query_parsers` dictionary.
8.  **Execute Parser**: The selected `parser` function is called with all relevant parameters (`table_name`, `selected_columns`, `where_clause`, `aql_output`, `limit`, `offset`, `sort_by`, `sort_order`, `entity`, `team_code`, `sport_code`). This delegated call will ultimately lead to `_generate_query`.
9.  **Return Value**: The result from the specific parser (which is actually `_generate_query`) is returned.

#### 3.4. `_get_selected_columns(self, sport_code, aql_output, entity, stat_period, query_type)`

```python
    def _get_selected_columns(self, sport_code, aql_output, entity, stat_period, query_type):
        """
        Determines which columns should be selected based on query type and conditions.

        Args:
            sport_code (str): Code identifying the sport
            aql_output (dict): The parsed AQL query output
            entity (str): Type of entity (player/team)
            stat_period (str): Statistical period (game/season)
            query_type (str): Type of query being executed

        Returns:
            list: Column names to select in the query

        Raises:
            ValueError: If no field mappings are found for the given parameters
        """
        selected_columns = self.mappings_handler.get_fields_for_query(sport_code, entity, stat_period, query_type)
        if not selected_columns:
            raise ValueError(f"No field mappings found for entity={entity}, stat_period={stat_period}, query_type={query_type}")

        print(f"Selected columns: {selected_columns}")
        if ('periods' not in aql_output) or (aql_output["periods"] in [None, [], [0]]):
            print("I am deleting period_number in selected columns")
            # If period number is not in qualifiers, then we default to '0'. In that case, we dont need to show this to the user
            if "period_number" in selected_columns:
                del selected_columns[selected_columns.index("period_number")]

        sel_col_pat = re.compile("\s+AND|and|OR|or\s+")
        col_extr_pat = re.compile("s_[a-z0-9_]+")
        
        conditions = aql_output.get(SortingFields.CONDITIONS.value, "").strip()
        stat_columns = {
            col_extr_pat.findall(stat)[0]
            for stat in sel_col_pat.split(conditions)
            if col_extr_pat.findall(stat) != []
        }
        print(f"Stat Columns: {stat_columns}")
        return list(set(selected_columns) | stat_columns)
```

**Purpose**: This private method determines the final set of columns that should be selected in the SQL query. It combines general fields with specific statistical columns derived from the AQL conditions.

**Parameters**:
*   `sport_code` (`str`): Code identifying the sport.
*   `aql_output` (`dict`): The parsed AQL query output.
*   `entity` (`str`): Type of entity ('player' or 'team').
*   `stat_period` (`str`): Statistical period ('game' or 'season').
*   `query_type` (`str`): Type of query being executed.

**Returns**:
*   `list`: A list of unique column names to be included in the SQL `SELECT` clause.

**Raises**:
*   `ValueError`: If `mappings_handler` does not return any field mappings for the provided parameters.

**Implementation Logic**:
1.  **Initial Column Retrieval**: `self.mappings_handler.get_fields_for_query()` is called to get a base list of selected columns based on the `sport_code`, `entity`, `stat_period`, and `query_type`.
2.  **Validate Mappings**: If `selected_columns` is empty, a `ValueError` is raised, indicating a configuration issue.
3.  **Handle `period_number`**:
    *   Checks if `aql_output` explicitly specifies `periods`, or if `periods` is `None`, an empty list `[]`, or `[0]`.
    *   If `period_number` is not explicitly requested (i.e., defaults to '0' or is absent), and if "period_number" is present in the `selected_columns` list, it is removed. This prevents showing `period_number` to the user when it's not relevant or explicitly requested.
4.  **Extract Stat Columns from Conditions**:
    *   `sel_col_pat = re.compile("\s+AND|and|OR|or\s+")`: A regular expression is compiled to split the `conditions` string by logical operators (`AND`, `OR`).
    *   `col_extr_pat = re.compile("s_[a-z0-9_]+")`: Another regex is compiled to extract potential stat column names, which are assumed to start with "s\_" followed by alphanumeric characters and underscores.
    *   `conditions = aql_output.get(SortingFields.CONDITIONS.value, "").strip()`: The raw conditions string is retrieved from `aql_output`.
    *   `stat_columns = { ... }`: A set comprehension is used to:
        *   Split `conditions` by `sel_col_pat`.
        *   For each resulting part (`stat`), it attempts to find stat column names using `col_extr_pat`.
        *   The first found stat column (if any) is added to the `stat_columns` set. Using a set ensures uniqueness.
5.  **Combine and Return**: The `selected_columns` list and the `stat_columns` set are combined into a single set (to ensure uniqueness) and then converted back into a list. This final list of unique column names is returned.

#### 3.5. `_build_where_clause(self, conditions, qualifiers, filters, query_type, periods, period_type)`

```python
    def _build_where_clause(self, conditions, qualifiers, filters, query_type, periods, period_type):
        """
        Builds SQL WHERE clause by combining conditions, qualifiers and filters.

        Args:
            conditions (str): Raw conditions from AQL query
            qualifiers (dict): Query qualifiers to apply
            filters (dict): Additional filters to apply
            query_type (str): Type of query being executed

        Returns:
            str: Complete WHERE clause for SQL query
        """
        where_conditions = set()

        if conditions and query_type != QueryTypes.STREAK.value:
            # Do not add conditions into where clause for streaks
            if any(op in conditions for op in [">", "<", "=", "!=", ">=", "<="]):
                where_conditions.add(f"({conditions.strip()})")
            elif query_type not in [QueryTypes.MIN.value, QueryTypes.MAX.value]:
                if not conditions.endswith("IS NOT NULL"):
                    where_conditions.add(f"({conditions.strip()} IS NOT NULL)")
                else:
                    where_conditions.add(f"({conditions.strip()})")

        for key, value in qualifiers.items():
            key = QUALIFIERS_MAPPING.get(key, key)
            if isinstance(value, list):
                formatted_value = "', '".join(value)
                where_conditions.add(f"{key} IN ('{formatted_value}')")
            else:
                where_conditions.add(f"{key} = '{value}'")

        filter_conditions = self._format_filters(filters)
        print(f"Filters: {filter_conditions}")
        where_conditions.update(filter_conditions)

        period_conditions = []
        ## Change this in R2: Check if the period is a quarter"
        period_number_condition = "(period_number='0' OR period_number IS NULL)"
        #period_type_condition = "period_type = 'REGULAR'"

        if period_type is not None:
            period_type = period_type.upper()
            period_type_condition = f"period_type = '{period_type}'"
            period_conditions += [period_type_condition]

        if periods is not None:
            if periods == 'ALL':
                periods = [1, 2, 3, 4]
            period_string = ",".join([f"'{x}'" for x in periods])
            period_number_condition = f"period_number IN ({period_string})"


        period_conditions += [period_number_condition]
        where_conditions.update(period_conditions)

        # value_conditions = [""]

        return " AND ".join(where_conditions) if where_conditions else ""
```

**Purpose**: This private method constructs the `WHERE` clause of the SQL query by aggregating conditions from various sources: raw AQL conditions, qualifiers, general filters, and period-specific criteria.

**Parameters**:
*   `conditions` (`str`): The raw string of conditions from the AQL query (e.g., "s_points > 10").
*   `qualifiers` (`dict`): A dictionary of qualifiers from the AQL query (e.g., `{'sport_code': 'MBB'}`).
*   `filters` (`dict`): A dictionary of additional filters provided separately.
*   `query_type` (`str`): The type of query (e.g., 'basic', 'streak').
*   `periods` (`list` or `str`): A list of period numbers or the string 'ALL' if specific periods are requested.
*   `period_type` (`str`): The type of period (e.g., 'REGULAR', 'OVERTIME').

**Returns**:
*   `str`: A concatenated string representing the full `WHERE` clause. Returns an empty string if no conditions are added.

**Implementation Logic**:
1.  **Initialize `where_conditions`**: An empty set `where_conditions` is used to store individual clause parts, ensuring uniqueness and order independence.
2.  **Process `conditions`**:
    *   If `conditions` is not empty and the `query_type` is not `STREAK` (streaks are handled differently for conditions):
        *   It checks if `conditions` contains any comparison operators (`>`, `<`, `=`, `!=`, `>=`, `<=`). If so, the `conditions` are added directly, wrapped in parentheses.
        *   If no comparison operators are found and the `query_type` is not `MIN` or `MAX` (which imply existence of a stat), it defaults to ensuring the condition is `IS NOT NULL`. If the condition already ends with "IS NOT NULL", it's added as is.
3.  **Process `qualifiers`**:
    *   Iterates through each `key`, `value` pair in `qualifiers`.
    *   The `key` is mapped using `QUALIFIERS_MAPPING` (from `app.constants`) to translate AQL-friendly qualifier names to their corresponding database column names.
    *   If `value` is a list, an `IN` clause is generated (e.g., `column IN ('val1', 'val2')`).
    *   Otherwise, an equality clause is generated (e.g., `column = 'value'`).
    *   Each generated condition is added to `where_conditions`.
4.  **Process `filters`**:
    *   `filter_conditions = self._format_filters(filters)`: Calls the helper method `_format_filters` to convert the `filters` dictionary into a list of SQL-ready conditions.
    *   `where_conditions.update(filter_conditions)`: Adds these formatted filter conditions to the `where_conditions` set.
5.  **Process `periods` and `period_type`**:
    *   `period_conditions = []`: An empty list to store period-related conditions.
    *   `period_number_condition = "(period_number='0' OR period_number IS NULL)"`: Initializes a default condition for `period_number`.
    *   If `period_type` is provided, it's converted to uppercase, and a condition like `period_type = 'REGULAR'` is added.
    *   If `periods` are provided:
        *   If `periods` is 'ALL', it's expanded to `[1, 2, 3, 4]`.
        *   The list of periods is formatted into an `IN` clause string (e.g., `period_number IN ('1', '2')`).
    *   The determined `period_number_condition` is added to `period_conditions`.
    *   `where_conditions.update(period_conditions)`: Adds all period conditions to the `where_conditions` set.
6.  **Join and Return**: Finally, all unique conditions in `where_conditions` are joined together with " AND ". If `where_conditions` is empty, an empty string is returned.

#### 3.6. `_format_filters(self, filters)`

```python
    def _format_filters(self, filters):
        """
        Formats filter conditions for SQL WHERE clause.

        Args:
            filters (dict): Dictionary of filters to format

        Returns:
            list: Formatted filter conditions ready for SQL
        """
        filter_conditions = []
        filter_mappings = self.mappings_handler.load_filter_mappings()
        for key, value in filters.items():
            if key in filter_mappings:
                column_name, operator = filter_mappings[key]
                formatted_value = "', '".join(value) if isinstance(value, list) else value
                if formatted_value not in ['NULL', 'NOT NULL']:
                    formatted_value = f"'{formatted_value}'"
                filter_conditions.append(f"{column_name} {operator} ({formatted_value})" if operator == "IN" else f"{column_name} {operator} {formatted_value}")
        return filter_conditions
```

**Purpose**: This private helper method converts a dictionary of generic filters into a list of SQL-formatted condition strings suitable for a `WHERE` clause.

**Parameters**:
*   `filters` (`dict`): A dictionary where keys are filter names and values are the filter criteria (can be a single value or a list).

**Returns**:
*   `list`: A list of strings, each representing a SQL filter condition (e.g., `"game_date > '2023-01-01'"`).

**Implementation Logic**:
1.  **Initialize `filter_conditions`**: An empty list to store the formatted SQL conditions.
2.  **Load Filter Mappings**: `filter_mappings = self.mappings_handler.load_filter_mappings()` retrieves a mapping from generic filter keys to their corresponding database column names and SQL operators (e.g., `{'date': ('game_date', '>')}`).
3.  **Iterate and Format**:
    *   For each `key`, `value` pair in the input `filters` dictionary:
        *   Checks if the `key` exists in `filter_mappings`.
        *   If a mapping exists, `column_name` and `operator` are extracted.
        *   `formatted_value` is prepared:
            *   If `value` is a list, it's joined into a comma-separated, single-quoted string (e.g., `'val1', 'val2'`).
            *   Otherwise, it's used directly.
            *   Unless `formatted_value` is `NULL` or `NOT NULL`, it's wrapped in single quotes (`'value'`).
        *   A SQL condition string is constructed:
            *   If the `operator` is "IN", the `formatted_value` is enclosed in parentheses (e.g., `column IN ('val1', 'val2')`).
            *   Otherwise, a direct comparison is used (e.g., `column = 'value'`).
        *   This condition string is appended to `filter_conditions`.
4.  **Return**: The list of formatted `filter_conditions` is returned.

#### 3.7. `_parse_query(self, query_type, table_name, selected_columns, where_clause, aql_output, limit, offset, sort_by, sort_order, entity, team_code, sport_code)`

```python
    def _parse_query(self, query_type, table_name, selected_columns, where_clause, aql_output, limit, offset, sort_by, sort_order, entity, team_code, sport_code):
        """
        Generic query parser that handles all query types.
        
        Args:
            query_type (str): Type of query (basic, min, max, ltw, streak)
            table_name (str): Name of the database table
            selected_columns (list): Columns to select
            where_clause (str): WHERE clause conditions
            aql_output (dict): Output from AQL parsing
            limit (int): Number of results to return
            offset (int): Number of results to skip
            sort_by (str): Column to sort by
            sort_order (str): Sort order (ASC/DESC)
            entity (str): Entity type (player/team)
            team_code (str): Team identifier code
            sport_code (str): Sport identifier code
            
        Returns:
            tuple: Generated SQL query, selected columns and stat columns
        """
        return self._generate_query(query_type, table_name, selected_columns, where_clause, aql_output, limit, offset, sort_by, sort_order, entity, team_code, sport_code)
```

**Purpose**: This is a generic internal method designed to centralize the dispatch to the core SQL generation logic. All specific `_parse_*_query` methods delegate to this method.

**Parameters**: This method accepts all parameters required for constructing a SQL query, as described in `convert_aql_to_sql`.

**Returns**:
*   `tuple`: The result of `_generate_query`, which includes the generated SQL query, selected columns, and stat columns.

**Implementation Logic**:
*   It directly calls `self._generate_query()` with all the provided parameters. This means that `_parse_query` acts as a simple pass-through to the actual SQL generation function.

#### 3.8. `_parse_basic_query(*args, **kwargs)`, `_parse_min_query(*args, **kwargs)`, `_parse_max_query(*args, **kwargs)`, `_parse_ltw_query(*args, **kwargs)`, `_parse_streak_query(*args, **kwargs)`

```python
    def _parse_basic_query(self, *args, **kwargs):
        """
        Parses a basic query by delegating to the generic parser.
        ...
        """
        return self._parse_query(QueryTypes.BASIC.value, *args, **kwargs)

    def _parse_min_query(self, *args, **kwargs):
        """
        Parses a minimum value query by delegating to the generic parser.
        ...
        """
        return self._parse_query(QueryTypes.MIN.value, *args, **kwargs)

    def _parse_max_query(self, *args, **kwargs):
        """
        Parses a maximum value query by delegating to the generic parser.
        ...
        """
        return self._parse_query(QueryTypes.MAX.value, *args, **kwargs)

    def _parse_ltw_query(self, *args, **kwargs):
        """
        Parses a last time when query by delegating to the generic parser.
        ...
        """
        return self._parse_query(QueryTypes.LTW.value, *args, **kwargs)

    def _parse_streak_query(self, *args, **kwargs):
        """
        Parses a streak query by delegating to the generic parser.
        ...
        """
        return self._parse_query(QueryTypes.STREAK.value, *args, **kwargs)
```

**Purpose**: These methods serve as specific entry points for different `query_type`s. They are called by `convert_aql_to_sql` based on the `query_type`.

**Parameters**:
*   `*args`: Variable-length argument list, passed directly to `_parse_query`.
*   `**kwargs`: Arbitrary keyword arguments, passed directly to `_parse_query`.

**Returns**:
*   `tuple`: The results returned by `_parse_query`.

**Implementation Logic**:
*   Each method simply calls `self._parse_query()` and explicitly passes its specific `QueryTypes` constant (e.g., `QueryTypes.BASIC.value`) as the `query_type` argument, along with all `*args` and `**kwargs` received. This design centralizes the actual parsing logic in `_parse_query` (and subsequently `_generate_query`) and provides a clean dispatch mechanism.

#### 3.9. `_get_sort_mapping(self, query_type, stat_column)`

```python
    def _get_sort_mapping(self, query_type, stat_column):
        """
        Gets the default sort column and order for different query types.

        Args:
            query_type (str): Type of query being executed
            stat_column (str): Statistical column being queried

        Returns:
            tuple: Contains sort column and sort order (ASC/DESC)
        """
        if query_type == QueryTypes.LTW.value:
            stat_period = getattr(self, "stat_period_for_ltw", "").lower()
            if stat_period == "season":
                return (SortingFields.SEASON.value, SortOrders.DESC.value)
            else:
                return (SortingFields.GAMEDATE.value, SortOrders.Orders.DESC.value)

        # Default Sorting orders for different query types
        sort_mapping = {
            QueryTypes.BASIC.value: (stat_column, SortOrders.DESC.value),
            QueryTypes.MIN.value: (stat_column, SortOrders.ASC.value),
            QueryTypes.MAX.value: (stat_column, SortOrders.DESC.value),
            QueryTypes.STREAK.value: (SortingFields.STREAKLENGTH.value, SortOrders.DESC.value)
        }
        
        return sort_mapping.get(query_type, (stat_column, SortOrders.ASC.value))
```

**Purpose**: This private method determines the default sorting column and order based on the `query_type` and the `stat_column` involved in the query. This ensures a consistent and logical default sort behavior for different query types.

**Parameters**:
*   `query_type` (`str`): The type of query (e.g., 'basic', 'min', 'max', 'ltw', 'streak').
*   `stat_column` (`str`): The name of the statistical column that is central to the query (e.g., 's_points').

**Returns**:
*   `tuple`: A tuple `(sort_column, sort_order)`, where `sort_column` is the column to sort by and `sort_order` is 'ASC' or 'DESC'.

**Implementation Logic**:
1.  **LTW (Last Time When) Special Handling**:
    *   If `query_type` is `QueryTypes.LTW.value`:
        *   It tries to retrieve `stat_period_for_ltw` from the instance (which is set in `_generate_query`). If not found, it defaults to an empty string.
        *   If `stat_period` is "season", it sorts by `SEASON` (descending).
        *   Otherwise (assumed to be 'game' or similar), it sorts by `GAMEDATE` (descending). This prioritizes the most recent occurrence.
2.  **General Query Type Mapping**:
    *   `sort_mapping` dictionary defines default sort behaviors for `BASIC`, `MIN`, `MAX`, and `STREAK` query types.
        *   For `BASIC` and `MAX`, it sorts the `stat_column` in `DESC` order (e.g., highest score first).
        *   For `MIN`, it sorts the `stat_column` in `ASC` order (e.g., lowest score first).
        *   For `STREAK`, it specifically sorts by `STREAKLENGTH` in `DESC` order.
3.  **Default Fallback**: `sort_mapping.get(query_type, (stat_column, SortOrders.ASC.value))` is used to retrieve the mapping. If `query_type` is not found in the `sort_mapping`, it defaults to sorting by the `stat_column` in `ASC` order.

#### 3.10. `_generate_query(self, query_type, table_name, selected_columns, where_clause, aql_output, limit, offset, sort_by, sort_order, entity, team_code, sport_code)`

```python
    def _generate_query(self, query_type, table_name, selected_columns, where_clause, aql_output, limit, offset, sort_by, sort_order, entity, team_code, sport_code):
        """
        Generate SQL queries from templates while avoiding duplicate WHERE clauses.

        Args:
            query_type (str): Type of query (basic, min, max, ltw, streak)
            table_name (str): Name of the database table
            selected_columns (list): Columns to select
            where_clause (str): WHERE clause conditions
            aql_output (dict): Output from AQL parsing
            limit (int): Number of results to return
            offset (int): Number of results to skip
            sort_by (str): Column to sort by
            sort_order (str): Sort order (ASC/DESC)
            entity (str): Entity type (player/team)
            team_code (str): Team identifier code
            sport_code (str): Sport identifier code

        Returns:
            tuple: Contains:
                - str: Generated SQL query
                - list: Selected columns
                - list: Stat columns used in query

        Raises:
            ValueError: If min/max query is missing required stat field
        """

        print("Sorting Info", sort_by, sort_order)
        template_key = (f"{query_type}_{sport_code.lower()}_{entity.lower()}" 
                       if query_type == QueryTypes.STREAK.value 
                       else query_type)
        
        sql_template = self.sql_loader.load_template(template_key)

        if sport_code in ("MBB", "WBB", "WBB"): # Typo: WBB listed twice
            formatted_team_code = f"'{team_code}'"
        else:
            formatted_team_code = team_code

        conditions = aql_output.get(SortingFields.CONDITIONS.value, "").strip()
        streak_length = aql_output.get("streak_len", "")

        if streak_length == None:
            streak_length = "1"

        print(f"Streak Length is {streak_length}")

        pat = re.compile("\s+AND|and|OR|or\s+")
        col_extr_pat = re.compile("s_[a-z0-9_]+")

        stat_columns = col_extr_pat.findall(conditions) if conditions else set()
        print(f"New stat cols: {stat_columns}")
        if query_type in [QueryTypes.MIN.value, QueryTypes.MAX.value]:
            if not stat_columns:
                raise ValueError(f"{query_type.upper()} query must have a valid stat field.")
            stat_column = list(stat_columns)[0]
        else:
            stat_column = next(iter(stat_columns), None)

        self.stat_period_for_ltw = aql_output.get("stat_period", "").lower()
        default_sort_by, default_sort_order = self._get_sort_mapping(query_type, stat_column)



        final_sort_by = sort_by or default_sort_by
        final_sort_order = default_sort_order
        if sort_order:
            final_sort_order = sort_order.upper()

        print("Sorting Info", final_sort_by, final_sort_order)

        # final_sort_order = (sort_order.upper()
        #                    if sort_order and sort_order.upper() in [SortOrders.ASC, SortOrders.DESC]
        #                    else default_sort_order)

        if query_type != QueryTypes.STREAK.value:
            stat_column_conditions = [f"{stat_column} != 0" for stat_column in stat_columns]
            where_clause = where_clause + " AND " +" AND ".join(stat_column_conditions)

        if query_type == QueryTypes.STREAK.value:
            where_clause = where_clause + " AND " + "period_type='REGULAR'"



        formatted_where_clause = (f"AND {where_clause}" 
                                if query_type in [QueryTypes.MIN.value, QueryTypes.MAX.value] and where_clause 
                                else f"WHERE {where_clause}" if where_clause else "")

        print(f"Formatted Where Clause: {formatted_where_clause, formatted_team_code}")

        sql_query = sql_template.format(
            table_name=table_name,
            selected_columns=", ".join(selected_columns),
            stat_column=stat_column or "",
            where_clause=formatted_where_clause.strip(),
            conditions=conditions,
            streak_length=streak_length,
            final_sort_by=final_sort_by,
            final_sort_order=final_sort_order,
            limit=limit,
            offset=offset,
            team_code=formatted_team_code
        )

        # count_query = f"SELECT COUNT(*) FROM {table_name}"
        # if where_clause:
        #     count_query += f" WHERE {where_clause}"
        # count_query += ";"

        return sql_query, selected_columns, list(stat_columns)
```

**Purpose**: This is the core method responsible for loading the appropriate SQL template and populating it with all the dynamically generated components (table name, selected columns, `WHERE` clause, sorting, pagination, etc.) to produce the final SQL query.

**Parameters**: This method accepts all parameters defined for `_parse_query`, detailing the query type, table, columns, conditions, limits, offsets, and sorting.

**Returns**:
*   `tuple`: Contains the generated SQL query string, the list of `selected_columns`, and a list of `stat_columns` used in the query.

**Raises**:
*   `ValueError`: If a `MIN` or `MAX` query is initiated without a valid statistical field in its conditions.

**Implementation Logic**:
1.  **Template Key Determination**:
    *   `template_key` is constructed:
        *   For `STREAK` queries, it's specific: `"{query_type}_{sport_code.lower()}_{entity.lower()}"` (e.g., `streak_mbb_player`).
        *   For other query types, it's simply the `query_type` (e.g., `basic`, `min`).
2.  **Load SQL Template**: `self.sql_loader.load_template(template_key)` retrieves the SQL template string corresponding to the `template_key`.
3.  **Format `team_code`**:
    *   If `sport_code` is 'MBB' or 'WBB' (note: 'WBB' is duplicated), `team_code` is wrapped in single quotes (`'team_code'`). This might be due to how `team_code` is stored in the database for these sports.
    *   Otherwise, `team_code` is used as is.
4.  **Extract Conditions and Streak Length**:
    *   `conditions` are retrieved from `aql_output`.
    *   `streak_length` is retrieved, defaulting to "1" if `None`.
5.  **Extract Stat Columns**:
    *   Regular expressions (`pat` for splitting, `col_extr_pat` for extracting `s_` prefixed columns) are used to find statistical column names within the `conditions` string. The results are stored in `stat_columns` (as a set for uniqueness).
6.  **Validate Stat Columns for MIN/MAX**:
    *   If `query_type` is `MIN` or `MAX` and no `stat_columns` were found, a `ValueError` is raised, as these queries inherently require a statistical field.
    *   `stat_column` is set to the first stat column found, or `None` if no stat columns are present.
7.  **Store `stat_period` for LTW**: `self.stat_period_for_ltw` is set to the `stat_period` from `aql_output`. This is used by `_get_sort_mapping` for LTW queries.
8.  **Determine Sorting**:
    *   `default_sort_by`, `default_sort_order` are obtained from `self._get_sort_mapping()`.
    *   `final_sort_by` is set to `sort_by` if provided, otherwise `default_sort_by`.
    *   `final_sort_order` defaults to `default_sort_order`, but is overridden by `sort_order` (converted to uppercase) if `sort_order` is provided.
9.  **Adjust `WHERE` Clause**:
    *   **For non-STREAK queries**: If `stat_columns` exist, an additional condition `"{stat_column} != 0"` is added for each stat column to the `where_clause`. This typically ensures that zero values for stats are excluded from consideration unless explicitly requested.
    *   **For STREAK queries**: `period_type='REGULAR'` is appended to the `where_clause`.
10. **Format `where_clause` for Template**:
    *   `formatted_where_clause` is created:
        *   If `query_type` is `MIN` or `MAX` AND `where_clause` exists, it's prefixed with `AND` (assuming the template already starts with `WHERE`).
        *   Otherwise, if `where_clause` exists, it's prefixed with `WHERE`.
        *   If `where_clause` is empty, `formatted_where_clause` remains empty.
11. **Populate SQL Template**: The `sql_template.format()` method is used to substitute placeholders in the template with all the generated and provided values.
    *   Placeholders include: `table_name`, `selected_columns`, `stat_column`, `where_clause`, `conditions`, `streak_length`, `final_sort_by`, `final_sort_order`, `limit`, `offset`, and `team_code`.
12. **Return Value**: The fully constructed `sql_query` string, `selected_columns` list, and `stat_columns` list are returned.
13. **Commented-out `count_query`**: A block of code for generating a `COUNT(*)` query is commented out, indicating this functionality might be handled elsewhere or is not currently in use.

### 4. Main Execution Flow (`__main__` block)

The provided Python script snippet **does not contain a `if __name__ == "__main__":` block**. This means it is designed to be imported as a module into another application, rather than being executed directly as a standalone script. The `SQLBuilder` class and its methods would be instantiated and called from other parts of the application.
