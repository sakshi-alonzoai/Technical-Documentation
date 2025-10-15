# Documentation for `dump_to_s3.py`

# PostgreSQL to S3 Database Dumper

## Overview

This Python script facilitates the dumping of PostgreSQL tables into an S3 bucket.  It leverages the `pg_dump` utility for database extraction and the Boto3 library for S3 interaction. Configuration is managed through a JSON file, allowing for flexible deployment and environment-specific settings. The script is designed to be robust, handling potential errors during both the database dump and S3 upload processes.  It offers both a command-line interface (CLI) using Typer and a programmatic interface for integration into other Python applications.


## Dependencies

The script relies on the following Python packages:

- `typer`: For creating the command-line interface.
- `boto3`: For interacting with AWS S3.
- `json`: For handling JSON configuration files.
- `logging`: For logging events and errors.
- `os`: For interacting with the operating system (e.g., environment variables).
- `subprocess`: For executing the `pg_dump` command.
- `tempfile`: For creating temporary files.
- `pathlib`: For easier path manipulation.
- `typing`: For type hinting.


It also requires the `pg_dump` utility to be installed and accessible in the system's PATH.  The PostgreSQL server must be configured and accessible to the user running the script.  AWS credentials must be provided either through environment variables or directly in the configuration file.


## Configuration (`environment-variables.json`)

The script uses a JSON configuration file (default: `environment-variables.json`) to specify database connection parameters and S3 settings.  The file should contain the following keys:

- `"PG_HOST"`: PostgreSQL server hostname or IP address (required).
- `"PG_PORT"`: PostgreSQL server port (default: 5432).
- `"PG_USERNAME"`: PostgreSQL username (required).
- `"PG_PASSWORD"`: PostgreSQL password (required).  **Note:** This password is read from the environment variable `PGPASSWORD` during execution, improving security.
- `"PG_DATABASE"`: PostgreSQL database name (required).
- `"AWS_ACCESS_KEY_ID"`: AWS access key ID (required).
- `"AWS_SECRET_ACCESS_KEY"`: AWS secret access key (required).
- `"AWS_REGION"`: AWS region (required).
- `"AWS_SESSION_TOKEN"`: (Optional) AWS session token.
- `"S3_BUCKET"`: Name of the S3 bucket (required).
- `"S3_PREFIX"`: (Optional) Prefix for S3 keys.


## Functions

### `load_config(config_path: Optional[str] = None) -> Dict[str, Any]`

Loads configuration settings from the specified JSON file (defaults to `environment-variables.json`).  Handles `FileNotFoundError` and `json.JSONDecodeError` exceptions for robustness.


### `class DatabaseDumper`

This class encapsulates the logic for dumping database tables to S3.

#### `__init__(...)`

The constructor initializes the `DatabaseDumper` object. It accepts parameters for all necessary configurations, including S3 bucket information and PostgreSQL connection details. It prioritizes parameters provided directly to the constructor over those found in the configuration file.  It performs lazy initialization of the S3 client.


#### `@property s3_client`

Provides lazy initialization of the Boto3 S3 client. This ensures the client is only created when needed.


#### `dump_table_to_s3(table_name: str) -> str`

Dumps a single PostgreSQL table to S3.  Uses `pg_dump` to generate the SQL dump to a temporary file, then uploads that file to S3.  Handles potential errors from both `pg_dump` and the S3 upload process using exception handling.  The function cleans up the temporary file after upload.


#### `dump_tables_to_s3(table_names: List[str]) -> List[str]`

Dumps multiple tables to S3, iterating through the provided list of table names and calling `dump_table_to_s3` for each. It provides more informative logging and error handling during batch uploads.


### `create_dumper_from_config(config_path: Optional[str] = None) -> DatabaseDumper`

Creates a `DatabaseDumper` instance directly from a configuration file, simplifying object creation.


### `dump_tables(...) -> List[str]`

Provides a programmatic interface for dumping tables to S3, allowing easy integration into other Python code. It handles configuration loading and object creation, making it convenient for programmatic use.


### `main(...)`

The main function, which is executed when the script is run from the command line. It uses the Typer library to handle command-line argument parsing and processes user input to configure and execute the database dump process. It includes comprehensive error handling and provides user-friendly output.



## Command-Line Interface (CLI)

The script can be run from the command line using the following command:

```bash
python your_script_name.py -t table1 table2 -b your_s3_bucket -p your_s3_prefix -c your_config_file.json -v
```

- `-t` or `--tables`:  A list of tables to dump (required).
- `-b` or `--bucket`: The S3 bucket name (optional, defaults to config).
- `-p` or `--prefix`: The S3 key prefix (optional, defaults to config).
- `-c` or `--config`: The path to the configuration file (optional, defaults to `environment-variables.json`).
- `-v` or `--verbose`: Enables verbose logging (optional).


## Error Handling

The script incorporates comprehensive error handling throughout, including explicit checks for missing configuration files, invalid JSON, database connection failures, and S3 upload issues.  Error messages are logged and communicated to the user through both logging and the command-line interface.


## Security Considerations

The script reads the PostgreSQL password from the environment variable `PGPASSWORD` instead of directly from the configuration file, enhancing security.  Consider using a more secure method of credential management for production environments.  Store the `environment-variables.json` file securely and ensure proper access controls for the S3 bucket.

