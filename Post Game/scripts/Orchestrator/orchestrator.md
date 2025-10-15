# Documentation for `orchestrator.py`

# Workflow Orchestrator Documentation

## Overview

This Python script acts as a workflow orchestrator for running sport-specific data processing pipelines.  It loads configuration from JSON files, manages environment variables, and executes sequences of pipelines and scripts, handling conditional logic and input parameters. The orchestrator supports three sports: Men's Football (MFB), Men's Basketball (MBB), and Women's Basketball (WBB). It offers both a command-line interface (CLI) and a programmatic API for execution.

## Dependencies

The script relies on the following Python packages:

* `json`: For JSON handling.
* `os`: For interacting with the operating system (environment variables, file system).
* `subprocess`: For running external scripts.
* `datetime`: For time tracking.
* `pathlib`: For path manipulation.
* `typing`: For type hinting.
* `requests`: For making HTTP requests to APIs.
* `typer`: For creating the command-line interface.
* `logging`: For logging messages.


**External Dependencies:** The script assumes the existence of:

*  `environment-variables.json`: A JSON file containing environment variables.
* `workflows.json`: A JSON file defining available workflows, their paths, and descriptions.
* Hopfile pipelines (`.hpl` files):  Pipelines defined in the Hopfile format, specified in `workflows.json`.
* Python scripts (`.py` files):  Python scripts, specified in `workflows.json`.


##  Modules and Functions


### 1. `load_environment_from_json(filename: str = "environment-variables.json") -> None`

Loads environment variables from a JSON file located in the same directory as the script.  It handles `FileNotFoundError` and `json.JSONDecodeError` exceptions, raising informative error messages.

### 2. `WorkflowExecutionResult` Class

A simple class to encapsulate the results of a workflow execution. It stores:

* `success`: Boolean indicating success or failure.
* `execution_time`: Execution time in seconds.
* `error_message`: Error message (if any).
* `start_time`:  The workflow's start time.
* `end_time`: The workflow's end time.


### 3. `WorkflowOrchestrator` Class

This is the core class for managing and executing workflows.

* `__init__(self, sport: str, workflow: str, workflow_inputs: Dict, *, env: Optional[Dict[str, str]] = None)`: The constructor initializes the orchestrator. It:
    * Loads environment variables, either from a JSON file or from a provided `env` dictionary (allowing for nested configurations via `_load_nested_environment`). It explicitly checks for  `HOP_RUN_PATH` and `HOP_PROJECT`.
    * Extracts the `API_BASE_URL` from environment variables or workflow inputs. It handles nested config structures via the `_extract_api_base_url` method.
    * Validates the `sport` input.
    * Loads the workflows configuration from `workflows.json` using `_load_workflows_config`.
    * Determines the workflow path using `_get_workflow_path`.
    * Loads and validates pipeline sequences from `pipeline_sequences.json` using `_load_sequences`.
    * Selects the appropriate pipeline sequence based on conditions and inputs using `_get_sequence`.
    * Validates the selected sequence using `_validate_sequences`.

* `_load_nested_environment(self, env: Dict) -> None`: Loads environment variables from a nested dictionary, flattening the structure and converting keys to uppercase with underscores.  It also handles the specific case of `api.base_url`.

* `_extract_api_base_url(self, env: Optional[Dict]) -> Optional[str]`: Extracts the API base URL from a nested dictionary, specifically looking for `env["api"]["base_url"]`.

* `_load_workflows_config(self) -> Dict`: Loads workflow configurations from `workflows.json`, validating the JSON structure.

* `_get_workflow_path(self) -> Path`: Retrieves the absolute path to the workflow directory based on the sport and workflow name from the configuration.

* `_evaluate_condition(self, condition: Dict, inputs: Dict) -> bool`: Evaluates a conditional expression based on workflow inputs.

* `_get_sequence(self, workflow_config: Dict, inputs: Dict) -> List`: Selects the appropriate sequence of steps to execute based on conditions in the workflow configuration.

* `_load_sequences(self) -> Dict`: Loads pipeline sequences from `pipeline_sequences.json`, validating the structure with `_validate_sequences`.

* `_parse_step_inputs(self, step: Dict) -> List[str]`: Parses and formats input parameters for a step, handling both inputs and config.

* `_run_step(self, step: Dict) -> bool`: Executes a single step (pipeline or script) using `run_pipeline` or `subprocess`, returning a boolean indicating success.  It handles real-time stdout and stderr output.  It also includes API step execution (`_run_api_step`).

* `_validate_sequences(self, sequences: Dict) -> None`:  Recursively validates the structure and contents of the `pipeline_sequences.json` file.  It verifies step types, input formats, and the existence of pipeline and script files.  It ensures correct formatting of conditional sequences and handles legacy formats.

* `_process_config_variables(self, config: Dict, inputs: Dict) -> Dict`: Processes configuration parameters, substituting variables (e.g., `${VAR}`) with values from the inputs dictionary.

* `_replace_variables(self, value: str, inputs: Dict) -> str`: Replaces variables (e.g. `${VAR}`) in a string with values from the inputs dictionary.

* `run(self, start_from: Optional[str] = None) -> WorkflowExecutionResult`: Executes the workflow sequence.  It allows for specifying a starting point.  It returns a `WorkflowExecutionResult`.

* `_run_api_step(self, step: Dict) -> bool`:  Handles API calls within the workflow. It constructs the URL, prepares headers and request body, handles timeouts, and checks the HTTP status code.  It improves timeout handling and provides more detailed error messages.



### 4. `run_workflow(sport: str, workflow: str, inputs: Dict[str, Any], *, env: Dict[str, str], start_from: Optional[str] = None) -> WorkflowExecutionResult`

A function providing a programmatic interface to run a workflow.  It instantiates the `WorkflowOrchestrator` and calls its `run` method. It is designed to be called from other Python code.


### 5. `list_available_workflows() -> List[Dict]`

Lists all available workflows from `workflows.json`.

### 6. Typer CLI (`run_cli` and `list_workflows`)

The script uses the `typer` library to define two command-line commands:

* `run_cli`: Runs a specified workflow from the command line, taking sport, workflow name, inputs (as a JSON string), and an optional starting point as arguments.
* `list_workflows`: Lists available workflows from `workflows.json`.


## Error Handling

The script includes comprehensive error handling throughout, catching exceptions such as `FileNotFoundError`, `json.JSONDecodeError`, `ValueError`, and `subprocess.CalledProcessError`.  Error messages are detailed and informative, aiding in debugging.

## Logging

The script utilizes Python's `logging` module to provide detailed logs during workflow execution, including start and end times, step progress, and error messages.  This enhances debugging and monitoring capabilities.


##  Improvements

* **Improved Error Handling:** More robust exception handling, particularly in the `_run_step` and `_run_api_step` methods.  Informative error messages provide detailed context for debugging.

* **Nested Configuration:**  The `_load_nested_environment` and `_extract_api_base_url` methods allow for nested configuration in the environment variables, enhancing flexibility.

* **API Call Enhancements:**  The `_run_api_step` method includes improvements like increased timeout and better error reporting from API responses.

* **Input Validation:**  Input validation is strengthened, particularly around JSON parsing and type checking.

* **Conditional Sequencing:** Enhanced support for conditional execution of different sequences within a workflow, providing greater control and flexibility.

* **Clearer Logging:** Increased logging provides better monitoring of the workflow execution.

* **Documentation:** Extensive documentation provides a comprehensive guide to the script's functionality and usage.


This documentation provides a complete and detailed overview of the provided Python script.  It is suitable for developers needing to understand, use, extend, or maintain this code.

