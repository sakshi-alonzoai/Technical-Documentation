# Documentation for `matches_helper.py`

## Technical Documentation: `matches_helper.py`

This document details the Python script `matches_helper.py`, which provides functions for transforming match records from a potentially inconsistent format into a standardized structure suitable for database ingestion or other data processing tasks.  The script leverages the `LoggingMixin` class (assumed to be defined elsewhere and handle logging functionality) for error reporting and logging.

### High-Level Overview

The primary function, `transform_match`, takes a single match record (a dictionary) as input and outputs a list containing two transformed records.  Each output record represents the match from the perspective of a single competitor, with fields prefixed to indicate whether the data pertains to the "team" or "opponent". The script also includes functions for name conversion (`camel_to_snake`), competitor validation (`validate_competitors`), and internal helper functions for data flattening and snake_case conversion.

### Function Details

**1. `camel_to_snake(name: str) -> str`**

This function converts a string from camelCase notation to snake_case.  It handles special cases, notably adding an underscore between an 's' and a following digit (e.g., "s1st" becomes "s_1st"). The conversion is achieved using regular expressions.

**Implementation Details:** Uses `re.sub()` for string manipulation.

**2. `validate_competitors(record: Dict[str, Any]) -> tuple[bool, str]`**

This function validates that a given match record contains exactly two competitors.  It checks the presence and length of the "competitors" list within the input record.

**Returns:** A tuple containing a boolean indicating validity and an explanatory string if invalid.

**3. `transform_match(record: Dict[str, Any]) -> List[Dict[str, Any]]`**

This is the core function of the script. It takes a single match record and transforms it into two records, one for each competitor.  The transformation involves:

* **Validation:** Calls `validate_competitors` to ensure the record has two competitors.  Invalid records are logged and result in an empty list being returned.
* **Base Record Creation:** Creates a base record containing common match information, excluding competitors. Keys are converted to snake_case using `camel_to_snake`.  Venue data (if present as a nested dictionary) is flattened and prefixed with "venue_".  A `match_id` is added for later record linking, derived from `matchId` or `_id`. If available, sport information from the metadata is included.
* **Competitor-Specific Data:** Iterates through the competitors, calling `_flatten_competitor_snake_case` to create flattened dictionaries with keys prefixed by "team_" and "opponent_". This data is merged into separate copies of the base record to generate the two output records.
* **Error Handling:** Uses a `try-except` block to catch and log any exceptions during the transformation process, including a detailed traceback.


**4. `_flatten_competitor_snake_case(competitor: Dict[str, Any], prefix: str = "") -> Dict[str, Any]`**

This is a helper function used by `transform_match`. It takes a competitor dictionary and a prefix (e.g., "team_", "opponent_") and returns a flattened dictionary.  Nested dictionaries are handled recursively; keys are converted to snake_case and prefixed appropriately.  It has special handling for `images` nested dictionaries and their nested `logo` fields. If a "linkDetailCompetitor" field exists, a new field with the provided prefix and "link_detail_competitor" is added.

**Implementation Details:** Uses recursion to handle nested dictionaries.

**5. `transform_match` (exported function)**

The `transform_match` function from the `MatchTransformer` class is exported directly for easy use in external scripts, simplifying access to the core transformation functionality.


### Dependencies

* Python 3.7+ (for type hinting)
* `re` module (for regular expression operations)
* A custom `LoggingMixin` class (assumed to be defined elsewhere and handle logging functionality).


### Usage Example

```python
from helper_scripts.matches_helper import transform_match

match_record = {
    "matchId": 123,
    "matchDate": "2024-03-08",
    "venue": {"name": "Stadium X", "city": "New York"},
    "competitors": [
        {"name": "TeamA", "score": 10, "images": {"logo": {"T1": {"url": "url1"}}}},
        {"name": "TeamB", "score": 5, "linkDetailCompetitor": "linkB"},
    ],
}

transformed_records = transform_match(match_record)
print(transformed_records)
```

This example showcases how to use the exported `transform_match` function to process a sample match record.  The output will be a list of two dictionaries, each representing the match from one competitor's perspective.  Error handling is incorporated to gracefully manage any issues encountered during processing.

