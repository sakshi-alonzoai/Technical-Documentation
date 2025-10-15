# Documentation for `GeniusToS3Download.py`

# Genius API to S3 Downloader with Folder Checksum - Technical Documentation

## 1. Overview

This Python script facilitates the download of data from the Genius Sports API and uploads it to an Amazon S3 bucket.  It provides both a command-line interface (CLI) and a programmatic interface for flexible usage. A key feature is the computation and storage of a folder-level checksum (SHA256) to ensure data integrity.  The script incorporates memory monitoring and garbage collection to manage memory usage efficiently, especially during large downloads.  Checkpointing is implemented to resume interrupted downloads.  Postgame downloads are supported, allowing downloads with a timestamp regardless of checkpoint status.

## 2. Modules and Dependencies

The script relies on several Python libraries. Ensure these are installed before execution (e.g., using `pip install -r requirements.txt` if a `requirements.txt` file is provided):

* `boto3`: For interacting with Amazon S3.
* `psutil`: For monitoring system resource usage (memory).
* `requests`: For making HTTP requests to the Genius Sports API.
* `typer`: For creating the command-line interface.
* `typing`, `typing_extensions`: For type hinting.
* `hashlib`: For SHA256 checksum calculation.
* `json`: For JSON handling.
* `os`, `time`, `datetime`, `pathlib`, `itertools`, `gc`, `traceback`, `random`, `resource`: Standard Python libraries.


## 3. Class Descriptions

### 3.1 `MemoryMonitor`

This class monitors the Resident Set Size (RSS) memory usage of the process and triggers garbage collection (`gc.collect()`) when a predefined threshold is exceeded.  It also allows setting a hard memory limit using `resource.setrlimit`.

* **`__init__(self, threshold_mb=1024, hard_limit_mb=None, log_interval=10)`:** Initializes the monitor with a memory threshold (in MB) for garbage collection, an optional hard memory limit, and a logging interval.
* **`_set_hard_memory_limit(self, max_memory_mb)`:** Sets a hard memory limit using `resource.setrlimit(resource.RLIMIT_AS)`.  Handles potential exceptions gracefully.
* **`get_memory_mb(self)`:** Returns the current RSS memory usage in MB.
* **`check_memory(self, force_log=False, force_collect=False)`:** Checks memory usage.  Triggers garbage collection if necessary or forced. Logs memory usage at regular intervals.

### 3.2 `DownloadResult`

A simple data structure to encapsulate the outcome of a download operation.

* **`__init__(self, success: bool, job_id: str, ...)`:** Initializes with success status, job ID, S3 path, checksum, messages, and error details.
* **`to_dict(self) -> Dict`:** Converts the result to a dictionary, suitable for API responses.

### 3.3 `GeniusS3Downloader`

The core class responsible for downloading data, managing checkpoints, and interacting with S3.

* **`__init__(self, task: Optional[str] = None, ...)`:** Initializes the downloader. Loads environment variables (AWS credentials, API key), initializes the boto3 S3 client, loads API configurations, and sets up checkpoint management. The constructor gracefully handles missing environment variables file with informative messages and default values.
* **`_load_env_variables(self, env_file: str)`:** Loads environment variables from a JSON file.  Provides a helpful message if the file is missing, including a sample JSON structure.
* **`_load_api_configurations(self)`:** Loads API configurations from JSON files in the 'resources' directory.  Handles cases where configuration files are missing.
* **`_load_checkpoints(self)`:** Loads download checkpoints from a JSON file. Handles file corruption gracefully.
* **`_save_checkpoints(self)`:** Saves the current download checkpoints to a JSON file.
* **`_create_job_id(self, sport, api, params)`:** Creates a unique job ID based on sport, API, and parameters.
* **`_get_checkpoint(self, job_id)`:** Retrieves the checkpoint information for a given job ID.
* **`_update_checkpoint(self, job_id, offset, completed=False)`:** Updates the checkpoint information.
* **`_build_url(self, sport, api, params)`:** Constructs the Genius Sports API URL based on sport, API, and parameters.  Handles cases where sport/api configurations are missing.
* **`_extract_query_params(self, api_info, params)`:** Extracts query parameters for the Genius API request.
* **`_generate_timestamp(self)`:** Generates a timestamp for postgame downloads.
* **`_compute_content_based_checksum(self, all_page_data: List[Dict], sort_keys: Optional[List[str]]) -> str`:** Computes a SHA256 checksum based on the combined data content, handling optional sorting based on specified keys. Includes error handling for sorting failures.
* **`download_data(self, sport: str, api: str, params: Dict[str, Union[str, int]]) -> DownloadResult`:** Downloads data from the Genius Sports API, uploads it to S3, and computes the folder checksum.  Includes robust error handling and retry logic for API requests.  Handles postgame downloads and checkpoint management appropriately.
* **`process_input_file(self, param: str, filename: str, sport: str, api: str) -> List[DownloadResult]`:** Processes a file containing values line by line.
* **`process_multiple_input_files(self, inputs: Dict[str, str], sport: str, api: str) -> List[DownloadResult]`:** Processes multiple input files to generate all combinations.
* **`get_checkpoints(self) -> Dict`:** Returns a copy of the checkpoints dictionary.
* **`reset_checkpoint(self, job_id: str) -> bool`:** Resets a checkpoint to its initial state.


## 4. CLI Interface (`typer`)

The script provides a CLI using the `typer` library.  The following commands are available:

* **`download`:** Downloads data from the Genius Sports API. Requires `--sport` and `--api`.  Accepts parameters either as command-line arguments or from text files.  Supports `--s3-prefix`, `--task`, `--force`, `--postgame`, `--memory-threshold`, and `--memory-limit` options.
* **`checkpoints`:** Lists existing download checkpoints. Supports `--task` for filtering.
* **`reset`:** Resets a specific checkpoint. Requires `job_id` argument and optionally `--task`.


## 5. Programmatic Interface

The script also provides a programmatic interface:

* **`create_downloader(**kwargs)`:** Creates a `GeniusS3Downloader` instance with specified parameters.
* **`download_genius_data(sport, api, params, **downloader_kwargs)`:** A convenience function for performing single downloads.


## 6.  Error Handling and Logging

The script incorporates extensive error handling throughout, catching exceptions and providing informative error messages.  Logging is implemented to track progress and report errors, with a `quiet` mode for suppressing output in programmatic usage.  The script's design promotes resilience against API request failures and file handling issues.


## 7.  Memory Management

Memory management is a crucial aspect of the script's design, especially for handling potentially large datasets. The `MemoryMonitor` class proactively manages memory consumption by triggering garbage collection when the memory usage surpasses a defined threshold.  The use of generators and careful resource management (e.g., closing files) further enhances memory efficiency.  A hard memory limit can optionally be set.


## 8. Checkpointing

The checkpointing mechanism allows interrupted downloads to resume from where they left off.  Checkpoints are stored in JSON files, with a separate file for each task (if specified). This ensures that multiple downloads can run concurrently without interfering with each other.  The checkpoint system is automatically handled by the `GeniusS3Downloader` class.


## 9.  Data Integrity

The script prioritizes data integrity through the computation and storage of SHA256 checksums. Both a page-level and a folder-level (content-based) checksum are generated, ensuring that the downloaded data is complete and accurate. The folder-level checksum is computed after all pages are downloaded, offering a reliable method for verifying data integrity.


## 10.  Postgame Downloads

The script supports postgame downloads, indicated by the `--postgame` flag. This is particularly useful for retrieving data after an event, even if it is already marked as complete in the checkpoint files.  In this mode, the script generates a timestamp to avoid conflicts with existing data.


## 11.  Configuration

The script utilizes an `environment-variables.json` file (or a file specified using the `env_file` argument) for configuration.  This file contains essential parameters, including AWS credentials, the S3 bucket name, and the Genius Sports API key.   This approach keeps sensitive credentials out of the main script and promotes flexibility.  Configuration is handled securely through environment variables, preventing hardcoding sensitive information.

