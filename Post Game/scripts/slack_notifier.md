# Documentation for `slack_notifier.py`

# Slack Notification Utility: Technical Documentation

## Overview

This Python script provides a utility for posting messages to Slack using the Slack Chat Post Message API.  It prioritizes loading configuration from a `environment-variables.json` file located in the `config` directory relative to the script's location. If the file is not found, it falls back to environment variables.  The script utilizes asynchronous operations for efficient communication with the Slack API.

## Dependencies

The script requires the following Python packages:

* `httpx`: For making asynchronous HTTP requests to the Slack API.
* `json`: For handling JSON data in the configuration file.
* `pathlib`: For easier path manipulation.

These can be installed using pip:  `pip install httpx`

## Functions

### `load_slack_config()`

This function is responsible for loading Slack API token and default channel information.  It employs a caching mechanism to avoid redundant file reads within a single process execution.

**Functionality:**

1. **Cache Check:** First, it checks if the Slack credentials are already cached in the `_cached_slack_credentials` variable. If present, it returns the cached credentials.

2. **File Loading:** It attempts to load configuration from `config/environment-variables.json`.  The path is relative to the script's directory, assuming the script resides in a `scripts` subdirectory of the project root.  The JSON file should contain `"SLACK_BOT_TOKEN"` and `"DEFAULT_CHANNEL"` keys.

3. **Environment Variable Fallback:** If the JSON file is not found (`FileNotFoundError`), the function falls back to loading credentials from environment variables `SLACK_BOT_TOKEN` and `DEFAULT_CHANNEL`.

4. **Error Handling:** The function handles `FileNotFoundError`, `json.JSONDecodeError`, and cases where required configuration parameters are missing, raising appropriate exceptions (`EnvironmentError`, `ValueError`).

5. **Caching:** Once credentials are loaded (from either the file or environment variables), they are stored in `_cached_slack_credentials` for subsequent calls within the same process.


**Return Value:**

A tuple containing the `SLACK_BOT_TOKEN` and `DEFAULT_CHANNEL` strings.

**Exceptions:**

* `FileNotFoundError`: If `environment-variables.json` is not found.
* `json.JSONDecodeError`: If `environment-variables.json` contains invalid JSON.
* `EnvironmentError`: If either `SLACK_BOT_TOKEN` or `DEFAULT_CHANNEL` are missing in both the config file and the environment variables.
* `ValueError`: If the JSON in environment-variables.json is invalid.



### `post_to_slack(text, channel=None)`

This asynchronous function sends a message to a Slack channel using the Slack API.

**Parameters:**

* `text` (str): The message to be sent to Slack.  This parameter is mandatory.
* `channel` (str, optional): The Slack channel ID to send the message to. If not provided, it defaults to the `DEFAULT_CHANNEL` from the configuration.

**Functionality:**

1. **Credential Loading:** Calls `load_slack_config()` to obtain Slack credentials.  Handles potential `EnvironmentError` exceptions during credential loading.

2. **Input Validation:** Checks if the `text` parameter is provided; if not, it raises a `ValueError`.

3. **API Request:**  Constructs a payload containing the message text and target channel, sets the authorization header using the loaded `SLACK_BOT_TOKEN`, and makes an asynchronous POST request to the Slack Chat Post Message API (`https://slack.com/api/chat.postMessage`).

4. **Response Handling:** Parses the JSON response from the Slack API and checks the `"ok"` key. If `"ok"` is `False`, it raises a `RuntimeError` with details from the error response.

5. **Return Value:** If the request is successful, it returns the JSON response data from the Slack API.


**Exceptions:**

* `RuntimeError`: If there's an issue loading Slack credentials, or if the Slack API request fails.
* `ValueError`: If the `text` parameter is missing or empty.



## Usage

The `post_to_slack` function is designed to be used within an asynchronous context (e.g., within an `async` function or with `asyncio.run()`).  Ensure that the necessary configuration is set either in `environment-variables.json` or environment variables before running.  Example usage:


```python
import asyncio

async def main():
    try:
        result = await post_to_slack("This is a test message!", channel="#general")
        print(f"Message sent successfully: {result}")
    except Exception as e:
        print(f"Error sending message: {e}")


if __name__ == "__main__":
    asyncio.run(main())

```

