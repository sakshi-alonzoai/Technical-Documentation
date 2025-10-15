# Documentation for `copy_s3_data.py`

# S3 Folder Copy Tool Documentation

## Overview

This Python script, `s3_copy.py`, facilitates copying the contents of multiple S3 paths into a new S3 "folder".  The new folder is created with a timestamp subfolder to ensure that repeated executions with the same name result in isolated snapshots, preventing accidental overwrites. The script leverages the `boto3` library for interacting with AWS S3 and `typer` for creating a command-line interface.


## Dependencies

- Python 3.7+
- `boto3` (for AWS S3 interaction): Install using `pip install boto3`
- `typer` (for command-line interface): Install using `pip install typer`


## Configuration

The script loads AWS credentials from a JSON configuration file, `environment-variables.json` by default. This file should contain the following keys:

- `AWS_ACCESS_KEY_ID`: Your AWS access key ID.
- `AWS_SECRET_ACCESS_KEY`: Your AWS secret access key.
- `AWS_REGION` (optional): Your AWS region.  If omitted, boto3 will attempt to determine the region automatically.
- `AWS_SESSION_TOKEN` (optional): Your AWS session token (for temporary credentials).

You can specify an alternative configuration file path using the `--config` argument.


## Functions

### `load_config(config_path: Optional[str] = None) -> Dict[str, Any]`

This function loads the AWS credentials from the specified JSON configuration file.

**Args:**

- `config_path`: (Optional) The path to the JSON config file. Defaults to `environment-variables.json`.

**Returns:**

- A dictionary containing the loaded configuration values.

**Raises:**

- `FileNotFoundError`: If the configuration file does not exist.
- `json.JSONDecodeError`: If the configuration file contains invalid JSON.

### `parse_s3_uri(uri: str) -> Tuple[str, str]`

Parses an S3 URI (e.g., `s3://my-bucket/my-key`) into its constituent bucket and key components.

**Args:**

- `uri`: The S3 URI string.

**Returns:**

- A tuple containing the bucket name and key.

**Raises:**

- `ValueError`: If the provided URI is not a valid S3 URI.

### `copy_s3_folder(sources: List[str], dest_folder_name: str, s3_client: Optional[Any] = None, config_path: Optional[str] = None) -> str`

This is the core function of the script. It copies S3 objects or entire prefixes from the specified source URIs to a new destination folder in S3. The destination folder includes a timestamp subfolder to prevent overwrites from subsequent runs.

**Args:**

- `sources`: A list of S3 URIs to copy.  If a source ends with a `/`, it represents a folder prefix; otherwise, it's a single object.  All sources must be within the same S3 bucket.
- `dest_folder_name`: The base name for the new S3 destination folder.
- `s3_client`: (Optional) A pre-configured boto3 S3 client. If not provided, the function will create one using the credentials from the configuration file.
- `config_path`: (Optional) Path to the configuration file containing AWS credentials. Defaults to `environment-variables.json`.


**Returns:**

- The S3 URI of the newly created destination folder (including the timestamp subfolder and a trailing slash).

**Raises:**

- `ValueError`: If the source URIs do not all belong to the same bucket.
- `RuntimeError`: If there is an error loading AWS credentials.


### `copy_command(s3_uris: List[str], name: str)`

This function is the Typer command handler for the `copy` command. It orchestrates the copying process by calling `copy_s3_folder` and prints the resulting S3 URI to the console.

**Args:**

- `s3_uris`: A list of S3 URIs to copy (obtained from the command-line arguments).
- `name`: The base name for the new S3 folder (obtained from the command-line arguments).


### `create_s3_folder_copy_tool() -> typer.Typer`

This function exposes the Typer app for programmatic use, allowing integration with other applications.


## Command-Line Interface

The script provides a command-line interface using `typer`.  To use it:

```bash
python s3_copy.py copy --s3-uris s3://my-bucket/source1/ s3://my-bucket/source2/ --name my-snapshot
```

Replace `s3://my-bucket/source1/`, `s3://my-bucket/source2/`, and `my-snapshot` with your actual S3 URIs and desired folder name.  Remember that folder prefixes must end with a `/`.


## Error Handling

The script includes comprehensive error handling, including checks for invalid S3 URIs, missing configuration files, incorrect JSON format, and AWS credential loading failures.  Error messages are informative, aiding in debugging.

##  Implementation Details

The script handles both individual object copies and the copying of entire folder prefixes.  For folder prefixes, it uses the S3 `list_objects_v2` API to efficiently retrieve all objects within the prefix and copies them individually to the destination.  Each source is copied into a uniquely named subfolder within the timestamped folder for organization and error isolation.  The use of `copy_object` ensures efficient transfer as it leverages S3's internal copy mechanism.

