# Documentation for `logging_mixin.py`

# Python Script Documentation: LoggingMixin Class

## Overview

The `LoggingMixin` class is designed to provide a standardized and reusable set of logging and file handling functionalities for Python scripts. It simplifies the process of setting up logging, saving results (including Pandas DataFrames), and managing error logs.  The class leverages the `pathlib`, `datetime`, `logging`, `json`, and `pandas` libraries.


## Class Structure and Functionality

The `LoggingMixin` class is implemented as a mixin, meaning it's intended to be inherited by other classes to add logging capabilities. It does not define its own core functionality beyond logging and file management.

### `__init__(self, script_name: str)`

* **Purpose:** Initializes the logging system upon instantiation.
* **Arguments:**
    * `script_name (str)`:  The name of the script using this mixin.  Used to create uniquely-named log directories and files.  Must be provided.
* **Implementation Details:**
    1. Creates a `logs` directory if it doesn't exist.
    2. Creates a timestamped subdirectory within `logs` (e.g., `my_script_20241027_103000`) to store logs for a specific run.
    3. Configures the `logging` module to output to both a file (within the run directory) and the console.  The log level is set to `INFO`.
    4. Stores a logger instance (`self.logger`) for later use.
    5. Logs initialization messages.

### `get_error_filepath(self, error_type: str, process_name: str | None = None, extension: str = "json") -> Path`

* **Purpose:** Constructs the file path for an error log file.  Provides a consistent naming scheme.
* **Arguments:**
    * `error_type (str)`: A string describing the type of error (e.g., "validation_errors").  Required.
    * `process_name (str | None)`: (Optional)  A string specifying the name of the process that generated the error.
    * `extension (str)`: (Optional) The file extension (default is "json").
* **Returns:** A `pathlib.Path` object representing the full path to the error file.

### `save_error_log(self, errors: list[Any] | dict[Any], error_type: str, process_name: str | None = None) -> Path`

* **Purpose:** Saves a list or dictionary of errors to a JSON file.
* **Arguments:**
    * `errors (list[Any] | dict[Any])`:  The error data to be saved.  Can be a list or a dictionary.
    * `error_type (str)`: The type of error (used in file naming). Required.
    * `process_name (str | None)`:  (Optional) Name of the process related to the errors.
* **Returns:** A `pathlib.Path` object representing the path where the error log was saved.
* **Implementation Details:** Uses the `json` library to serialize the error data with proper indentation for readability.

### `save_results(self, results: dict[str, Any], filename: str, subdirectory: str | None = None) -> Path`

* **Purpose:** Saves a dictionary of results to a JSON file.
* **Arguments:**
    * `results (dict[str, Any])`: The results data to be saved. Must be a dictionary.
    * `filename (str)`: The base filename (without extension). Required.
    * `subdirectory (str | None)`: (Optional)  A subdirectory within the run directory to store the results.
* **Returns:** A `pathlib.Path` object representing the path where the results were saved.

### `save_dataframe(self, df: pd.DataFrame, filename: str, extension: str = "csv", subdirectory: str | None = None) -> Path`

* **Purpose:** Saves a Pandas DataFrame to a file. Supports CSV, JSON, and Excel formats.
* **Arguments:**
    * `df (pd.DataFrame)`: The Pandas DataFrame to save. Required.
    * `filename (str)`: The base filename (without extension). Required.
    * `extension (str)`: (Optional)  The file extension ("csv", "json", "excel" or "xlsx"). Defaults to "csv".
    * `subdirectory (str | None)`: (Optional) A subdirectory within the run directory.
* **Returns:** A `pathlib.Path` object representing the path where the DataFrame was saved.
* **Error Handling:** Raises a `ValueError` if an unsupported file extension is provided.

### `log_progress(self, current: int, total: int, operation: str, frequency: int = 1000) -> None`

* **Purpose:** Logs progress updates for long-running operations at a specified frequency.
* **Arguments:**
    * `current (int)`: The current number of items processed. Required.
    * `total (int)`: The total number of items to process. Required.
    * `operation (str)`: A description of the operation. Required.
    * `frequency (int)`: (Optional)  How often to log progress (in number of items). Defaults to 1000.


### `log_summary(self, summary: dict[str, Any], title: str = "Summary") -> None`

* **Purpose:** Logs a formatted summary of results.
* **Arguments:**
    * `summary (dict[str, Any])`: A dictionary containing summary information. Required.
    * `title (str)`: (Optional)  A title for the summary section. Defaults to "Summary".



## Dependencies

* `pathlib`
* `datetime`
* `logging`
* `sys`
* `json`
* `pandas`
* `typing`


## Usage Example

```python
from your_module import LoggingMixin  # Replace your_module

class MyScript(LoggingMixin):
    def __init__(self):
        super().__init__("my_script")
        # ... your script logic ...

    def run(self):
      # ... your script's main function
      self.log_summary({'processed_items':1000, 'errors':5})
      # ...


if __name__ == "__main__":
    script = MyScript()
    script.run()
```

This example shows how to inherit from `LoggingMixin` and use its methods within a custom script.  Remember to replace `"your_module"` with the actual module where you define `LoggingMixin`.

