# Documentation for `test_stat_matcher.py`

This document provides a comprehensive, line-by-line technical explanation of the provided Python script.

---

## Technical Documentation: `test_stat_matcher.py`

### 1. High-Level Overview

This Python script (`test_stat_matcher.py`) is a suite of unit tests for the `StatMatcher` and `StatIndex` classes, presumably located in `app.services.stat_matcher`. It uses the `pytest` framework for test execution and `unittest.mock` for isolating the `StatMatcher` class from its dependencies. The tests verify the core functionalities of `StatMatcher`, including its ability to match user queries to predefined statistics, its factory method for creating instances from sport and entity types, and a method for retrieving all available matchers.

### 2. Dependencies and Imports

The script begins by importing necessary modules:

```python
import pytest
```
*   **`pytest`**: The testing framework used to define and run the tests.

```python
from unittest.mock import MagicMock, patch
```
*   **`unittest.mock`**: A standard library module for creating mock objects.
    *   **`MagicMock`**: A flexible mock object that can act as a substitute for real objects during tests. It automatically creates attributes and methods as they are accessed.
    *   **`patch`**: A decorator or context manager used to temporarily replace objects with mocks, primarily for isolating the code under test from its dependencies.

```python
from app.services.stat_matcher import StatMatcher, StatIndex
```
*   **`app.services.stat_matcher`**: This indicates that the `StatMatcher` and `StatIndex` classes are part of an application structure, specifically within the `services` sub-package.
    *   **`StatMatcher`**: The primary class being tested, responsible for matching text queries to statistical metrics.
    *   **`StatIndex`**: A dependency of `StatMatcher`, likely responsible for indexing and storing the statistical data and their vector representations.

```python
import numpy as np
```
*   **`numpy`**: A fundamental library for numerical computing in Python, especially for working with arrays and matrices. It is used here to create mock array-based return values for the `StatIndex` model.

### 3. Function Explanations

#### 3.1. `mock_stat_index()`

This function is a `pytest` fixture, meaning it's a special function that provides setup and teardown for tests. It's designed to create a mock `StatIndex` object that can be injected into tests, allowing `StatMatcher` to be tested without needing a real `StatIndex` instance or its underlying machine learning model.

```python
@pytest.fixture
def mock_stat_index():
    # ...
    return index
```
*   **`@pytest.fixture`**: Decorator that marks `mock_stat_index` as a pytest fixture. When a test function requests `mock_stat_index` as an argument, pytest will call this function and pass its return value to the test.

```python
    index = MagicMock(spec=StatIndex)
```
*   **`index = MagicMock(spec=StatIndex)`**: Creates a `MagicMock` object that "mimics" the `StatIndex` class. The `spec=StatIndex` argument ensures that the mock has the same attributes and methods as `StatIndex`, and will raise an `AttributeError` if an attempt is made to access an attribute or method not present in the real `StatIndex` class. This helps catch typos and ensures the mock behaves realistically.

```python
    model = MagicMock()
    model.encode.return_value = np.array([[0.1, 0.2], [0.3, 0.4]])
```
*   **`model = MagicMock()`**: Creates another `MagicMock` to simulate the machine learning model typically held by `StatIndex`.
*   **`model.encode.return_value = np.array([[0.1, 0.2], [0.3, 0.4]])`**: Configures the mocked `model` such that whenever its `encode` method is called, it returns a predefined NumPy array. This array represents dummy vector embeddings (e.g., for query strings). Here, it's a 2x2 array.

```python
    # Ensure first dimension (rows) matches len(corpus_embeddings)
    model.similarity.return_value = np.array([
        [0.99, 0.01],  # sPoints match high
        [0.20, 0.10],  # sYards match low
    ])
```
*   **`model.similarity.return_value = np.array(...)`**: Configures the mocked `model` so that its `similarity` method returns a predefined NumPy array. This array simulates the similarity scores between a query embedding and the corpus embeddings.
    *   The comment `Ensure first dimension (rows) matches len(corpus_embeddings)` suggests that the first row `[0.99, 0.01]` might represent the similarity of the first `corpus_embedding` with two query embeddings (or vice-versa, depending on the `StatMatcher`'s internal logic). Given the later `match` call, it's more likely this represents the similarity of *a single query* against *multiple corpus embeddings*. So, `0.99` would be the similarity to `sPoints` and `0.01` to `sYards` for one query, and `0.20` to `sPoints` and `0.10` to `sYards` for another query. However, for a single query (like "Points and Yards" in the test), we'd expect a 1xN array or an Nx1 array of similarities. The structure `[[0.99, 0.01], [0.20, 0.10]]` implies *two* corpus embeddings, each yielding *two* similarity scores with something. Let's assume for the test it means the `StatMatcher` will process a query, generate an embedding, and then `model.similarity` will be called with that query embedding and `corpus_embeddings`. If the `model.similarity` takes two sets of embeddings and returns a matrix of pairwise similarities, then this mock means for the query "Points and Yards", its similarity to `sPoints` is `0.99` and to `sYards` is `0.01` (from the first row of the returned array, assuming `StatMatcher` extracts the first row's values).

```python
    index.model = model
```
*   **`index.model = model`**: Assigns the mocked `model` to the `model` attribute of the mocked `StatIndex` object.

```python
    # Make sure this matches the row count in similarity
    index.corpus_embeddings = np.array([[0.1, 0.2], [0.3, 0.4]])
```
*   **`index.corpus_embeddings = np.array([[0.1, 0.2], [0.3, 0.4]])`**: Sets the `corpus_embeddings` attribute of the mock `StatIndex`. These are the pre-computed embeddings for the known statistics. The comment indicates it should match the number of rows expected by the `similarity` method (in this case, 2 embeddings).

```python
    # Mapping should match corpus_embeddings order
    index.stat_mapping = [
        {"stat": "sPoints", "description": "Points scored"},
        {"stat": "sYards", "description": "Yards gained"},
    ]
    index.ind2stat = ["sPoints", "sYards"]
    index.stat2ind = {"sPoints": 0, "sYards": 1}
```
*   These lines set up internal data structures within the mock `StatIndex` that map between statistical identifiers, their descriptions, and their indices within the `corpus_embeddings` array.
    *   **`index.stat_mapping`**: A list of dictionaries, where each dictionary represents a statistic with its short identifier (`"stat"`) and a human-readable `description`. The order here corresponds to the order of `corpus_embeddings`.
    *   **`index.ind2stat`**: A list mapping an index (its position) to a stat identifier (e.g., `ind2stat[0]` is `"sPoints"`).
    *   **`index.stat2ind`**: A dictionary mapping a stat identifier to its index (e.g., `stat2ind["sPoints"]` is `0`).

```python
    return index
```
*   **`return index`**: The fixture returns the fully configured `MagicMock` object, which will be provided to any test that requests `mock_stat_index`.

#### 3.2. `test_stat_matcher_match_returns_top_k()`

This test function verifies that the `StatMatcher.match()` method correctly processes a query and returns a list of top `k` matched statistics.

```python
def test_stat_matcher_match_returns_top_k(mock_stat_index):
```
*   **`def test_stat_matcher_match_returns_top_k(mock_stat_index):`**: Defines a pytest test function. It requests the `mock_stat_index` fixture, which pytest will provide.

```python
    matcher = StatMatcher(mock_stat_index)
```
*   **`matcher = StatMatcher(mock_stat_index)`**: Instantiates `StatMatcher` by passing the mocked `StatIndex` object. This ensures `StatMatcher` uses the predefined mock behaviors for `encode`, `similarity`, and stat mappings.

```python
    results = matcher.match("Points and Yards", top_k=2)
```
*   **`results = matcher.match("Points and Yards", top_k=2)`**: Calls the `match` method of the `StatMatcher` instance with a sample query string and requests the top 2 results. Based on the `mock_stat_index` setup, "sPoints" (0.99 similarity) should be ranked higher than "sYards" (0.01 similarity).

```python
    assert isinstance(results, list)
```
*   **`assert isinstance(results, list)`**: Asserts that the return value `results` is a list.

```python
    assert len(results) > 0
```
*   **`assert len(results) > 0`**: Asserts that the list of results is not empty. Given `top_k=2` and two available stats, it's expected to have 2 results.

```python
    assert all("stat" in item for item in results)
```
*   **`assert all("stat" in item for item in results)`**: Asserts that every item in the `results` list is a dictionary (or dictionary-like object) that contains a key named `"stat"`. This verifies the structure of the returned items.

```python
    assert any(item["stat"] in ["sPoints", "sYards"] for item in results)
```
*   **`assert any(item["stat"] in ["sPoints", "sYards"] for item in results)`**: Asserts that at least one of the returned items has a `"stat"` value of either `"sPoints"` or `"sYards"`. This confirms that the matching process correctly identified expected statistics. Given the mock setup, both should ideally be present in the top 2.

#### 3.3. `test_stat_matcher_from_sport_and_entity()`

This test function verifies the `StatMatcher.from_sport_and_entity()` class method, which is responsible for creating a `StatMatcher` instance based on a sport and entity type (e.g., "Football" and "Player"). This method likely loads a specific `StatIndex` configuration.

```python
@patch("app.services.stat_matcher.StatIndex")
def test_stat_matcher_from_sport_and_entity(mock_stat_index_class):
```
*   **`@patch("app.services.stat_matcher.StatIndex")`**: This decorator replaces the actual `StatIndex` class (wherever it's imported in `app.services.stat_matcher`) with a `MagicMock` object for the duration of this test. `mock_stat_index_class` will be this `MagicMock` representing the `StatIndex` class itself.

```python
    mock_index = MagicMock()
    mock_index.load = MagicMock()
```
*   **`mock_index = MagicMock()`**: Creates a `MagicMock` instance. This will represent the *instance* of `StatIndex` that `StatMatcher.from_sport_and_entity` is expected to create.
*   **`mock_index.load = MagicMock()`**: Configures the `mock_index` such that its `load` method is also a `MagicMock`. This allows us to track if `load` was called and with what arguments.

```python
    mock_stat_index_class.return_value = mock_index
```
*   **`mock_stat_index_class.return_value = mock_index`**: This is crucial. It tells the patched `StatIndex` *class* (represented by `mock_stat_index_class`) that whenever it is instantiated (i.e., `StatIndex(...)` is called), it should return our `mock_index` *instance* instead of a real `StatIndex` object.

```python
    with patch("app.services.stat_matcher.INDEX_MAP", {"MFB": {"Player": "mock_path"}}):
```
*   **`with patch("app.services.stat_matcher.INDEX_MAP", {"MFB": {"Player": "mock_path"}}):`**: This uses `patch` as a context manager to temporarily replace a hypothetical global or module-level dictionary named `INDEX_MAP` within `app.services.stat_matcher`. This `INDEX_MAP` is assumed to store paths to `StatIndex` configurations, keyed by sport and entity type. For this test, it's set to a simplified dictionary: for "MFB" (Men's Football), "Player" entity, the path is "mock_path".

```python
        matcher = StatMatcher.from_sport_and_entity("MFB", "Player")
```
*   **`matcher = StatMatcher.from_sport_and_entity("MFB", "Player")`**: Calls the class method under test with example arguments. This call is expected to:
    1.  Look up `"mock_path"` in the (patched) `INDEX_MAP`.
    2.  Instantiate `StatIndex()` (which will actually return `mock_index` due to the earlier patch).
    3.  Call `mock_index.load("mock_path")`.
    4.  Return a `StatMatcher` instance initialized with `mock_index`.

```python
        assert isinstance(matcher, StatMatcher)
```
*   **`assert isinstance(matcher, StatMatcher)`**: Asserts that the object returned by the class method is indeed an instance of `StatMatcher`.

```python
        mock_index.load.assert_called_once_with("mock_path")
```
*   **`mock_index.load.assert_called_once_with("mock_path")`**: Verifies that the `load` method of the mock `StatIndex` instance was called exactly once, and specifically with `"mock_path"` as its argument. This confirms that the `from_sport_and_entity` method correctly identified and used the configuration path.

#### 3.4. `test_stat_matcher_get_all_stat_matchers()`

This test function verifies the `StatMatcher.get_all_stat_matchers()` class method, which is expected to return a dictionary of `StatMatcher` instances for all configured sport and entity combinations.

```python
@patch("app.services.stat_matcher.StatMatcher.from_sport_and_entity")
def test_stat_matcher_get_all_stat_matchers(mock_from_sport_and_entity):
```
*   **`@patch("app.services.stat_matcher.StatMatcher.from_sport_and_entity")`**: This decorator patches the `from_sport_and_entity` class method of `StatMatcher`. Instead of calling the real implementation, the test will use the `MagicMock` provided as `mock_from_sport_and_entity`. This isolates `get_all_stat_matchers` from the internal logic of `from_sport_and_entity`.

```python
    dummy_matcher = MagicMock()
```
*   **`dummy_matcher = MagicMock()`**: Creates a simple `MagicMock` object. This will serve as a placeholder for the `StatMatcher` instance that `from_sport_and_entity` is supposed to return for each configuration.

```python
    mock_from_sport_and_entity.side_effect = lambda s, e: dummy_matcher
```
*   **`mock_from_sport_and_entity.side_effect = lambda s, e: dummy_matcher`**: Configures the patched `from_sport_and_entity` method. `side_effect` allows you to define a function that will be called whenever the mock is called. Here, it's a `lambda` function that takes `s` (sport) and `e` (entity) arguments (though they aren't used in this lambda) and always returns `dummy_matcher`. This ensures that `get_all_stat_matchers` will receive the same mock object for every sport/entity combination it processes.

```python
    with patch("app.services.stat_matcher.INDEX_MAP", {
        "MFB": {"Player": "path1"},
        "MBB": {"Team": "path2"},
        "WBB": {"Team": "path2"},
    }):
```
*   **`with patch("app.services.stat_matcher.INDEX_MAP", {...}):`**: Similar to the previous test, this temporarily replaces the `INDEX_MAP` with a dictionary containing multiple sport/entity configurations. This allows `get_all_stat_matchers` to iterate over a realistic set of configurations.

```python
        matchers = StatMatcher.get_all_stat_matchers()
```
*   **`matchers = StatMatcher.get_all_stat_matchers()`**: Calls the static/class method under test. This method is expected to iterate through the `INDEX_MAP` and call `StatMatcher.from_sport_and_entity` for each sport-entity pair, collecting the returned `StatMatcher` instances into a nested dictionary.

```python
    assert isinstance(matchers, dict)
```
*   **`assert isinstance(matchers, dict)`**: Asserts that the top-level return value is a dictionary.

```python
    assert matchers["MFB"]["Player"] == dummy_matcher
    assert matchers["MBB"]["Team"] == dummy_matcher
    assert matchers["WBB"]["Team"] == dummy_matcher
```
*   **`assert matchers[...] == dummy_matcher`**: These assertions verify that the `matchers` dictionary contains the correct structure (nested dictionaries for sport and entity) and that for each configuration, the value stored is the `dummy_matcher` instance, which was expected to be returned by the patched `from_sport_and_entity` method. This confirms that `get_all_stat_matchers` correctly iterated through all configurations and collected the results.

### 4. Main Execution Flow

This script does not contain an `if __name__ == "__main__":` block. This is typical for `pytest` test files, as they are not meant to be executed directly as standalone scripts. Instead, `pytest` is invoked from the command line (e.g., `pytest test_stat_matcher.py` or `pytest`) and it automatically discovers and runs the test functions (those starting with `test_`).

### 5. Important Implementation Details, Dependencies, and Configuration Variables

*   **Test-driven Development (TDD) / Unit Testing**: This script exemplifies good unit testing practices by isolating components and mocking dependencies.
*   **`pytest`**: The chosen testing framework provides fixtures, clear test discovery, and powerful assertion rewriting.
*   **`unittest.mock`**: Essential for dependency injection and creating isolated tests. `MagicMock` provides flexible mock objects, and `patch` allows for temporary replacement of objects during tests.
*   **`numpy`**: Used to simulate array-based data, which is common in machine learning contexts (embeddings, similarity scores).
*   **Assumed `app.services.stat_matcher` structure**: The tests make strong assumptions about the `StatMatcher` and `StatIndex` classes:
    *   `StatIndex` has attributes like `model` (which has `encode` and `similarity` methods), `corpus_embeddings`, `stat_mapping`, `ind2stat`, `stat2ind`, and a `load()` method.
    *   `StatMatcher` has an `__init__` method that takes a `StatIndex` object, a `match()` method, and class methods `from_sport_and_entity()` and `get_all_stat_matchers()`.
*   **Configuration variable `INDEX_MAP`**: This is a crucial (mocked) dependency. It is assumed to be a dictionary-like object (likely defined globally or at the module level within `app.services.stat_matcher`) that maps sport and entity types to paths or identifiers for `StatIndex` configurations. Its structure appears to be `{"Sport": {"Entity": "config_path"}}`.
*   **Semantic Search/Embedding Model**: The mock for `StatIndex.model` implies that the underlying `StatMatcher` uses a machine learning model (likely a sentence transformer or similar embedding model) to convert text queries into vector embeddings (`encode`) and then calculate similarities between these query embeddings and a corpus of pre-computed stat embeddings (`similarity`).
*   **Stat Ranking**: The `mock_stat_index` fixture's `model.similarity.return_value` directly influences the expected ranking of stats, with `sPoints` being high and `sYards` being low for the specific query, which `StatMatcher` then interprets for `top_k` results.
