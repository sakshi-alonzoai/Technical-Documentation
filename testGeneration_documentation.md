# Technical Documentation: `testGeneration.py`

## 1. Overview

This script, `testGeneration.py`, is a command-line utility designed to automatically generate natural language test queries for a sports statistics application. It leverages a combination of predefined templates, data from a MongoDB database, and local JSON files to construct a variety of queries. These generated queries can be used to test the natural language understanding (NLU) and query processing capabilities of a sports-focused AI or search engine.

The script supports generating queries for different sports (`MFB`, `MBB`, `WBB`), entities (`player`, `team`), and query types (`simple`, `qualifiers`, `min_max`, `ltw`, `streak`, `periods`). It can generate a specified number of random queries or enumerate all possible queries for a given configuration.

## 2. Dependencies

The script relies on the following Python libraries:

*   **`argparse`**: For parsing command-line arguments.
*   **`json`**: For working with JSON data (templates and stat mappings).
*   **`os`**: For file and directory operations.
*   **`random`**: For random selection of templates and data.
*   **`python-decouple`**: For managing configuration variables (like database credentials) from environment variables or `.env` files.
*   **`pymongo`**: The official Python driver for MongoDB, used to connect to and query the database.
*   **`pandas`**: Used for data manipulation, although its usage is commented out in the current version.
*   **`copy`**: Specifically `deepcopy` is used to create independent copies of stat objects when generating aliases.

You can install these dependencies using `pip`:

```bash
pip install -r requirements.txt
```

## 3. Configuration

### 3.1. Environment Variables

The script requires the following environment variables to be set for database connection:

*   `DB_URI`: The MongoDB connection string (e.g., `mongodb://localhost:27017/`).
*   `DB_NAME`: The name of the MongoDB database to use.

These are loaded using the `decouple` library, which typically reads from a `.env` file in the project root.

### 3.2. File Structure

The script expects a specific directory structure to locate templates and stat mappings:

*   **`templates/`**: Contains the JSON files with the natural language query templates. It is organized by entity (`player`, `team`) and then by query type (`simple.json`, `qualifiers.json`, etc.).
*   **`stat_mapping/`**: Contains JSON files that map human-readable stat names to their internal database representations. This is organized by sport code (`MFB`, `MBB`, `WBB`) and then by entity (`Player.json`, `Team.json`).
*   **`query_outputs/`**: This is the default output directory where generated queries are saved as JSON files. The script will create this directory if it doesn't exist.

## 4. Core Components

### 4.1. `load_templates(base_path="templates")`

This function is responsible for loading the natural language query templates from the `templates` directory.

*   It iterates through the `player` and `team` subdirectories within the `base_path`.
*   For each subdirectory, it reads all `.json` files.
*   The name of each JSON file (without the extension) becomes a key in the `templates` dictionary (e.g., `simple`, `qualifiers`).
*   The content of each JSON file is parsed into a list of template objects.

The resulting `templates` dictionary has a nested structure: `templates[entity][query_type]`.

### 4.2. `generate_query(...)`

This is the core function of the script, where the query generation logic resides.

**Parameters:**

*   `query_type`: The type of query to generate (e.g., "simple").
*   `num_samples`: The number of queries to generate.
*   `sport_code`: The sport code (e.g., "MFB").
*   `entity`: The entity type ("player" or "team").
*   `g_team_id`: The team ID used to fetch a list of players.
*   `stat_period`: Filters templates by the statistical period (e.g., "game", "season").
*   `generate_all`: If `True`, it generates queries for all possible stats instead of a random selection.
*   `generate_aliases`: If `True`, it generates additional queries for all known aliases of a stat.
*   `template_id`: If provided, it generates a query for a specific template ID.

**Logic:**

1.  **Data Fetching:** It begins by fetching all necessary data for query construction:
    *   Stat mappings (`get_stat_mapping`)
    *   Team names (`get_teams`)
    *   Conference names (`get_conferences`)
    *   Player names for a given team (`get_players`)
    *   And other dimension data like player positions, seasons, etc.

2.  **Template Selection:** It selects the appropriate list of templates from the loaded `templates` dictionary based on the `entity` and `query_type`.

3.  **Iteration and Generation:** It then enters a loop to generate `num_samples` queries. In each iteration:
    *   It selects a template (either randomly or by the provided `template_id`).
    *   It selects one or more random stats from the `stat_mappings`.
    *   It generates random values for the stats.
    *   It populates the `nlp_query` from the template with the selected stat names, values, and any other required qualifiers (like team names, player names, etc.).
    *   It constructs the `expected_output` (the AQL - Athlyte Query Language representation) by filling in the template with the corresponding internal stat labels and values.

4.  **Alias Generation:** If `generate_aliases` is true, it creates copies of the query for each alias of the selected stat, providing broader test coverage.

5.  **Output:** The function returns a list of dictionaries, where each dictionary represents a structured test query containing the natural language query and the expected AQL output.

### 4.3. Data Fetching Functions

These are a group of helper functions that retrieve data from either the MongoDB database or local files.

*   **`get_stat_mapping(sport_code, entity)`**: Reads a JSON file from the `stat_mapping/` directory corresponding to the given sport and entity.
*   **`get_teams()`**: Fetches all team names from the `teams` collection in MongoDB.
*   **`get_players(g_team_id)`**: Fetches player names from the `ActiveRoster` collection for a specific team ID.
*   **`get_conferences()`**: Fetches all conference names from the `conferences` collection.
*   **`get_divisions()`**: Returns a hardcoded list of division names.
*   **`get_player_classes(...)`**: Returns a list or dictionary of player class names (e.g., "freshmen", "sophomores").
*   **`get_player_positions(sport_code)`**: Returns a dictionary mapping position names to their abbreviations for a given sport.

### 4.4. `save_queries_to_file(...)`

This function takes the list of generated queries and saves them to a JSON file in the `query_outputs` directory. The filename is timestamped to prevent overwriting previous runs.

## 5. Execution

The script is intended to be run from the command line. The `main()` function parses command-line arguments and triggers the query generation process.

**Example Usage:**

*   Generate 5 simple queries for a player in Men's Football:

    ```bash
    python testGeneration.py --num_samples 5 --query_type simple --entity player --sport_code MFB --save-queries
    ```

*   Generate all possible qualifier queries for a team in Men's Basketball:

    ```bash
    python testGeneration.py --query_type qualifiers --entity team --sport_code MBB --generate_all --save-queries
    ```

*   Generate a query for a specific template (ID 123):
    ```bash
    python testGeneration.py --template_id 123 --save-queries
    ```

*   Display the custom help message:

    ```bash
    python testGeneration.py --help-custom
    ```

## 6. Code Improvements and Suggestions

For a beginner looking to improve this script, here are some suggestions:

### 6.1. Reduce Database Calls

The `generate_query` function calls all data fetching functions (`get_teams`, `get_players`, etc.) on every invocation. For generating a large number of samples, these data could be fetched once and passed as arguments or stored in a global cache.

**Before:**
```python
def generate_query(...):
    teams = get_teams()
    players = get_players(g_team_id)
    # ... loop ...
```
**After:**
```python
def generate_query(..., teams, players):
    # ... loop ...

def main():
    teams = get_teams()
    players = get_players(args.g_team_id)
    results = generate_query(..., teams=teams, players=players)
```

### 6.2. Decouple Data Loading from Generation

The `generate_query` function is doing too much. It's fetching data, selecting templates, and generating queries. This makes it hard to test and maintain. You could refactor it by creating a dedicated class or module that handles data loading and caching, and another for the generation logic itself.

**Example Structure:**
```python
class DataProvider:
    def __init__(self, sport_code, g_team_id):
        self.sport_code = sport_code
        self.g_team_id = g_team_id
        self._cache = {}

    def get_teams(self):
        if 'teams' not in self._cache:
            # Fetch from DB
        return self._cache['teams']
    # ... other data methods

class QueryGenerator:
    def __init__(self, data_provider, templates):
        self.data_provider = data_provider
        self.templates = templates

    def generate(self, ...):
        # Generation logic here
```

### 6.3. Improve Error Handling

The script has minimal error handling. For example, if a database connection fails, the script will crash with an unhandled exception. If a template file is missing, it will also fail.

*   Wrap database calls in `try...except` blocks to handle connection errors gracefully.
*   Check if files exist before trying to open them in `load_templates` and `get_stat_mapping`.
*   The `ValueError` for a missing `template_id` is good, but you could add more checks for other invalid inputs.

### 6.4. Enhance Modularity with Functions

The `generate_query` function is very long. You can break down parts of its logic into smaller, more manageable functions. For instance, the logic for handling qualifiers or filling in the template could be extracted into their own helper functions.

**Example:**
```python
def _populate_qualifiers(template, teams, players, ...):
    qualifiers = {}
    # ... logic for populating qualifiers ...
    return qualifiers

def generate_query(...):
    # ...
    qualifiers = _populate_qualifiers(template, teams, players, ...)
    # ...
```

### 6.5. Add Unit Tests

To ensure the script works as expected and to prevent regressions when making changes, you should add unit tests. You can use Python's built-in `unittest` framework or a library like `pytest`.

*   Write tests for `load_templates` to ensure it correctly loads and parses JSON files.
*   Write tests for the data fetching functions (you might need to mock the database).
*   Write tests for `generate_query` to verify that it produces the correct output structure for different inputs.

By implementing these improvements, the script will become more robust, maintainable, and easier to understand and extend.
