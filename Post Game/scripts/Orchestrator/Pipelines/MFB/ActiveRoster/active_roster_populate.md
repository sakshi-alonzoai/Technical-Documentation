# Documentation for `active_roster_populate.py`

# Active Roster Population Script: Technical Documentation

## Overview

The Python script `active_roster_populate.py` is designed to maintain a temporary table (`temp_active_roster_mfb`) mirroring the `active_roster_mfb` table in a PostgreSQL database.  The script efficiently updates this temporary table by inserting data from `temp_persons_mfb` representing individuals (with non-null `person_id`) that are not already present in the `active_roster_mfb` table.  This ensures that `temp_active_roster_mfb` always contains the most up-to-date information, excluding duplicates.  The script is specifically tailored for the MFB (likely a sports league or organization) data model.


## Function Details

### 1. `parse_args()`

This function parses command-line arguments passed to the script. It expects arguments in the `KEY=VALUE` format.  The following keys are *required*:

*   `PG_HOST`: The hostname or IP address of the PostgreSQL server.
*   `PG_PORT`: The port number the PostgreSQL server is listening on.
*   `PG_DATABASE`: The name of the PostgreSQL database.
*   `PG_USERNAME`: The PostgreSQL username.
*   `PG_PASSWORD`: The PostgreSQL password.

The function validates that all required arguments are present.  If any are missing, it logs an error and exits with a status code of 1.  Malformed arguments (not in `KEY=VALUE` format) are ignored with a warning.  It returns a dictionary containing the parsed key-value pairs.

### 2. `main()`

The `main` function orchestrates the entire process.  It performs the following steps:

1.  **Parses Command-Line Arguments:** Calls `parse_args()` to retrieve database connection parameters.

2.  **Establishes Database Connection:** Uses the `psycopg2` library to connect to the PostgreSQL database using the parameters obtained from `parse_args()`.  Error handling is implemented to catch connection failures and exit gracefully with an error message.

3.  **Manages the Temporary Table (`temp_active_roster_mfb`):**
    *   Checks if the temporary table exists using a SQL query.
    *   If it exists, truncates the table using `TRUNCATE TABLE`.
    *   If it does not exist, creates the table as a copy of `active_roster_mfb` using `CREATE TABLE ... LIKE ... INCLUDING ALL`.

4.  **Inserts Missing Rows:** Executes an SQL `INSERT ... SELECT` statement to populate the temporary table. The `SELECT` statement cleverly performs the following:
    *   Retrieves distinct persons and their team information from `temp_persons_mfb`, filtering out entries with `person_id IS NULL`.
    *   Performs a `LEFT JOIN` with `active_roster_mfb` to identify persons not already present in the `active_roster_mfb` table.
    *   Generates a new `player_id` (UUID) for each new entry.
    *   Inserts only the rows where a match in `active_roster_mfb` was *not* found (`a.g_person_id IS NULL`).  This avoids duplicate entries.

5.  **Logs Results:** Logs the number of rows inserted and reports success or failure.

6.  **Cleans Up:** Closes the database connection in a `finally` block to ensure resources are released even if errors occur.

## Implementation Details and Dependencies

*   **Python 3:** This script requires Python 3.x.
*   **psycopg2:** The `psycopg2` library is used for interacting with the PostgreSQL database.  It must be installed (`pip install psycopg2-binary`).
*   **Logging:** The `logging` module is used for logging messages to the console.
*   **SQL Injection Prevention:** The script uses `psycopg2.sql` to prevent SQL injection vulnerabilities.  SQL queries are constructed using parameterized queries.
*   **UUID Generation:** The `gen_random_uuid()` function (not explicitly defined in the provided code, likely a database function) is used to generate universally unique identifiers for new `player_id` entries.
*   **Error Handling:**  The script incorporates comprehensive error handling to gracefully manage exceptions during database connection and query execution.

## Usage

The script is invoked from the command line with the following required environment variables:

```bash
python active_roster_populate.py PG_HOST=<host> PG_PORT=<port> PG_DATABASE=<db> PG_USERNAME=<user> PG_PASSWORD=<pass>
```

Replace `<host>`, `<port>`, `<db>`, `<user>`, and `<pass>` with the correct values for your PostgreSQL database configuration.

## Potential Improvements

*   **Configuration File:** Instead of relying on command-line arguments, consider using a configuration file (e.g., YAML or JSON) for better maintainability.
*   **More Robust Error Handling:** Implement more specific error handling based on different types of exceptions that might occur (e.g., database errors, connection errors).
*   **Transaction Management:** Wrap the database operations within a transaction to ensure atomicity (all operations succeed or none do).


This enhanced documentation provides a comprehensive guide to the functionality, implementation, and usage of the `active_roster_populate.py` script.  The added detail and structure improve clarity and usability for technical users.

