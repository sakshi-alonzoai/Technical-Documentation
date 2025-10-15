# Documentation for `test_nlp_processor.py`

This document provides a comprehensive, line-by-line technical explanation of the provided Python script. The script is a suite of unit tests for the `NlpProcessor` service, designed to verify its functionality in converting natural language queries into a structured query language (AQL-like) format.

---

### **1. High-Level Overview**

This Python script is a collection of `pytest` unit tests for the `NlpProcessor` class, located in `app.services.nlp_processor`. The primary purpose of these tests is to ensure that `NlpProcessor` correctly processes natural language inputs, integrates with various external services (such as a database, a content management system (`Folio`), a large language model (`LLM`), and a stat matcher), and outputs a well-formed structured query.

The tests meticulously mock all external dependencies of `NlpProcessor` using `unittest.mock.MagicMock` and `unittest.mock.patch` to ensure that only the logic within `NlpProcessor` itself is being tested, isolating potential issues.

### **2. Dependencies and Imports**

The script begins by importing necessary modules and classes:

*   **`import pytest`**: Imports the `pytest` framework, which is used for writing and running the tests.
*   **`from unittest.mock import MagicMock, patch`**: Imports `MagicMock` for creating mock objects with default mock behaviors and `patch` for temporarily replacing objects with mocks during tests.
*   **`from app.services.nlp_processor import NlpProcessor`**: Imports the `NlpProcessor` class, which is the System Under Test (SUT).
*   **`from app.constants import Qualifiers`**: Imports the `Qualifiers` enum or constant definitions, likely used to define standard query parameters or types.

### **3. Fixture: `mock_dependencies`**

```python
@pytest.fixture
def mock_dependencies():
    mock_llm = MagicMock()
    mock_stat_matchers = {"MFB": {"Player": MagicMock()}}
    processor = NlpProcessor(mock_llm, mock_stat_matchers)
    return processor, mock_llm, mock_stat_matchers
```

This section defines a `pytest` fixture named `mock_dependencies`. Fixtures are used to provide a fixed baseline for tests, allowing code to be reused and tests to be more readable.

*   **`@pytest.fixture`**: Decorator that marks the `mock_dependencies` function as a `pytest` fixture. This means `pytest` will execute this function once per test that requests it and pass its return value to the test function.
*   **`def mock_dependencies():`**: Defines the fixture function.
*   **`mock_llm = MagicMock()`**: Creates a `MagicMock` object to simulate the behavior of a Large Language Model (LLM). This mock will be used in `NlpProcessor` without actually calling a real LLM service.
*   **`mock_stat_matchers = {"MFB": {"Player": MagicMock()}}`**: Creates a dictionary representing stat matchers. It's structured to indicate a specific sport/league ("MFB" - likely 'Major Football' or similar) and a player type ("Player"). Inside this structure, another `MagicMock` is placed to simulate a stat matching service for players in MFB.
*   **`processor = NlpProcessor(mock_llm, mock_stat_matchers)`**: Instantiates the `NlpProcessor` class, passing the created `mock_llm` and `mock_stat_matchers` as its dependencies. This ensures that the `NlpProcessor` instance under test uses these mocked components instead of real ones.
*   **`return processor, mock_llm, mock_stat_matchers`**: The fixture returns a tuple containing the initialized `NlpProcessor` instance, the `mock_llm` object, and the `mock_stat_matchers` dictionary. Test functions can then request this fixture and destructure the tuple to get access to these components.

### **4. Test Case: `test_convert_to_aql_success`**

```python
@patch("app.services.nlp_processor.db")
@patch("app.services.nlp_processor.Folio")
@patch("app.services.nlp_processor.MappingsHandler")
def test_convert_to_aql_success(mock_mappings, mock_folio, mock_db, mock_dependencies):
    processor, mock_llm, mock_stat_matchers = mock_dependencies
    mock_team = {Qualifiers.TEAM_NAME.value: "Arkansas"}
    mock_db.__getitem__.return_value.find_one.return_value = mock_team

    mock_folio().get_prompt.return_value.text = "Conditions: {{ natural_language_query }} | {{ context.teamName }}"
    mock_stat_matchers["MFB"]["Player"].match.return_value = [{"stat": "sPoints", "description": "Points scored"}]
    mock_mappings().get_query_samples.return_value = ["query sample"]
    mock_mappings().get_pos_mapping.return_value = {"QB": "Quarterback"}

    llm_output = {
        "conditions": "sPoints > 20",
        "stat_period": "game",
        "qualifiers": {}
    }

    mock_llm.generate_response.return_value = str(llm_output).replace("'", '"')
    mock_llm.generate_response.return_value = '{"conditions": "sPoints > 20", "stat_period": "game", "qualifiers": {}}'

    result = processor.convert_to_aql("Who scored more than 20 points?", "MFB", "Player", "31")

    assert "conditions" in result
    assert result["qualifiers"]["teamName"] == "Arkansas"
```

This test case verifies the successful end-to-end conversion of a natural language query into a structured AQL-like format by `NlpProcessor`.

*   **`@patch(...)` decorators**: These decorators, applied in reverse order of execution (from bottom to top), replace specific modules or objects with `MagicMock` instances during the test.
    *   **`@patch("app.services.nlp_processor.MappingsHandler")`**: Mocks the `MappingsHandler` class. The mock instance is passed as `mock_mappings` to the test function.
    *   **`@patch("app.services.nlp_processor.Folio")`**: Mocks the `Folio` class. The mock instance is passed as `mock_folio`. `Folio` likely represents a content or prompt management system.
    *   **`@patch("app.services.nlp_processor.db")`**: Mocks the `db` object (likely a database client or connection). The mock instance is passed as `mock_db`.
*   **`def test_convert_to_aql_success(...)`**: Defines the test function. It receives the `mock_mappings`, `mock_folio`, and `mock_db` from the `@patch` decorators, and `mock_dependencies` from the `pytest` fixture.
*   **`processor, mock_llm, mock_stat_matchers = mock_dependencies`**: Unpacks the `mock_dependencies` fixture's return value to get the `NlpProcessor` instance and its internal mocks.
*   **`mock_team = {Qualifiers.TEAM_NAME.value: "Arkansas"}`**: Defines a sample team dictionary to be returned by the mocked database. `Qualifiers.TEAM_NAME.value` ensures consistency with the application's constant definitions.
*   **`mock_db.__getitem__.return_value.find_one.return_value = mock_team`**: Configures the `mock_db`. This simulates `db['collection_name'].find_one(...)` returning the `mock_team`. `__getitem__` is used to mock dictionary-like access (e.g., `db["teams"]`).
*   **`mock_folio().get_prompt.return_value.text = "Conditions: {{ natural_language_query }} | {{ context.teamName }}"`**: Configures `mock_folio`. It simulates `Folio().get_prompt('nlp_query_conversion').text` (assuming 'nlp_query_conversion' is the prompt name) returning a specific templated string for the LLM. This string includes placeholders for the natural language query and team name.
*   **`mock_stat_matchers["MFB"]["Player"].match.return_value = [{"stat": "sPoints", "description": "Points scored"}]`**: Configures the `mock_stat_matchers`. It simulates the stat matcher for "MFB" and "Player" finding a stat like "sPoints" when asked to match against the query.
*   **`mock_mappings().get_query_samples.return_value = ["query sample"]`**: Configures `mock_mappings` to return sample queries.
*   **`mock_mappings().get_pos_mapping.return_value = {"QB": "Quarterback"}`**: Configures `mock_mappings` to return a position (POS) mapping.
*   **`llm_output = { ... }`**: Defines a dictionary representing the expected structured output from the LLM.
*   **`mock_llm.generate_response.return_value = str(llm_output).replace("'", '"')`**: Configures `mock_llm` to return the `llm_output` dictionary, converted to a JSON string. The `replace("'", '"')` is necessary because `str()` in Python uses single quotes, while JSON requires double quotes.
*   **`mock_llm.generate_response.return_value = '{"conditions": "sPoints > 20", "stat_period": "game", "qualifiers": {}}'`**: This line overwrites the previous `return_value` with an explicit JSON string. This is the final and effective mock for the LLM's response.
*   **`result = processor.convert_to_aql("Who scored more than 20 points?", "MFB", "Player", "31")`**: Calls the `convert_to_aql` method of the `NlpProcessor` instance with a sample natural language query, sport code, player type, and team code.
*   **`assert "conditions" in result`**: Asserts that the returned `result` dictionary contains a key named "conditions".
*   **`assert result["qualifiers"]["teamName"] == "Arkansas"`**: Asserts that the "qualifiers" dictionary within the `result` contains a "teamName" key with the value "Arkansas", demonstrating that the team context was correctly integrated.

### **5. Test Case: `test_convert_to_aql_team_not_found`**

```python
@patch("app.services.nlp_processor.db")
def test_convert_to_aql_team_not_found(mock_db, mock_dependencies):
    processor, *_ = mock_dependencies
    mock_db.__getitem__.return_value.find_one.return_value = None

    with pytest.raises(ValueError, match="No team found for teamCode"):
        processor.convert_to_aql("test", "MFB", "Player", "99")
```

This test case verifies `NlpProcessor`'s error handling when a specified team code does not correspond to an existing team in the database.

*   **`@patch("app.services.nlp_processor.db")`**: Mocks the `db` object, similar to the previous test.
*   **`def test_convert_to_aql_team_not_found(mock_db, mock_dependencies):`**: Defines the test function, receiving `mock_db` and `mock_dependencies`.
*   **`processor, *_ = mock_dependencies`**: Unpacks the fixture, but only the `processor` instance is needed, so other returned values are ignored using `*_`.
*   **`mock_db.__getitem__.return_value.find_one.return_value = None`**: Configures `mock_db` to simulate a scenario where `db['collection_name'].find_one(...)` returns `None`, indicating that no team was found for the given `teamCode`.
*   **`with pytest.raises(ValueError, match="No team found for teamCode"):`**: This `pytest` context manager asserts that a `ValueError` will be raised by the code block within it. The `match` parameter further checks if the error message contains the specified string.
*   **`processor.convert_to_aql("test", "MFB", "Player", "99")`**: Calls `convert_to_aql` with a dummy query and a non-existent `teamCode` ("99") to trigger the expected error.

### **6. Test Case: `test_convert_to_aql_missing_prompt` (First Occurrence)**

```python
@patch("app.services.nlp_processor.db")
@patch("app.services.nlp_processor.Folio")
def test_convert_to_aql_missing_prompt(mock_folio, mock_db, mock_dependencies):
    processor, *_ = mock_dependencies
    mock_db.__getitem__.return_value.find_one.return_value = {"teamName": "Alabama"}
    mock_folio().get_prompt.return_value = None

    with pytest.raises(ValueError, match="Prompt 'nlp_query_conversion' is missing"):
        processor.convert_to_aql("test", "MFB", "Player", "31")
```

This test case verifies error handling when the required "nlp_query_conversion" prompt is not found in the `Folio` system.

*   **`@patch("app.services.nlp_processor.db")` and `@patch("app.services.nlp_processor.Folio")`**: Mocks the `db` and `Folio` objects.
*   **`def test_convert_to_aql_missing_prompt(...)`**: Defines the test function.
*   **`processor, *_ = mock_dependencies`**: Retrieves the `processor` instance.
*   **`mock_db.__getitem__.return_value.find_one.return_value = {"teamName": "Alabama"}`**: Configures `mock_db` to successfully find a team, as the focus of this test is on the `Folio` prompt.
*   **`mock_folio().get_prompt.return_value = None`**: Configures `mock_folio` to simulate that the `get_prompt` method returns `None`, indicating that the prompt is missing.
*   **`with pytest.raises(ValueError, match="Prompt 'nlp_query_conversion' is missing"):`**: Asserts that a `ValueError` with the specific message "Prompt 'nlp_query_conversion' is missing" is raised.
*   **`processor.convert_to_aql("test", "MFB", "Player", "31")`**: Calls `convert_to_aql` to trigger the prompt lookup and subsequent error.

### **7. Test Case: `test_convert_to_aql_missing_prompt` (Second Occurrence - Invalid LLM JSON)**

```python
@patch("app.services.nlp_processor.db")
@patch("app.services.nlp_processor.Folio")
def test_convert_to_aql_missing_prompt(mock_folio, mock_db, mock_dependencies):
    processor, mock_llm, *_ = mock_dependencies
    mock_llm.generate_response.return_value = "INVALID_JSON"
    mock_db.__getitem__.return_value.find_one.return_value = {"teamName": "Alabama"}
    mock_folio().get_prompt.return_value.text = "{{ natural_language_query }}"
    mock_llm.generate_response.return_value = "INVALID_JSON"

    with pytest.raises(ValueError, match="Invalid JSON received from LLM"):
        processor.convert_to_aql("query", "MFB", "Player", "31")
```

**Note**: This function name `test_convert_to_aql_missing_prompt` is a duplicate of the previous test. In a real project, this would typically be renamed to something more descriptive like `test_convert_to_aql_invalid_llm_json_response` to avoid confusion and properly reflect its purpose.

This test case specifically verifies error handling when the Large Language Model (LLM) returns a response that is not valid JSON.

*   **`@patch("app.services.nlp_processor.db")` and `@patch("app.services.nlp_processor.Folio")`**: Mocks the `db` and `Folio` objects.
*   **`def test_convert_to_aql_missing_prompt(...)`**: Defines the test function.
*   **`processor, mock_llm, *_ = mock_dependencies`**: Retrieves the `processor` and `mock_llm` instances from the fixture.
*   **`mock_llm.generate_response.return_value = "INVALID_JSON"` (first instance)**: Configures the LLM to return a non-JSON string. This line is redundant as it is immediately overwritten by the second instance.
*   **`mock_db.__getitem__.return_value.find_one.return_value = {"teamName": "Alabama"}`**: Configures `mock_db` to successfully find a team.
*   **`mock_folio().get_prompt.return_value.text = "{{ natural_language_query }}"`**: Configures `mock_folio` to return a valid prompt text, ensuring the test focuses solely on the LLM's response.
*   **`mock_llm.generate_response.return_value = "INVALID_JSON"` (second instance)**: This is the effective configuration for `mock_llm`, ensuring it returns a string that cannot be parsed as JSON.
*   **`with pytest.raises(ValueError, match="Invalid JSON received from LLM"):`**: Asserts that a `ValueError` will be raised with the specific message "Invalid JSON received from LLM".
*   **`processor.convert_to_aql("query", "MFB", "Player", "31")`**: Calls `convert_to_aql`, which will attempt to parse the "INVALID_JSON" string from the LLM, leading to the expected error.

---
