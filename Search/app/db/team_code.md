# Documentation for `team_code.py`

This document provides a comprehensive technical overview and detailed explanation of the provided Python script.

---

## Technical Documentation: Database Synchronization Script

### 1. Overview

This Python script facilitates the synchronization of team codes from a MongoDB collection to a PostgreSQL table. Its primary purpose is to read a mapping of `teamName` to `teamCode` from a specified MongoDB collection, then use this mapping to update the `team_code` column in a PostgreSQL table (`tmn_temp`) where team names match. The script relies on environment variables for database credentials and configurations, ensuring sensitive information is kept separate from the codebase. It includes robust error handling and ensures database connections are properly closed.

### 2. Dependencies

The script requires the following Python libraries:

*   **`psycopg2`**: A PostgreSQL adapter for Python. Used for connecting to and interacting with PostgreSQL databases.
*   **`decouple`**: A library that helps to organize settings based on environment variables. Used here to load sensitive credentials from a `.env` file.
*   **`pymongo`**: The official MongoDB driver for Python. Used for connecting to and interacting with MongoDB databases.

These can typically be installed using pip:
```bash
pip install psycopg2-binary python-decouple pymongo
```

### 3. Configuration

The script uses `python-decouple` to load configuration settings and sensitive credentials from environment variables, typically stored in a `.env` file in the project's root directory.

#### 3.1 MongoDB Configuration Variables

*   **`DB_URI`**: (Required) The connection URI for the MongoDB instance (e.g., `mongodb://user:pass@host:port/`).
*   **`DB_NAME`**: (Required) The name of the MongoDB database to connect to.

#### 3.2 PostgreSQL Configuration Variables

*   **`PG_HOST`**: (Required) The hostname or IP address of the PostgreSQL server.
*   **`PG_PORT`**: (Optional, default: `5432`) The port number on which the PostgreSQL server is listening.
*   **`PG_DB`**: (Required) The name of the PostgreSQL database to connect to.
*   **`PG_USER`**: (Required) The username for authenticating with the PostgreSQL database.
*   **`PG_PASSWORD`**: (Required) The password for authenticating with the PostgreSQL database.

#### Example `.env` file:
```
DB_URI="mongodb://localhost:27017/"
DB_NAME="my_database"

PG_HOST="localhost"
PG_PORT="5432"
PG_DB="my_pg_db"
PG_USER="pg_user"
PG_PASSWORD="pg_password"
```

### 4. Script Logic and Execution Flow

The script executes sequentially within a `try...except...finally` block to handle potential errors and ensure resource cleanup.

#### 4.1 Initialization and MongoDB Data Retrieval

```python
try:
    # Load MongoDB credentials from .env
    MONGO_URI = config("DB_URI")
    DB_NAME = config("DB_NAME")
```
*   **`try:`**: Initiates a `try` block to catch potential exceptions during execution.
*   **`MONGO_URI = config("DB_URI")`**: Loads the MongoDB connection URI from the environment variable `DB_URI` using `decouple.config()`.
*   **`DB_NAME = config("DB_NAME")`**: Loads the MongoDB database name from the environment variable `DB_NAME`.

```python
    # Initialize MongoDB connection
    client = pymongo.MongoClient(MONGO_URI)
    db = client[DB_NAME]  # Access the database
    teams_collection = db["teams"]
```
*   **`client = pymongo.MongoClient(MONGO_URI)`**: Establishes a connection to the MongoDB server using the loaded `MONGO_URI`.
*   **`db = client[DB_NAME]`**: Accesses the specified MongoDB database using the connection client.
*   **`teams_collection = db["teams"]`**: Selects the "teams" collection within the connected MongoDB database. This is the source collection for team data.

```python
    # Build team_name → team_code map from Mongo
    mongo_map = {
        doc["teamName"].strip(): doc["teamCode"]
        for doc in teams_collection.find({}, {"teamName": 1, "teamCode": 1})
    }
    print(f"Loaded {len(mongo_map)} teams from MongoDB.")
```
*   **`mongo_map = { ... }`**: This line constructs a dictionary named `mongo_map`. This dictionary will store a mapping where keys are `teamName` values (stripped of whitespace) and values are their corresponding `teamCode`s.
*   **`for doc in teams_collection.find({}, {"teamName": 1, "teamCode": 1})`**: Iterates through all documents in the `teams_collection`. The second argument `{"teamName": 1, "teamCode": 1}` specifies that only the `_id`, `teamName`, and `teamCode` fields should be returned, optimizing data transfer.
*   **`doc["teamName"].strip(): doc["teamCode"]`**: For each document (`doc`), it takes the `teamName` field, strips any leading/trailing whitespace, and uses it as a key. The `teamCode` field is used as the corresponding value.
*   **`print(f"Loaded {len(mongo_map)} teams from MongoDB.")`**: Prints a confirmation message indicating how many team entries were successfully loaded into the `mongo_map` from MongoDB.

#### 4.2 PostgreSQL Connection

```python
    # Load Postgres credentials from .env
    PG_HOST = config("PG_HOST")
    PG_PORT = config("PG_PORT", default="5432")
    PG_DB = config("PG_DB")
    PG_USER = config("PG_USER")
    PG_PASSWORD = config("PG_PASSWORD")
```
*   **`PG_HOST = config("PG_HOST")`**: Loads the PostgreSQL host from the environment variable `PG_HOST`.
*   **`PG_PORT = config("PG_PORT", default="5432")`**: Loads the PostgreSQL port from `PG_PORT`. If `PG_PORT` is not found in the environment, it defaults to `5432`.
*   **`PG_DB = config("PG_DB")`**: Loads the PostgreSQL database name from `PG_DB`.
*   **`PG_USER = config("PG_USER")`**: Loads the PostgreSQL username from `PG_USER`.
*   **`PG_PASSWORD = config("PG_PASSWORD")`**: Loads the PostgreSQL password from `PG_PASSWORD`.

```python
    # Create and export connection
    pg_conn = psycopg2.connect(
        host=PG_HOST,
        port=PG_PORT,
        dbname=PG_DB,
        user=PG_USER,
        password=PG_PASSWORD
    )
    pg_cursor = pg_conn.cursor()
```
*   **`pg_conn = psycopg2.connect(...)`**: Establishes a connection to the PostgreSQL database using the credentials loaded from environment variables.
*   **`pg_cursor = pg_conn.cursor()`**: Creates a cursor object. A cursor allows you to execute SQL commands within the database session.

#### 4.3 Data Synchronization (Update Loop)

```python
    updated = 0
    for team_name, team_code in mongo_map.items():
        pg_cursor.execute(
            """
            UPDATE tmn_temp
            SET team_code = %s
            WHERE TRIM(LOWER(team_name)) = TRIM(LOWER(%s))
            """,
            (team_code, team_name)
        )
        if pg_cursor.rowcount > 0:
            print(f"Updated team_code for '{team_name}' to '{team_code}'")
            updated += 1
        else:
            print(f"No match found in tmn_temp for '{team_name}'")
```
*   **`updated = 0`**: Initializes a counter to keep track of the number of rows updated in PostgreSQL.
*   **`for team_name, team_code in mongo_map.items():`**: Iterates through each `(team_name, team_code)` pair in the `mongo_map` dictionary created earlier.
*   **`pg_cursor.execute(...)`**: Executes an SQL `UPDATE` statement for each team.
    *   **`UPDATE tmn_temp`**: Specifies the target table in PostgreSQL to be `tmn_temp`.
    *   **`SET team_code = %s`**: Sets the `team_code` column to the `team_code` value from the current `mongo_map` entry. The `%s` is a placeholder for `psycopg2` to safely insert the value.
    *   **`WHERE TRIM(LOWER(team_name)) = TRIM(LOWER(%s))`**: This is the matching condition. It compares the `team_name` column in `tmn_temp` with the `team_name` from `mongo_map`. Both sides of the comparison are transformed using `TRIM()` (to remove leading/trailing whitespace) and `LOWER()` (to convert to lowercase) to ensure a case-insensitive and whitespace-robust match. The second `%s` is a placeholder for the `team_name` from `mongo_map`.
    *   **`(team_code, team_name)`**: This tuple provides the values for the `%s` placeholders in the SQL query, in the order they appear.
*   **`if pg_cursor.rowcount > 0:`**: After executing the `UPDATE` query, `pg_cursor.rowcount` returns the number of rows affected by the last operation. If it's greater than 0, it means an update occurred.
    *   **`print(...)`**: Prints a success message indicating which team was updated.
    *   **`updated += 1`**: Increments the `updated` counter.
*   **`else:`**: If `pg_cursor.rowcount` is 0, no row in `tmn_temp` matched the current `team_name`.
    *   **`print(...)`**: Prints a message indicating that no match was found for the current team name.

```python
    pg_conn.commit()
    print(f"Update complete. {updated} rows updated.")
```
*   **`pg_conn.commit()`**: Commits the transaction to the PostgreSQL database. All `UPDATE` operations performed by `pg_cursor.execute()` are batched and only become permanent in the database after `commit()` is called.
*   **`print(f"Update complete. {updated} rows updated.")`**: Prints a final summary of the total number of rows updated in the PostgreSQL database.

#### 4.4 Error Handling and Resource Cleanup

```python
except Exception as e:
    print("An error occurred:", e)
finally:
    try:
        pg_cursor.close()
        pg_conn.close()
    except Exception:
        pass
```
*   **`except Exception as e:`**: If any `Exception` occurs within the `try` block (e.g., connection failure, invalid SQL, missing environment variable), this block catches it.
    *   **`print("An error occurred:", e)`**: Prints an error message along with the details of the exception.
*   **`finally:`**: This block is always executed, regardless of whether an exception occurred or not. It's crucial for ensuring that resources (like database connections) are properly closed.
    *   **`try:`**: A nested `try` block within `finally` is used to gracefully attempt to close the cursor and connection, even if they might not have been successfully opened (e.g., if the connection failed initially).
        *   **`pg_cursor.close()`**: Closes the PostgreSQL cursor.
        *   **`pg_conn.close()`**: Closes the PostgreSQL database connection.
    *   **`except Exception:`**: Catches any exception that might occur during the closing process (e.g., if `pg_cursor` or `pg_conn` were never successfully initialized).
        *   **`pass`**: Ignores any errors during the closing process, ensuring the script exits cleanly without further issues.

### 5. Important Implementation Details

*   **Environment Variables**: The script's reliance on `decouple` and `.env` files is a best practice for managing configuration and sensitive credentials, promoting security and portability.
*   **Robust Matching**: The use of `TRIM(LOWER())` in the `WHERE` clause of the SQL `UPDATE` statement ensures that team names are matched in a case-insensitive manner and are robust to leading or trailing whitespace, which often causes data mismatch issues.
*   **Transaction Management**: The explicit call to `pg_conn.commit()` is vital. Without it, the `UPDATE` operations would not be saved to the database. This also means that if an error occurs before `commit()`, none of the updates will be persisted, maintaining data integrity.
*   **Resource Management**: The `finally` block guarantees that database connections and cursors are always closed, preventing resource leaks and ensuring efficient operation. The nested `try...except` within `finally` provides extra robustness for scenarios where connections might not have been fully established.
*   **Batch Processing**: While the script iterates and updates one row at a time, the `commit()` operation is done once at the end, making it a single transaction for all updates. For very large datasets, optimizing to commit in smaller batches or using `executemany` might be considered, though for this script's design, a single commit is sufficient.
