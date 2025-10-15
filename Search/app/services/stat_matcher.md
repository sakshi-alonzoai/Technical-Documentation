# Documentation for `stat_matcher.py`

This document provides comprehensive technical documentation for the given Python script. It covers the script's overall purpose, detailed explanations of classes and functions, their interactions, and the command-line interface provided.

---

## Technical Documentation: Statistical Indexing and Matching

### 1. High-Level Overview

This Python script implements a robust system for creating, managing, and querying statistical indexes using SentenceTransformer models. Its primary purpose is to enable semantic similarity search for statistical terms and their descriptions. It allows users to:

1.  **Create Statistical Indexes**: Generate vector embeddings for a corpus of statistics based on their names, aliases, categories, subcategories, and descriptions.
2.  **Save/Load Indexes**: Persist and retrieve these indexes (including embeddings and metadata) to and from disk.
3.  **Match Queries**: Perform semantic searches against the indexed statistics using natural language queries, identifying the most relevant statistics.
4.  **Test Matching Performance**: Evaluate the accuracy of the matching process against a predefined set of queries and expected outcomes.
5.  **Command-Line Interface (CLI)**: Provide a user-friendly interface for creating, testing, and querying indexes directly from the terminal.

The script is designed to handle statistical data for various sports and entities (e.g., player stats for NBA, team stats for NFL) by organizing indexes in a hierarchical file structure.

### 2. Imports

The script begins by importing necessary libraries:

*   `numpy as np`: Essential for numerical operations, particularly for handling vector embeddings and similarity scores.
*   `sentence_transformers.SentenceTransformer`: The core library for loading pre-trained sentence embedding models and performing encoding and similarity calculations.
*   `pickle`: Used for serializing and deserializing Python objects (the statistical index) to and from files.
*   `enum.Enum`: Provides the base class for creating enumeration types, used here for defining available SentenceTransformer models.
*   `json`: For parsing and generating JSON data, primarily used for reading statistical mappings and test queries, and for outputting results.
*   `typer`: A library for building robust command-line applications, used to expose the script's functionalities as CLI commands.
*   `os`: Provides a way of using operating system-dependent functionality, crucial for path manipulation (joining, checking existence, listing directories).
*   `re`: The regular expression module, used for pattern matching and splitting strings (e.g., parsing query conditions).
*   `time` (within `if __name__ == '__main__':`): For measuring the execution time of various operations.

### 3. Global Constants and Configuration

These variables define important paths and mappings for the application.

*   `PROJECT_ROOT = os.path.dirname(os.path.dirname(os.path.dirname(__file__)))`
    *   **Purpose**: Dynamically determines the absolute path to the project's root directory.
    *   **Logic**:
        *   `__file__` is the path to the current script (`index_manager.py`).
        *   `os.path.dirname(__file__)` gets the directory containing the script.
        *   Applying `os.path.dirname()` three times sequentially navigates up three levels in the directory hierarchy, assuming a typical project structure where this script might be deep within a `src/utils` or `app/ml_utils` type folder.
*   `INDEX_BASEPATH = os.path.join(PROJECT_ROOT, 'app', 'ml_assets')`
    *   **Purpose**: Defines the base directory where all machine learning assets, specifically the statistical indexes, are stored.
    *   **Logic**: Constructs the full path by joining `PROJECT_ROOT` with `app` and `ml_assets`.
*   `INDEX_MAP = { ... }`
    *   **Purpose**: A nested dictionary that maps `sport_code` (e.g., 'NBA') to another dictionary, which then maps an `entity` type (e.g., 'player', 'team') to the absolute path of its corresponding index file. This provides a structured way to locate specific indexes.
    *   **Logic**:
        1.  It iterates through each `sport_code` directory found directly under `INDEX_BASEPATH`.
        2.  For each `sport_code`, it creates an inner dictionary.
        3.  Inside the inner dictionary, it iterates through each file (`fname`) within the `sport_code`'s directory.
        4.  It uses `fname.split(".")[0]` to get the filename without its extension (which typically represents the `entity`).
        5.  The value stored is the full path to that index file, constructed using `os.path.join(INDEX_BASEPATH, sport_code, fname)`.

### 4. `STModelChoice` Enum

```python
class STModelChoice(Enum):
    """
    Enumeration for different SentenceTransformer model choices.
    Currently supports:
    - MINILM_L6_V2: A lightweight and efficient model for sentence embeddings
    """
    MINILM_L6_V2 = "all-MiniLM-L6-v2"
```

*   **Purpose**: This `Enum` (enumeration) defines a fixed set of pre-trained SentenceTransformer models that can be used for encoding text. It improves code readability and prevents hardcoding model names.
*   **Members**:
    *   `MINILM_L6_V2`: Represents the "all-MiniLM-L6-v2" model, known for its balance of performance and efficiency in generating sentence embeddings.
*   **Usage**: When initializing `StatIndex`, one can pass `STModelChoice.MINILM_L6_V2` as the `encoding_model` argument. The `.value` attribute (`STModelChoice.MINILM_L6_V2.value`) provides the actual string name required by the `SentenceTransformer` library.

### 5. `StatIndex` Class

```python
class StatIndex:
    """
    A class for indexing statistical mappings using SentenceTransformer embeddings.
    Provides functionality to create, save, and load statistical indexes for efficient similarity search.

    Attributes:
        stat_mapping_file (str): Path to the JSON file containing statistical mappings.
        encoding_model (STModelChoice): The SentenceTransformer model used for encoding.
        stat2ind (dict): Mapping of statistic names to their corresponding indices.
        ind2stat (list): List of statistic names indexed by their position.
        corpus_embeddings (list): Precomputed embeddings for statistics.
        model (SentenceTransformer): The SentenceTransformer model instance.
    """
    # ... methods ...
```

This class is responsible for managing the statistical index, including loading raw data, generating embeddings, and saving/loading the entire index structure.

#### 5.1. `__init__(self, stat_mapping_file, encoding_model: STModelChoice = STModelChoice.MINILM_L6_V2.value)`

*   **Purpose**: Initializes a `StatIndex` object. It sets up the path to the raw statistical mapping file and loads the specified SentenceTransformer model.
*   **Parameters**:
    *   `stat_mapping_file` (`str`): The file path to a JSON file containing the raw statistical definitions. If `None` (e.g., when loading an existing index), it's skipped.
    *   `encoding_model` (`STModelChoice` or `str`, optional): The SentenceTransformer model to use. Defaults to `STModelChoice.MINILM_L6_V2.value`.
*   **Implementation Logic**:
    1.  `if stat_mapping_file:`: Checks if a `stat_mapping_file` is provided. If so, it stores the path and loads the JSON content from the file into `self.stat_mapping`.
    2.  `self.encoding_model = encoding_model`: Stores the specified encoding model.
    3.  `self.stat2ind = {}`, `self.ind2stat = []`, `self.corpus_embeddings = []`: Initializes empty data structures that will hold the index components.
    4.  `if not isinstance(self.encoding_model, str): self.encoding_model = self.encoding_model.value`: Ensures that `self.encoding_model` is a string (the model name) suitable for `SentenceTransformer`, handling cases where an `STModelChoice` Enum member was passed.
    5.  `self.model = SentenceTransformer(self.encoding_model)`: Instantiates the `SentenceTransformer` model using the specified model name. This might download the model if it's not cached locally.

#### 5.2. `create(self)`

*   **Purpose**: Builds the statistical index by transforming the raw `stat_mapping` data into a corpus of text and then generating embeddings for this corpus.
*   **Implementation Logic**:
    1.  `corpus = [...]`: A list comprehension creates the `corpus`. For each `stat` dictionary in `self.stat_mapping`, it constructs a descriptive string by concatenating the statistic's `name`, `aliases`, `category`, `sub_category`, and `description`. `get('key', default_value)` is used to handle potentially missing keys gracefully.
    2.  `self.ind2stat = [stat['stat'] for stat in self.stat_mapping]`: Creates a list `ind2stat` where each element is the unique 'stat' identifier from the raw mapping, ordered by its original position.
    3.  `self.stat2ind = {stat: i for i, stat in enumerate(self.ind2stat)}`: Creates a dictionary `stat2ind` that maps each 'stat' identifier to its integer index within `self.ind2stat` (and thus within the `corpus`).
    4.  `self.corpus_embeddings = self.model.encode(corpus)`: Uses the loaded `SentenceTransformer` model to encode the entire `corpus` into a matrix of dense vector embeddings. Each row in `corpus_embeddings` corresponds to the embedding of a statistic in the `corpus`.

#### 5.3. `save(self, path: str)`

*   **Purpose**: Serializes and saves the current state of the `StatIndex` object to a file.
*   **Parameters**:
    *   `path` (`str`): The file path where the index data will be saved.
*   **Implementation Logic**:
    1.  `with open(path, 'wb') as f:`: Opens the specified file in binary write mode (`'wb'`).
    2.  `all_objs = { ... }`: Creates a dictionary containing all essential attributes of the `StatIndex` instance (`stat_mapping_file`, `stat_mapping`, `stat2ind`, `ind2stat`, `corpus_embeddings`, `encoding_model`). `str(self.encoding_model)` ensures the model name is stored as a string.
    3.  `pickle.dump(all_objs, f)`: Uses `pickle` to serialize the `all_objs` dictionary and write it to the file.

#### 5.4. `load(self, path: str)`

*   **Purpose**: Deserializes and loads a previously saved `StatIndex` object from a file, reconstructing its state.
*   **Parameters**:
    *   `path` (`str`): The file path from which the index data will be loaded.
*   **Implementation Logic**:
    1.  `with open(path, 'rb') as f:`: Opens the specified file in binary read mode (`'rb'`).
    2.  `all_objs = pickle.load(f)`: Uses `pickle` to deserialize the data from the file back into a dictionary.
    3.  The script then assigns the loaded values back to the instance attributes: `self.stat_mapping_file`, `self.stat_mapping`, `self.stat2ind`, `self.ind2stat`, `self.corpus_embeddings`.
    4.  `encoding_model = all_objs['encoding_model']`: Retrieves the model name that was used when the index was saved.
    5.  `if encoding_model != self.encoding_model:`: Checks if the loaded `encoding_model` differs from the one currently set in the `StatIndex` object (which might be the default or `None` if `stat_mapping_file` was `None` during `__init__`).
    6.  If they differ, it updates `self.encoding_model` and re-initializes `self.model` with the correct `SentenceTransformer` instance to ensure compatibility with the loaded embeddings. This handles potential mismatches if an index was saved with a different model than the one currently assumed. `if not isinstance(self.encoding_model, str): self.encoding_model = self.encoding_model.value` ensures it's always a string before instantiating `SentenceTransformer`.

### 6. `StatMatcher` Class

```python
class StatMatcher:
    """
    A class to match query statistics against the indexed statistical embeddings.
    Provides functionality for semantic similarity search across statistical descriptions.

    Attributes:
        index (StatIndex): An instance of StatIndex for performing queries.
        condition_splitter (re.Pattern): Regular expression to split query conditions.
    """
    # ... methods ...
```

This class provides the functionality to take a natural language query and find the most semantically similar statistics from a loaded `StatIndex`.

#### 6.1. `__init__(self, index: StatIndex)`

*   **Purpose**: Initializes a `StatMatcher` object with a `StatIndex` instance.
*   **Parameters**:
    *   `index` (`StatIndex`): An already loaded `StatIndex` object containing the statistical embeddings.
*   **Implementation Logic**:
    1.  `self.index = index`: Stores the provided `StatIndex` instance.
    2.  `self.condition_splitter = re.compile("AND|OR|WITHOUT", re.IGNORECASE)`: Compiles a regular expression pattern to split queries. This pattern looks for "AND", "OR", or "WITHOUT" (case-insensitive) to break down a complex query into individual components.

#### 6.2. `from_sport_and_entity(sport_code: str, entity: str)` (Static Method)

*   **Purpose**: A static factory method that creates and returns a `StatMatcher` instance for a specific sport and entity combination by loading the corresponding pre-built index.
*   **Parameters**:
    *   `sport_code` (`str`): The code for the sport (e.g., 'NBA', 'NFL').
    *   `entity` (`str`): The type of entity (e.g., 'player', 'team').
*   **Returns**:
    *   `StatMatcher`: A configured `StatMatcher` instance, or `None` if the index file is not found.
*   **Implementation Logic**:
    1.  `index_file = INDEX_MAP[sport_code][entity]`: Uses the global `INDEX_MAP` to retrieve the path to the specific index file.
    2.  Debug print statement for `index_file`.
    3.  `if not index_file or not os.path.exists(index_file):`: Checks if the `index_file` path is valid and if the file actually exists on disk.
    4.  If the file is not found, an error message is printed, and `None` is returned.
    5.  `index = StatIndex(None)`: Creates a new `StatIndex` instance without an initial mapping file, as it will be loaded from a saved file.
    6.  `index.load(index_file)`: Loads the statistical index data from the identified `index_file`.
    7.  `return StatMatcher(index)`: Creates and returns a `StatMatcher` instance, initialized with the loaded `index`.

#### 6.3. `get_all_stat_matchers()` (Static Method)

*   **Purpose**: A static factory method that creates and returns a dictionary of `StatMatcher` instances for all available sport and entity combinations defined in `INDEX_MAP`.
*   **Returns**:
    *   `dict`: A nested dictionary where `stat_matchers[sport_code][entity]` holds the corresponding `StatMatcher` object.
*   **Implementation Logic**:
    1.  `stat_matchers = {}`: Initializes an empty dictionary to store the results.
    2.  `for sport_code, entities in INDEX_MAP.items():`: Iterates through each sport and its associated entities in the global `INDEX_MAP`.
    3.  `stat_matchers[sport_code] = {}`: Creates an inner dictionary for the current `sport_code`.
    4.  `for entity in entities:`: Iterates through each `entity` type for the current `sport_code`.
    5.  `stat_matchers[sport_code][entity] = StatMatcher.from_sport_and_entity(sport_code, entity)`: Calls `from_sport_and_entity` to create a `StatMatcher` for the specific sport/entity and stores it in the nested dictionary.
    6.  `return stat_matchers`: Returns the complete dictionary of `StatMatcher` instances.

#### 6.4. `match(self, query: str, top_k: int = 10)`

*   **Purpose**: Takes a natural language query, processes it, and finds the `top_k` most semantically similar statistics from the loaded index.
*   **Parameters**:
    *   `query` (`str`): The input query string (e.g., "player total points").
    *   `top_k` (`int`, optional): The number of top matching statistics to return. Defaults to 10.
*   **Returns**:
    *   `list`: A list of dictionaries, where each dictionary represents a matching statistic with its full metadata as stored in `stat_mapping`.
*   **Implementation Logic**:
    1.  `query_components = self.condition_splitter.split(query)`: Splits the input `query` string into multiple components using the `condition_splitter` regex (AND, OR, WITHOUT). This allows matching against individual phrases within a complex query.
    2.  `query_embeddings = self.index.model.encode(query_components)`: Encodes each `query_component` into a vector embedding using the `SentenceTransformer` model from the `StatIndex`.
    3.  `scores = self.index.model.similarity(self.index.corpus_embeddings, query_embeddings)`: Computes the cosine similarity between all `corpus_embeddings` (from the index) and all `query_embeddings`. The result `scores` is a matrix where `scores[i, j]` is the similarity between corpus embedding `i` and query component embedding `j`.
    4.  Debug print statements for embedding shapes.
    5.  `all_top_k_choices = []`: Initializes an empty list to collect indices of top-k matches.
    6.  `for i in range(scores.shape[1]):`: Iterates through each query component's similarity scores (columns in the `scores` matrix).
        *   `score_col = scores[:, i]`: Extracts the similarity scores for the current query component against all corpus statistics.
        *   `top_k_choices = np.argpartition(score_col, -top_k)[-top_k:]`: Uses `np.argpartition` to efficiently find the indices of the `top_k` highest scores in `score_col`. This is faster than a full sort for just the top k elements.
        *   `all_top_k_choices += top_k_choices`: Appends these `top_k` indices to the `all_top_k_choices` list.
    7.  `all_top_k_choices = list(set(all_top_k_choices))`: Converts the list to a `set` to remove duplicate indices (if multiple query components matched the same statistic highly), then converts it back to a list.
    8.  `all_top_k_choices = [self.index.stat_mapping[i] for i in all_top_k_choices]`: Retrieves the full statistic metadata dictionaries from `self.index.stat_mapping` using the unique top-k indices.
    9.  `return all_top_k_choices`: Returns the list of matching statistic metadata.

### 7. `test_stat_matching_with_sample` Function

```python
def test_stat_matching_with_sample(stat_mapping_file: str, test_file: str, output_file: str, top_k: int = 10):
    """
    Tests the statistical matching process with sample queries and evaluates accuracy.
    Processes a set of test queries, compares results against known correct matches,
    and generates a detailed report of matching performance.

    Args:
        stat_mapping_file (str): Path to the statistical mapping JSON file
        test_file (str): Path to the test queries JSON file
        output_file (str): Path to save the test results
        top_k (int, optional): The number of top matches to consider. Defaults to 10.
    """
    # ... implementation ...
```

*   **Purpose**: This standalone function orchestrates a test run for the `StatMatcher`. It loads a set of test queries, performs matching, and calculates performance metrics like accuracy and average matching time.
*   **Parameters**:
    *   `stat_mapping_file` (`str`): Path to the JSON file containing the statistics definitions (used to create the index).
    *   `test_file` (`str`): Path to a JSON file containing test queries, each with an expected `aql` (Abstract Query Language) condition that specifies the correct statistics.
    *   `output_file` (`str`): Path where the detailed test results will be saved in JSON format.
    *   `top_k` (`int`, optional): The number of top matches to consider for evaluation. Defaults to 10.
*   **Implementation Logic**:
    1.  `just_stats = re.compile("s[a-zA-Z]+[0-9]*[a-zA-Z]*")`: Compiles a regex to extract stat codes from the `aql` conditions in the test file (e.g., `sPoints` from `"sPoints > 10"`).
    2.  `stat_index = StatIndex(stat_mapping_file)`: Initializes a `StatIndex` with the provided mapping file.
    3.  **Index Creation Time Measurement**:
        *   `s = time.time()` and `e = time.time()`: Records start and end times for index creation.
        *   `stat_index.create()`: Builds the embeddings for the index.
        *   Prints the time taken for index creation.
    4.  `test_queries = json.loads(open(test_file).read())`: Loads the test queries from the specified JSON file.
    5.  `test_queries = [(x["query"], just_stats.findall(x["aql"]["conditions"])) for x in test_queries]`: Processes the loaded test queries. For each query, it extracts the natural language `query` string and uses the `just_stats` regex to find all expected 'stat' codes from the `aql` `conditions`.
    6.  `stat_mat = StatMatcher(stat_index)`: Initializes a `StatMatcher` with the newly created `stat_index`.
    7.  `avg_stat_matching_time = 0`, `in_top_5 = 0`, `total_stats = 0`, `new_collection = []`: Initializes variables for accumulating performance metrics and storing detailed results.
    8.  **Query Loop**: `for query in test_queries:` iterates through each test query.
        *   **Matching Time Measurement**: Records start and end times for the `match` operation.
        *   `top_k_matches = stat_mat.match(query[0], top_k=top_k)`: Calls the `StatMatcher`'s `match` method with the query string.
        *   Accumulates `avg_stat_matching_time`.
        *   `top_k_stats = [x["stat"] for x in top_k_matches]`: Extracts only the 'stat' codes from the returned `top_k_matches`.
        *   `matched_stats = 0`: Counter for correctly matched stats in the current query.
        *   `for stat in query[1]:`: Iterates through each *expected* `stat` code for the current query.
            *   `if stat in top_k_stats:`: Checks if the expected stat is present in the `top_k` matches returned by the matcher.
            *   Increments `matched_stats` and `total_stats` accordingly.
        *   `in_top_5 += matched_stats`: Adds the count of matched stats for this query to the overall `in_top_5` (which acts as a cumulative count of correctly identified individual stats).
        *   `new_collection.append(...)`: Appends a dictionary containing the original query, the `matched_stats` from the model, the `real_stats` (expected), and `unmatched_stats` to `new_collection` for detailed reporting.
    9.  **Results Reporting**:
        *   Prints overall accuracy (`in_top_5 / total_stats`) and average matching time per query.
        *   `with open(output_file, "w") as f: f.write(json.dumps(new_collection, indent=4))`: Saves the `new_collection` (detailed results) to the specified `output_file` in a human-readable JSON format.

### 8. Main Execution Block (`if __name__ == '__main__':`)

This block sets up and runs the command-line interface using `typer`.

*   `import time`, `import re`: These imports are placed here because they are specifically used by the `test_stat_matching_with_sample` function and CLI commands, not strictly necessary for the core `StatIndex` and `StatMatcher` classes themselves.
*   `app = typer.Typer()`: Initializes a Typer application instance.

#### 8.1. `@app.command("create")` decorator and `create_index` function

```python
    @app.command("create")
    def create_index(stat_mapping_file: str, save_path: str):
        """
        CLI command to create and save a new statistical index.

        Args:
            stat_mapping_file (str): Path to the statistical mapping JSON file
            save_path (str): Path where the created index should be saved
        """
        stat_index = StatIndex(stat_mapping_file)

        s = time.time()
        stat_index.create()
        e = time.time()

        print(f"Stat Mapping Index Creation took {e - s:.2f} seconds")
        stat_index.save(save_path)
```

*   **Purpose**: Defines a CLI command `create` to build a new index from scratch and save it.
*   **Parameters**:
    *   `stat_mapping_file` (CLI argument): Path to the JSON file containing raw statistics.
    *   `save_path` (CLI argument): Path where the serialized index will be stored.
*   **Implementation Logic**:
    1.  Creates a `StatIndex` instance.
    2.  Measures the time taken for `stat_index.create()`.
    3.  Prints the creation time.
    4.  Calls `stat_index.save(save_path)` to persist the created index.

#### 8.2. `@app.command("test")` decorator and `test` function

```python
    @app.command("test")
    def test(stat_mapping_file: str, test_file: str, output_file: str, top_k: int = 10):
        """
        CLI command to run statistical matching tests.

        Args:
            stat_mapping_file (str): Path to the statistical mapping JSON file
            test_file (str): Path to the test queries JSON file
            output_file (str): Path to save the test results
            top_k (int, optional): Number of top matches to consider. Defaults to 10.
        """
        test_stat_matching_with_sample(stat_mapping_file, test_file, output_file, top_k)
```

*   **Purpose**: Defines a CLI command `test` to run the statistical matching evaluation.
*   **Parameters**: Directly maps to the `test_stat_matching_with_sample` function's parameters.
*   **Implementation Logic**: Simply calls `test_stat_matching_with_sample` with the provided CLI arguments.

#### 8.3. `@app.command("query")` decorator and `query` function

```python
    @app.command("query")
    def query(index_file: str, query: str, top_k: int = 10):
        """
        CLI command to perform a single query against a statistical index.

        Args:
            index_file (str): Path to the saved statistical index
            query (str): The query string to match
            top_k (int, optional): Number of top matches to return. Defaults to 10.
        """
        stat_index = StatIndex(None)
        stat_index.load(index_file)
        stat_mat = StatMatcher(stat_index)
        s = time.time()
        results = stat_mat.match(query, top_k=top_k)
        e = time.time()
        print(json.dumps(results, indent=4))
        print(f"Query took {e - s:.2f} seconds")
```

*   **Purpose**: Defines a CLI command `query` to perform a single semantic search against a pre-existing index.
*   **Parameters**:
    *   `index_file` (CLI argument): Path to a previously saved index file.
    *   `query` (CLI argument): The natural language query string.
    *   `top_k` (CLI argument, optional): Number of top matches to return.
*   **Implementation Logic**:
    1.  Initializes a `StatIndex` (without a mapping file, as it will be loaded).
    2.  Loads the index from `index_file`.
    3.  Creates a `StatMatcher` instance.
    4.  Measures the time taken for `stat_mat.match()`.
    5.  Prints the query results (list of dictionaries) in a pretty-printed JSON format.
    6.  Prints the time taken for the query.

*   `app()`:
    *   **Purpose**: This line executes the Typer application, parsing command-line arguments and dispatching to the appropriate command function (`create_index`, `test`, or `query`) based on user input.

### 9. Important Implementation Details, Dependencies, and Configuration

*   **SentenceTransformer Model**: The script relies heavily on the `sentence_transformers` library. The default model, "all-MiniLM-L6-v2", is chosen for its efficiency and good performance for general-purpose sentence embeddings. This model will be downloaded automatically by the library if not present locally.
*   **Index Structure**: The `StatIndex` class stores `stat_mapping`, `stat2ind`, `ind2stat`, and `corpus_embeddings`. This comprehensive storage ensures that when an index is loaded, all necessary information (raw data, mappings, and the actual embeddings) is available for matching.
*   **Robustness in Loading**: The `StatIndex.load` method includes a check to ensure the `SentenceTransformer` model is correctly instantiated, even if the model name stored in the index differs from the default or current `StatIndex`'s `encoding_model`.
*   **Query Splitting**: The `StatMatcher` uses a regex (`condition_splitter`) to break down complex queries by "AND", "OR", and "WITHOUT" operators. This allows for multi-component query processing, where each component is embedded separately, and their top matches are aggregated. This approach can potentially handle more nuanced queries.
*   **Efficiency**: `np.argpartition` is used in `StatMatcher.match` for efficient retrieval of top-k elements, avoiding a full sort of similarity scores.
*   **File Organization**: The `INDEX_BASEPATH` and `INDEX_MAP` global variables enforce a structured way of organizing and accessing different statistical indexes based on `sport_code` and `entity`. This makes the system scalable and manageable for multiple domains.
*   **CLI**: The `typer` library provides a user-friendly and type-hinted command-line interface, which simplifies interaction with the indexing and matching functionalities.
*   **JSON for Data**: Statistical mappings and test queries are expected in JSON format, facilitating easy data exchange and readability.
*   **Error Handling**: Basic error handling is present in `StatMatcher.from_sport_and_entity` to check for the existence of index files. More comprehensive error handling (e.g., for malformed JSON, missing keys) could be added for production readiness.
