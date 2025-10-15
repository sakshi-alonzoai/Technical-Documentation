# Documentation for `RunPipelines.py`

# Apache Hop Pipeline Runner Documentation

## Overview

This Python script provides a command-line interface (CLI) for executing Apache Hop pipelines.  It leverages the `typer` library for creating a user-friendly CLI experience and handles pipeline execution, logging, error handling, and timing information.  The script allows users to specify pipeline paths, parameters, and logging levels.  Environment variables are loaded from a JSON configuration file for flexibility.

## Dependencies

The script depends on several Python libraries:

- `json`: For parsing the environment variable JSON file.
- `os`: For interacting with the operating system, including environment variables and file system operations.
- `subprocess`: For executing the Apache Hop pipeline runner.
- `time`: For measuring pipeline execution time.
- `datetime`: For creating timestamps in log files.
- `pathlib`: For easier path manipulation.
- `typing`: For type hinting.
- `typer`: For creating a command-line interface.

These libraries can be installed using pip:  `pip install json os subprocess time datetime pathlib typing typer`

## Configuration

The script loads environment variables from a JSON file named `environment-variables.json`, located in the same directory as the script. This file should contain key-value pairs representing environment variables, for example:

```json
{
  "HOP_RUN_PATH": "/path/to/hop/run",
  "HOP_PROJECT": "MyProject"
}
```

The script requires the `HOP_RUN_PATH` and `HOP_PROJECT` environment variables to be set.  `HOP_RUN_PATH` specifies the path to the Apache Hop pipeline runner executable, and `HOP_PROJECT` specifies the Hop project name.

## Functions

### `load_environment_from_json(filename: str = "environment-variables.json") -> None`

This function loads environment variables from the specified JSON file.  It handles `FileNotFoundError` and `json.JSONDecodeError` exceptions, providing informative error messages to the user. The default filename is "environment-variables.json".

### `run_pipeline(pipeline_path: str, param_pairs: List[str] = None) -> bool`

This function executes the Apache Hop pipeline using the `subprocess` module.  It constructs the command line arguments based on environment variables and user-supplied parameters. The function prints the command being executed for transparency.  Standard output and standard error streams are handled in real-time.  If the pipeline execution fails (non-zero return code), it logs the error details to a file named  `error_{pipeline_name}_{timestamp}.log` in the script's directory and returns `False`.  Otherwise, it returns `True`.

### `run_pipeline_with_timing(pipeline_path: str, param_pairs: List[str] = None) -> bool`

This function wraps `run_pipeline`, adding functionality to measure and display the pipeline execution time. It prints start and end times, total execution time, and success/failure messages.  It calls `run_pipeline` to perform the actual pipeline execution.

### `main(pipeline_path: str, parameters: Optional[List[str]] = None, log_level: str = "Detailed", quiet: bool = False)`

This is the main function of the script, decorated with `@app.command` from the `typer` library to create a command-line interface.  It parses command-line arguments, loads environment variables, validates the pipeline path, runs the pipeline (either with or without timing information depending on the `quiet` flag), and handles exceptions, providing informative error messages to the user.

## Command-Line Interface

The script can be run from the command line using the following syntax:

```bash
python RunPipelines.py <pipeline_path> [parameters] [options]
```

- **`<pipeline_path>`**:  The full path to the `.hpl` pipeline file.  This is a required argument.
- **`[parameters]`**: Optional parameter pairs for the pipeline, in the format `PARAM1=value1 PARAM2=value2`.  These are passed to the Apache Hop pipeline runner.
- **`[options]`**:
    - `--log-level` or `-l`: Sets the logging level (Basic, Detailed, Debug, Rowlevel, Error). Defaults to "Detailed".
    - `--quiet` or `-q`: Suppresses timing and status messages.

## Error Handling

The script includes robust error handling for file not found errors, invalid JSON in the environment file, pipeline execution failures, and other exceptions.  Error messages are clear and informative, guiding the user towards resolving the issue.  Error details are logged to files, which can be helpful for debugging.

## Example Usage

```bash
# Run pipeline with default settings
python RunPipelines.py my_pipeline.hpl

# Run pipeline with parameters
python RunPipelines.py my_pipeline.hpl PARAM1=value1 PARAM2=value2

# Run pipeline with debug logging
python RunPipelines.py my_pipeline.hpl -l Debug

# Run pipeline quietly
python RunPipelines.py my_pipeline.hpl -q
```

