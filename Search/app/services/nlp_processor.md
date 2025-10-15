# Documentation for `nlp_processor.py`

This document provides a comprehensive technical overview and detailed explanation of the `NlpProcessor` Python script.

---

## Technical Documentation: `NlpProcessor`

### 1. High-Level Overview

The `NlpProcessor` class is a core component responsible for converting natural language queries into structured Analytics Query Language (AQL) using Large Language Models (LLMs). It integrates with several internal and external systems to achieve this, including a prompt version control system (Folio), dynamic data mapping handlers, and a PostgreSQL database for contextual information.

**Primary Purpose:** To abstract the complexity of natural language understanding and translation into machine-readable query structures, enabling applications to process user queries efficiently across different sports and entities while maintaining data consistency and prompt version control.

**Key Capabilities:**
*   **Natural Language to AQL Conversion:** Leverages LLMs to transform user-friendly text into structured AQL objects.
*   **Prompt Versioning:** Manages and retrieves LLM prompts using Folio, ensuring prompt consistency and traceability.
*   **Dynamic Mappings:** Incorporates sport-specific and entity-specific data mappings (e.g., statistics, positions, divisions, query examples) to enrich LLM context.
*   **Team Context Integration:** Automatically includes relevant team information (e.g., team name) into the generated AQL qualifiers.
*   **RAG (Retrieval Augmented Generation) Support:** Optionally uses RAG for dynamically matching relevant statistics based on the input query, rather than providing all available stats.

### 2. Dependencies and Imports

The script relies on several standard and custom libraries:

*   **`import time`**: Used for measuring the execution time of certain operations, such as stat matching and LLM response generation.
*   **`from typing import List`**: Used for type hinting, specifically `List` to indicate that a variable is expected to be a list.
*   **`from app.llms.openai_connector import OpenAIConnector`**: Imports a concrete implementation of an LLM connector specifically for OpenAI models. This suggests the system is designed to be pluggable with different LLM providers.
*   **`from app.llms.base import BaseLLMConnector`**: Imports the abstract base class for LLM connectors. This promotes polymorphism and allows the `NlpProcessor` to work with any LLM connector adhering to this interface.
*   **`from folioprompts import Folio`**: Imports the `Folio` client, which is responsible for managing and retrieving version-controlled prompts.
*   **`from app.data_config.mappings.mappings_handler import MappingsHandler`**: Imports a handler for loading various configuration mappings (e.g., stat mappings, query samples, position mappings).
*   **`from app.db.postgres_handler import PostgresHandler`**: Imports a handler for interacting with a PostgreSQL database, specifically used here to retrieve team information.
*   **`import json`**: Standard Python library for working with JSON data, used for parsing LLM responses and formatting mapping data.
*   **`from jinja2 import Template`**: Imports Jinja2's `Template` class, used for rendering dynamic prompts by injecting variables into a template string.
*   **`from app.constants import Qualifiers, General`**: Imports application-specific constants. `Qualifiers` are used to define standard keys for AQL qualifiers (e.g., `TEAM_NAME`), while `General` is imported but not directly used in the provided snippet.

### 3. Class: `NlpProcessor`

This class encapsulates the logic for natural language query processing.

#### 3.1. Class Definition and Attributes

```python
class NlpProcessor:
    """
    Converts natural language queries into structured AQL using an LLM with Folio version control.
    
    This class handles the processing of natural language queries by:
    1. Converting them to AQL (Analytics Query Language) format using LLMs
    2. Managing prompt versioning through Folio
    3. Handling dynamic mappings for different sports and entities
    4. Processing team context and qualifiers
    
    Attributes:
        llm (BaseLLMConnector): LLM connector instance for query processing
        folio (Folio): Folio instance for prompt version control
        mappings_handler (MappingsHandler): Handler for loading various mappings
        stat_matchers: Dictionary of stat matchers by sport code and entity
        postgres_handler (PostgresHandler): Handler for PostgreSQL database interactions
    """
```

*   **Purpose**: As described in the docstring, the `NlpProcessor`'s primary role is to convert natural language queries into AQL, managing prompts and mappings in the process.
*   **Attributes**:
    *   `llm (BaseLLMConnector)`: An instance of an LLM connector (e.g., `OpenAIConnector`), used to interact with the LLM API.
    *   `folio (Folio)`: An instance of the `Folio` client, used to retrieve and manage LLM prompts based on their version.
    *   `mappings_handler (MappingsHandler)`: An instance responsible for loading various data mappings (e.g., statistical definitions, example queries, positional mappings) from configuration files or a database.
    *   `stat_matchers`: A dictionary expected to contain specialized "stat matcher" objects, indexed by `sport_code` and `entity`. These matchers are designed to identify relevant statistics within a natural language query, potentially using RAG.
    *   `postgres_handler (PostgresHandler)`: An instance for interacting with the PostgreSQL database, used to fetch contextual data like team names.

#### 3.2. Method: `__init__(self, llm: BaseLLMConnector, stat_matchers)`

```python
    def __init__(self, llm: BaseLLMConnector, stat_matchers):
        """
        Initialize the NLP processor with required dependencies.

        Args:
            llm (BaseLLMConnector): LLM connector instance, defaults to OpenAI GPT-4
            stat_matchers: Dictionary mapping sport codes and entities to their respective stat matchers
        """
        self.llm = llm
        print(f"I am using LLM: {self.llm.model}")
        self.folio = Folio()
        self.mappings_handler = MappingsHandler()
        self.stat_matchers = stat_matchers
        self.postgres_handler = PostgresHandler()
```

*   **Purpose**: The constructor initializes the `NlpProcessor` instance by injecting necessary dependencies and setting up internal handlers.
*   **Parameters**:
    *   `llm (BaseLLMConnector)`: An instantiated LLM connector object. This allows the `NlpProcessor` to be independent of a specific LLM implementation. The docstring mentions a default of OpenAI GPT-4, implying that typically an `OpenAIConnector` instance would be passed.
    *   `stat_matchers`: A dictionary containing objects capable of matching statistics. Its structure is expected to be `stat_matchers[sport_code][entity_type] -> stat_matcher_object`.
*   **Initialization Logic**:
    *   `self.llm = llm`: Stores the provided LLM connector instance.
    *   `print(f"I am using LLM: {self.llm.model}")`: Prints the model name of the initialized LLM, useful for debugging and logging.
    *   `self.folio = Folio()`: Initializes an instance of the Folio client for prompt management.
    *   `self.mappings_handler = MappingsHandler()`: Initializes an instance of the `MappingsHandler` to load configuration data.
    *   `self.stat_matchers = stat_matchers`: Stores the provided dictionary of stat matcher objects.
    *   `self.postgres_handler = PostgresHandler()`: Initializes an instance of the `PostgresHandler` for database operations.

#### 3.3. Method: `async def convert_to_aql(self, basic_query: str, sport_code: str, entity: str, team_code: str, use_rag: bool = True) -> dict`

```python
    async def convert_to_aql(self, basic_query: str, sport_code: str, entity: str, team_code: str, use_rag: bool = True) -> dict:
        """
        Converts a natural language query into structured AQL format using LLM processing.

        This method performs several key steps:
        1. Retrieves team context from the database
        2. Loads appropriate prompt template from Folio
        3. Dynamically loads relevant stat mappings and query samples
        4. Processes the query through LLM
        5. Parses and validates the response
        6. Ensures proper team context in qualifiers

        Args:
            basic_query (str): Natural language query to be converted
            sport_code (str): Sport identifier (e.g., "MFB" for football)
            entity (str): Query target entity type ("Player" or "Team")
            team_code (str): Team identifier code
            use_rag (bool, optional): Whether to use RAG for stat matching. Defaults to True.

        Returns:
            dict: Structured AQL containing:
                - conditions: Query conditions
                - qualifiers: Query qualifiers including team context
                - stat_period: Statistical period for the query

        Raises:
            ValueError: If team not found or if LLM response is invalid
        """
        
        # Fetch team name for context from PostgreSQL
        team_doc = self.postgres_handler.get_team_by_code(team_code)
        if not team_doc or "team_name" not in team_doc:
            raise ValueError(f"No team found for teamCode: {team_code}")

        team_name = team_doc["team_name"]

        context = {Qualifiers.TEAM_NAME.value: team_name}

        # Step 1: Load the latest prompt version from Folio
        prompt_entry = self.folio.get_prompt("nlp_query_conversion")
        if not prompt_entry:
            raise ValueError("Prompt 'nlp_query_conversion' is missing from Folio. Please add it.")

       # print(sport_code, entity, team_code)
        selected_stat_matcher = self.stat_matchers[sport_code][entity]
        # Step 2: Load stat mapping and query samples dynamically
        if use_rag:
            s = time.time()
            stat_mapping = selected_stat_matcher.match(basic_query)
            # print("[DEBUG] Raw stat_mapping from match():", json.dumps(stat_mapping, indent=2))
            e = time.time()
            print(f"Took {e - s:.2f} seconds to load {len(stat_mapping)} stats")
            # print("Stat Mapping ::",stat_mapping)
        else:
            stat_mapping = self.mappings_handler.get_stat_mapping(sport_code, entity)

        stat_mapping = [
            {
                "stat": x["stat"],
                "full_name": x["name"],
                "description": x["description"],
                "category": x.get("category", "Unknown"),
                "aliases": x.get("aliases", []),
                "sub_category": x.get("sub_category", "")
            }
            for x in stat_mapping
        ]

        query_samples = self.mappings_handler.get_query_samples(sport_code, entity)
        pos_mapping = self.mappings_handler.get_pos_mapping(sport_code)
        division_mapping = self.mappings_handler.get_division_mapping(sport_code)


        template = Template(prompt_entry.text)

        # Step 3: Format the prompt dynamically
        formatted_prompt = template.render(
            natural_language_query=basic_query,
            sport_code=sport_code,
            entity=entity,
            context=context,
            stat_mapping=json.dumps(stat_mapping, indent=2),
            query_samples=query_samples,
            pos_mapping=json.dumps(pos_mapping, indent=2),
            division_mapping=division_mapping
        )

        ##### DEBUG STATEMENT !!!
        #print("Formatted Prompt:\n", formatted_prompt)

        # Step 4: Get AQL response from LLM
        s = time.time()
        aql_response = await self.llm.generate_response(
            formatted_prompt  #, response_format="json"  # Ensure LLM returns JSON
        )
        e = time.time()
        print(f"AQL Generation took {e - s:.2f} seconds")
        # Debug: Print response
        print("Raw LLM Response:\n", repr(aql_response))  # Use repr() to see if response is empty or malformed

        # Step 6: Parse response safely
        try:
            if isinstance(aql_response, dict):
                return aql_response  #  Already a dict, return as-is

            aql_response = aql_response.strip("`json\n").strip("`")  # Remove any markdown
            aql_response = json.loads(aql_response)  #  Parse cleaned JSON string
            #  Ensure 'qualifiers' exists and is a dict
            if 'qualifiers' not in aql_response or not isinstance(aql_response['qualifiers'], dict):
                aql_response['qualifiers'] = {}

            # if Qualifiers.TEAM_NAME.value not in aql_response['qualifiers']:
            #     if (Qualifiers.OPPONENT_TEAM_NAME.value not in aql_response['qualifiers']) or (aql_response['qualifiers'][Qualifiers.OPPONENT_TEAM_NAME.value]
            #             != team_name):
            #         aql_response['qualifiers'][Qualifiers.TEAM_NAME.value] = team_name
            return aql_response

        except json.JSONDecodeError:
            raise ValueError(f"Invalid JSON received from LLM:\n{repr(aql_response)}")
```

*   **Purpose**: This asynchronous method is the core logic for converting a natural language query into an AQL structure. It orchestrates the retrieval of contextual data, prompt templating, LLM interaction, and response parsing.
*   **Parameters**:
    *   `basic_query (str)`: The raw natural language query provided by the user (e.g., "who had the most passing yards last season for the Cowboys?").
    *   `sport_code (str)`: A unique identifier for the sport (e.g., "MFB" for Men's Football). Used to fetch sport-specific data.
    *   `entity (str)`: The type of entity the query is about, typically "Player" or "Team". Used for entity-specific mappings.
    *   `team_code (str)`: A unique identifier for the team involved in the query (e.g., "DAL" for Dallas Cowboys). Used to fetch team-specific context.
    *   `use_rag (bool, optional)`: A flag indicating whether Retrieval Augmented Generation (RAG) should be used for stat matching. If `True` (default), a `stat_matcher` dynamically identifies relevant stats. If `False`, all configured stats for the given `sport_code` and `entity` are loaded statically.
*   **Returns**:
    *   `dict`: A dictionary representing the AQL structure. It is expected to contain keys like `conditions`, `qualifiers`, and `stat_period`.
*   **Raises**:
    *   `ValueError`: If the specified `team_code` does not correspond to an existing team, if the required Folio prompt is missing, or if the LLM returns a response that cannot be parsed as valid JSON.
*   **Implementation Logic - Step-by-Step**:

    1.  **Fetch Team Context**:
        *   `team_doc = self.postgres_handler.get_team_by_code(team_code)`: Retrieves team details from PostgreSQL using the `team_code`.
        *   **Validation**: Checks if `team_doc` is found and contains a `team_name` field. If not, a `ValueError` is raised.
        *   `team_name = team_doc["team_name"]`: Extracts the team name.
        *   `context = {Qualifiers.TEAM_NAME.value: team_name}`: Creates a `context` dictionary containing the team name, using a constant from `app.constants.Qualifiers` for the key. This context will be injected into the LLM prompt.

    2.  **Load Prompt from Folio**:
        *   `prompt_entry = self.folio.get_prompt("nlp_query_conversion")`: Fetches the LLM prompt template identified by "nlp_query_conversion" from the Folio system. This ensures consistent and version-controlled prompts.
        *   **Validation**: If `prompt_entry` is not found, a `ValueError` is raised.

    3.  **Load Stat Mappings and Query Samples**:
        *   `selected_stat_matcher = self.stat_matchers[sport_code][entity]`: Selects the appropriate stat matcher object based on the `sport_code` and `entity`.
        *   **Conditional Stat Mapping Loading (`use_rag`)**:
            *   If `use_rag` is `True`:
                *   `stat_mapping = selected_stat_matcher.match(basic_query)`: The `selected_stat_matcher` is invoked with the `basic_query` to identify and return only the most relevant statistics. This is a form of RAG. Execution time is measured and printed.
            *   If `use_rag` is `False`:
                *   `stat_mapping = self.mappings_handler.get_stat_mapping(sport_code, entity)`: The `MappingsHandler` loads all pre-configured statistics for the given `sport_code` and `entity` statically.
        *   **Stat Mapping Transformation**: The raw `stat_mapping` (list of dictionaries) is transformed into a standardized format, ensuring specific keys (`stat`, `full_name`, `description`, `category`, `aliases`, `sub_category`) are present for the LLM. Default values are provided for optional keys.
        *   `query_samples = self.mappings_handler.get_query_samples(sport_code, entity)`: Loads example natural language queries and their corresponding AQL translations to guide the LLM.
        *   `pos_mapping = self.mappings_handler.get_pos_mapping(sport_code)`: Loads mappings for player positions (e.g., "QB" -> "Quarterback").
        *   `division_mapping = self.mappings_handler.get_division_mapping(sport_code)`: Loads mappings for divisions (e.g., "AFC East" -> "AFC East").

    4.  **Format the LLM Prompt with Jinja2**:
        *   `template = Template(prompt_entry.text)`: Creates a Jinja2 template object from the text of the loaded prompt.
        *   `formatted_prompt = template.render(...)`: Renders the template by injecting all the gathered contextual information:
            *   `natural_language_query`: The original user query.
            *   `sport_code`, `entity`: Identifiers.
            *   `context`: The dictionary containing `team_name`.
            *   `stat_mapping`: JSON string of relevant statistics.
            *   `query_samples`: List of example queries.
            *   `pos_mapping`: JSON string of position mappings.
            *   `division_mapping`: List of division mappings.
        *   A commented-out `print` statement allows for debugging the final prompt sent to the LLM.

    5.  **Get AQL Response from LLM**:
        *   `aql_response = await self.llm.generate_response(formatted_prompt)`: Asynchronously calls the LLM connector (`self.llm`) to generate a response based on the fully formatted prompt. The `response_format="json"` parameter is commented out, suggesting it might be handled implicitly by the prompt or the LLM's default behavior, but it's an important consideration for structured output. Execution time is measured and printed.
        *   `print("Raw LLM Response:\n", repr(aql_response))`: Prints the raw LLM response for debugging purposes. `repr()` is used to show potential control characters or malformed strings.

    6.  **Parse and Validate LLM Response**:
        *   **Check for pre-parsed dictionary**: `if isinstance(aql_response, dict): return aql_response`: If the LLM connector already returned a dictionary (meaning it handled JSON parsing internally), it's returned directly.
        *   **Clean and Parse JSON**:
            *   `aql_response = aql_response.strip("`json\n").strip("`")`: Attempts to remove common markdown code block delimiters (e.g., `` ```json `` and `` ``` ``) from the LLM's string response.
            *   `aql_response = json.loads(aql_response)`: Parses the cleaned string as a JSON object into a Python dictionary.
            *   **Error Handling**: If `json.JSONDecodeError` occurs during parsing, a `ValueError` is raised with the malformed response.
        *   **Ensure Qualifiers**:
            *   `if 'qualifiers' not in aql_response or not isinstance(aql_response['qualifiers'], dict): aql_response['qualifiers'] = {}`: Ensures that the returned AQL always has a `qualifiers` key, and that it's a dictionary. If missing or not a dict, it initializes an empty dictionary.
        *   **Commented-out Team Context Logic**: The commented block (`if Qualifiers.TEAM_NAME.value not in aql_response['qualifiers']: ...`) indicates a previous or alternative logic to automatically inject the `team_name` into the AQL `qualifiers` if it wasn't already specified by the LLM (unless it was an opponent query). This logic is currently inactive.
        *   `return aql_response`: Returns the fully processed and validated AQL dictionary.

### 4. Constants Used

*   **`app.constants.Qualifiers`**: An enumeration or class defining standardized string values for AQL qualifiers.
    *   `Qualifiers.TEAM_NAME.value`: Used to refer to the key for a team's name in the AQL qualifiers or context dictionary.
    *   `Qualifiers.OPPONENT_TEAM_NAME.value`: (Used in commented-out code) Would represent the key for an opponent's team name.

### 5. External Integrations

*   **Folio**: Used for managing version-controlled LLM prompts. This ensures that the system always uses the intended prompt, which can be updated and managed externally without code changes.
*   **LLM Connector (`app.llms.base.BaseLLMConnector` and `app.llms.openai_connector.OpenAIConnector`)**: Provides an interface to interact with various LLM providers (e.g., OpenAI). This modular design allows for easy switching or adding of different LLMs.
*   **PostgreSQL (`app.db.postgres_handler.PostgresHandler`)**: Integrated for retrieving structured data, specifically team information, which is crucial for providing context to the LLM.
*   **Jinja2**: A popular templating engine used to dynamically construct the LLM prompt by embedding various data points (natural language query, mappings, context) into a predefined template. This allows for flexible and complex prompt engineering.
*   **`stat_matchers`**: An extensible mechanism for integrating RAG or other intelligent retrieval systems that can dynamically fetch contextually relevant statistics based on a user's query, improving the LLM's ability to generate accurate AQL.

### 6. Main Execution Flow

The provided script defines a class `NlpProcessor` but does not include a `if __name__ == "__main__":` block for direct execution. This indicates that `NlpProcessor` is intended to be instantiated and used as a component within a larger application (e.g., a web service, an API endpoint, or a background processing job). An external service would create an instance of `NlpProcessor`, provide it with an LLM connector and stat matchers, and then call the `convert_to_aql` method to process user queries.
