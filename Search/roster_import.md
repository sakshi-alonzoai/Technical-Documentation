# Documentation for `roster_import.py`

This document provides comprehensive technical documentation for the given Python script.

---

## Technical Documentation: PostgreSQL CSV Data Importer

### 1. High-Level Overview

This Python script is designed for efficient bulk data import into a PostgreSQL database table named `active_roster`. It connects to a PostgreSQL database using credentials loaded from environment variables (typically via a `.env` file), reads data from a specified CSV file (`compressed_data.csv`), and uses PostgreSQL's highly optimized `COPY` command to insert the data. The script includes robust error handling and ensures proper transaction management and resource cleanup.

### 2. Dependencies

The script relies on the following Python libraries:

*   **`psycopg2`**: A PostgreSQL adapter for Python, used to connect to the database and execute SQL commands.
*   **`python-decouple`**: A library for separating settings from code, enabling the loading of configuration variables from `.env` files or environment variables.

To install these dependencies, use pip:
```bash
pip install psycopg2-binary python-decouple
```

### 3. Configuration

The script loads sensitive PostgreSQL connection details from environment variables using `decouple.config()`. These variables are typically defined in a `.env` file located in the same directory as the script.

Example `.env` file:
```
PG_HOST=localhost
PG_PORT=5432
PG_DB=your_database_name
PG_USER=your_username
PG_PASSWORD=your_password
```

The following configuration variables are used:

*   `PG_HOST`: (String) The hostname or IP address of the PostgreSQL server. Loaded from the `PG_HOST` environment variable.
*   `PG_PORT`: (String, default: "5432") The port number on which the PostgreSQL server is listening. Loaded from the `PG_PORT` environment variable. Defaults to "5432" if not specified.
*   `PG_DB`: (String) The name of the PostgreSQL database to connect to. Loaded from the `PG_DB` environment variable.
*   `PG_USER`: (String) The username for connecting to the PostgreSQL database. Loaded from the `PG_USER` environment variable.
*   `PG_PASSWORD`: (String) The password for the PostgreSQL user. Loaded from the `PG_PASSWORD` environment variable.

### 4. Implementation Details and Execution Flow

The script's execution can be broken down into the following stages:

#### 4.1. Import Statements

```python
import psycopg2
# PostgreSQL connection details
from decouple import config
```
*   `import psycopg2`: Imports the `psycopg2` library, making its functions and classes available for database interaction.
*   `from decouple import config`: Imports the `config` function specifically from the `decouple` library, used for reading configuration variables.

#### 4.2. Configuration Loading

```python
# Load from .env
PG_HOST = config("PG_HOST")
PG_PORT = config("PG_PORT", default="5432")
PG_DB = config("PG_DB")
PG_USER = config("PG_USER")
PG_PASSWORD = config("PG_PASSWORD")
```
These lines load the PostgreSQL connection parameters from the environment.
*   `PG_HOST = config("PG_HOST")`: Retrieves the value associated with `PG_HOST` from environment variables or `.env`.
*   `PG_PORT = config("PG_PORT", default="5432")`: Retrieves `PG_PORT`. If not found, it defaults to "5432".
*   `PG_DB = config("PG_DB")`: Retrieves the database name.
*   `PG_USER = config("PG_USER")`: Retrieves the username.
*   `PG_PASSWORD = config("PG_PASSWORD")`: Retrieves the password.

#### 4.3. Database Connection Establishment

```python
# Create and export connection
conn = psycopg2.connect(
    host=PG_HOST,
    port=PG_PORT,
    dbname=PG_DB,
    user=PG_USER,
    password=PG_PASSWORD
)
```
*   `conn = psycopg2.connect(...)`: Establishes a connection to the PostgreSQL database using the loaded configuration variables. The returned `conn` object represents the database connection. This is the entry point for all database operations.

```python
cur = conn.cursor()
```
*   `cur = conn.cursor()`: Creates a new cursor object. A cursor is an object that allows you to execute SQL commands and retrieve results from the database. All SQL operations are performed through this cursor.

#### 4.4. Data Import Logic (`try...except...finally`)

This block encapsulates the core data import functionality, including file handling, database operations, error management, and resource cleanup.

```python
try:
    with open("compressed_data.csv", "r", encoding="utf-8") as f:
        next(f)  # Skip header line if the CSV has a header
        cur.copy_expert("""
            COPY active_roster (
                "_id",
                "gPersonId",
                "gTeamId",
                "playerId",
                "firstName",
                "familyName",
                "defaultShirtNumber",
                "statCrewShirtNumber",
                "playerClass",
                "defaultPlayingPosition",
                "playerName"
            )
            FROM STDIN WITH CSV
        """, f)
    conn.commit()
    print("✅ Data imported successfully.")
except Exception as e:
    conn.rollback()
    print("❌ Error during import:", e)
finally:
    cur.close()
    conn.close()
```

*   **`try` block**:
    *   `with open("compressed_data.csv", "r", encoding="utf-8") as f:`: Opens the `compressed_data.csv` file in read mode (`"r"`) with UTF-8 encoding. The `with` statement ensures the file is automatically closed even if errors occur. The file object is assigned to `f`.
    *   `next(f)`: Reads and discards the first line of the file. This is crucial if the CSV file contains a header row that should not be imported into the database.
    *   `cur.copy_expert(...)`: This is the most efficient way to import large amounts of data into PostgreSQL from a file-like object using `psycopg2`.
        *   **SQL Command (`COPY active_roster ... FROM STDIN WITH CSV`)**:
            *   `COPY active_roster`: Specifies the target table for the data import, which is `active_roster`.
            *   `("_id", "gPersonId", ..., "playerName")`: This is an explicit list of the target columns in the `active_roster` table into which the data from the CSV file will be inserted. The order of these columns **must** match the order of columns in the `compressed_data.csv` file.
            *   `FROM STDIN`: Indicates that the data source is standard input, which in this `copy_expert` context means the data will be provided by the Python script (specifically, the file object `f`).
            *   `WITH CSV`: Specifies that the input data is in CSV format, allowing PostgreSQL to correctly parse delimiters, quotes, and escape characters.
        *   `f`: The second argument to `copy_expert` is the file-like object (`f`) from which the `COPY` command will read its data. `psycopg2` handles streaming the contents of `f` to the PostgreSQL server.
    *   `conn.commit()`: If the `copy_expert` operation completes successfully within the `try` block, this line commits the transaction. This makes all changes (the imported data) permanent in the database.
    *   `print("✅ Data imported successfully.")`: Prints a success message to the console.

*   **`except Exception as e:` block**:
    *   This block catches any `Exception` that occurs during the execution of the `try` block. This ensures that errors during file reading or database operations do not crash the script.
    *   `conn.rollback()`: If an error occurs, this line rolls back the transaction. This means any changes attempted within the `try` block (e.g., partial data import) are discarded, ensuring the database remains in its consistent state before the script began.
    *   `print("❌ Error during import:", e)`: Prints an error message to the console, including the specific error details (`e`).

*   **`finally` block**:
    *   This block is guaranteed to execute whether an exception occurred or not. It's used for cleanup operations.
    *   `cur.close()`: Closes the database cursor. It's good practice to close cursors when they are no longer needed to release resources.
    *   `conn.close()`: Closes the database connection. This is essential to release the connection back to the connection pool or simply free up resources on both the client and server sides.

### 5. Usage

1.  **Create a `.env` file**: In the same directory as the Python script, create a file named `.env` and populate it with your PostgreSQL connection details as described in Section 3.
2.  **Prepare `compressed_data.csv`**: Ensure you have a CSV file named `compressed_data.csv` in the same directory. The columns in this CSV file *must* be in the exact order specified in the `COPY` statement.
    *   Example `compressed_data.csv` (without header, or with a header if `next(f)` is used):
        ```csv
        1,p123,t456,ABC1,John,Doe,10,10,Freshman,Forward,John Doe
        2,p124,t456,ABC2,Jane,Smith,12,12,Sophomore,Guard,Jane Smith
        ```
3.  **Run the script**: Execute the Python script from your terminal:
    ```bash
    python your_script_name.py
    ```

The script will then attempt to connect to the database, import the data, and report success or failure.
