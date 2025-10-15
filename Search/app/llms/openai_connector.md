# Documentation for `openai_connector.py`

This document provides a comprehensive technical breakdown of the `OpenAIConnector` Python script, detailing its purpose, functionality, and implementation specifics.

---

## 1. Overview

This Python script defines an `OpenAIConnector` class, designed to facilitate interaction with OpenAI's large language models (LLMs), specifically GPT-4o. It acts as a wrapper around the `openai` Python library, providing a standardized interface for generating responses from an OpenAI model. The connector inherits from a `BaseLLMConnector` (presumably an abstract base class for various LLM integrations) and handles API key management, model selection, and structured prompt delivery. The script also includes a `__main__` block for demonstrating the connector's usage with example prompts.

## 2. Dependencies and Imports

The script relies on several external libraries and internal modules:

*   **`import openai`**: The official Python client library for the OpenAI API. This is used to make API calls to OpenAI's language models.
*   **`import os`**: Python's standard library module for interacting with the operating system, primarily used here for accessing environment variables.
*   **`from dotenv import load_dotenv`**: A library that loads environment variables from a `.env` file into `os.environ`. This is crucial for securely managing sensitive information like API keys.
*   **`from app.llms.base import BaseLLMConnector`**: An internal module that defines a base class or interface for various LLM connectors. The `OpenAIConnector` inherits from this, implying a common structure for different LLM integrations within the `app` project.
*   **`from app.constants import LLMModels`**: An internal module likely containing an enumeration or similar structure for predefined LLM model names and potentially keys for retrieving API keys from environment variables.

## 3. Environment Variable Configuration

```python
load_dotenv()
```
This line calls the `load_dotenv()` function, which scans the current directory and its parents for a `.env` file. If found, it loads any key-value pairs defined within that file into the environment variables, making them accessible via `os.getenv()`. This is a best practice for managing configuration and sensitive data (like API keys) outside of the main codebase.

**Note:** A commented-out line `dotenv_path = os.path.join(os.path.dirname(__file__), "..", ".env")` suggests that at one point, there might have been an explicit path specification for the `.env` file, potentially to locate it outside the immediate script directory (e.g., in the project root). The current `load_dotenv()` without arguments defaults to finding `.env` in the current working directory or parent directories.

## 4. Class: `OpenAIConnector`

The `OpenAIConnector` class is the core component of this script, designed to encapsulate all logic related to communicating with the OpenAI API.

```python
class OpenAIConnector(BaseLLMConnector):
    """
    OpenAI GPT-4o Connector, inheriting from BaseLLMConnector.
    """
```
*   **Class Definition**: Defines `OpenAIConnector` as a new class.
*   **Inheritance**: It inherits from `BaseLLMConnector`, indicating that it implements the common interface defined by the base class. This promotes consistency across different LLM integrations.
*   **Docstring**: Provides a brief description of the class's purpose.

### 4.1. `__init__` Method (Constructor)

```python
    def __init__(self, model: str | None = LLMModels.GPT4.value):
        """
        Initializes the OpenAI GPT-4o connector.
        """
        self.model = model
        self.api_key = os.getenv(LLMModels.OPENAI_API_KEY.value)
        if not self.api_key:
            raise ValueError("API key is missing. Make sure it's set in the .env file or environment variables.")

        super().__init__(self.api_key)
        # self.client = openai.OpenAI(api_key=self.api_key)  # New API Client
        # self.client = openai.ChatCompletionz
```
This method initializes a new instance of the `OpenAIConnector`.

*   **Parameters**:
    *   `model` (`str | None`): The specific OpenAI model to use for generating responses. It defaults to `LLMModels.GPT4.value`, which is expected to resolve to a string like "gpt-4" or "gpt-4o" (though the class docstring mentions GPT-4o, `GPT4` constant might resolve to a slightly different model). The type hint `str | None` indicates it can be a string or `None`, but the default ensures it's usually a string.
*   **Implementation Logic**:
    1.  `self.model = model`: Stores the specified model name as an instance attribute.
    2.  `self.api_key = os.getenv(LLMModels.OPENAI_API_KEY.value)`: Retrieves the OpenAI API key from the environment variables. `LLMModels.OPENAI_API_KEY.value` is expected to be the name of the environment variable (e.g., "OPENAI_API_KEY").
    3.  `if not self.api_key: raise ValueError(...)`: Checks if the API key was successfully loaded. If not, it raises a `ValueError` with a helpful message, instructing the user to configure the `.env` file or environment variables.
    4.  `super().__init__(self.api_key)`: Calls the constructor of the parent class (`BaseLLMConnector`), passing the `api_key`. This suggests that `BaseLLMConnector` might handle common initialization tasks, such as storing the API key or performing base validations.
    5.  **Commented-out lines**:
        *   `# self.client = openai.OpenAI(api_key=self.api_key)`: This line, if uncommented, would initialize the newer OpenAI client object. The current `generate_response` method uses the older `openai.ChatCompletion.acreate` class method directly, not an instance of `openai.OpenAI`. This comment might indicate an intention to switch to the newer client API in the future.
        *   `# self.client = openai.ChatCompletionz`: This appears to be a typo or an incomplete thought and is commented out.

### 4.2. `async def generate_response` Method

```python
    async def generate_response(self, user_prompt: str,
                          system_prompt: str = "You are an AI assistant", 
                          temperature: float = 0.7):
        """
        Generates a response using OpenAI's GPT-4o.
        
        Args:
            system_prompt (str): Instructions to guide the assistant's behavior.
            user_prompt (str): The actual input query from the user.
            model (str): OpenAI model name (default: gpt-4o).  <- NOTE: This arg is not in method signature, it's from self.model
            temperature (float): Controls randomness (default: 0.7).

        Returns:
            str: The generated response from GPT-4o.
        """
        try:
            response = await openai.ChatCompletion.acreate(
                model=self.model,
                messages=[
                    {"role": "system", "content": system_prompt},
                    {"role": "user", "content": user_prompt}
                ],
                # max_tokens=max_tokens,
                temperature=temperature if self.model != LLMModels.GPT_O4_MINI.value else 1
            )
            return response.choices[0].message.content
        except openai.OpenAIError as e:
            raise RuntimeError(f"OpenAI API Error: {str(e)}")
```
This is an asynchronous method responsible for sending prompts to the OpenAI API and retrieving the generated text response.

*   **Asynchronous Nature**: The `async` keyword indicates this is a coroutine, meaning it must be `await`ed when called from another `async` function or run within an `asyncio` event loop.
*   **Parameters**:
    *   `user_prompt` (`str`): The primary input query provided by the user. This is mandatory.
    *   `system_prompt` (`str`): An optional instruction set that guides the AI's behavior and persona. Defaults to "You are an AI assistant".
    *   `temperature` (`float`): Controls the randomness of the generated response. Higher values (e.g., 1.0) make the output more creative and diverse, while lower values (e.g., 0.2) make it more focused and deterministic. Defaults to `0.7`.
*   **Docstring Anomaly**: The docstring lists `model (str)` as an argument, but `model` is actually an instance attribute (`self.model`) determined during initialization, not a parameter of this method. This should be corrected for accuracy.
*   **Return Value**:
    *   `str`: The text content of the AI's generated response.
*   **Implementation Logic**:
    1.  **`try...except` block**: Encapsulates the API call to handle potential errors gracefully.
    2.  `response = await openai.ChatCompletion.acreate(...)`: This is the core API call to OpenAI.
        *   `await`: Crucial for an `async` function to pause execution until the API call completes, without blocking the entire program.
        *   `openai.ChatCompletion.acreate`: This is the asynchronous class method used to create chat completions.
        *   `model=self.model`: Specifies the OpenAI model to use, obtained from the instance's `self.model` attribute.
        *   `messages`: A list of dictionaries representing the conversation history. It includes:
            *   `{"role": "system", "content": system_prompt}`: The system message sets the context or persona for the AI.
            *   `{"role": "user", "content": user_prompt}`: The user's query.
        *   `# max_tokens=max_tokens,`: This line is commented out, indicating that a `max_tokens` limit is not currently being enforced in the API call, or was an option considered.
        *   `temperature=temperature if self.model != LLMModels.GPT_O4_MINI.value else 1`: This line dynamically sets the temperature. If the selected model is *not* `GPT_O4_MINI` (which might be a specific token-constrained or cost-optimized model), it uses the provided `temperature` parameter. Otherwise, it hardcodes the `temperature` to `1`. This suggests `GPT_O4_MINI` might have different optimal or required temperature settings.
    3.  `return response.choices[0].message.content`: After a successful response, it extracts the actual text content from the first choice provided by the model. OpenAI typically returns a list of choices, but for most single-response scenarios, the first choice is sufficient.
    4.  `except openai.OpenAIError as e:`: Catches any exceptions specifically related to the OpenAI API.
    5.  `raise RuntimeError(f"OpenAI API Error: {str(e)}")`: Re-raises the OpenAI API error as a `RuntimeError` with a more generic prefix, allowing higher-level code to catch and handle API failures.

## 5. Main Execution Block (`if __name__ == "__main__":`)

```python
# === Test the OpenAIConnector when running this script directly ===
if __name__ == "__main__":
    openai_llm = OpenAIConnector()

    user_prompt = "Give me a nice romantic poem."

    # Test with custom system prompt
    response_custom = openai_llm.generate_response(user_prompt, system_prompt="Explain concepts in simple terms.")
    
    # Test with default system prompt
    response_default = openai_llm.generate_response(user_prompt)

    print("\n=== OpenAI GPT-4o Response (Custom System Prompt) ===")
    print(response_custom)

    print("\n=== OpenAI GPT-4o Response (Default System Prompt) ===")
    print(response_default)
```
This block demonstrates how to use the `OpenAIConnector` when the script is executed directly.

*   **`if __name__ == "__main__":`**: This standard Python construct ensures that the code inside this block only runs when the script is executed as the main program, not when imported as a module into another script.
*   `openai_llm = OpenAIConnector()`: An instance of the `OpenAIConnector` is created using its default model (e.g., `LLMModels.GPT4.value`).
*   `user_prompt = "Give me a nice romantic poem."`: A sample user query is defined.
*   **Test with custom system prompt**:
    *   `response_custom = openai_llm.generate_response(user_prompt, system_prompt="Explain concepts in simple terms.")`: Calls the `generate_response` method, providing the `user_prompt` and a specific `system_prompt` tailored for simple explanations.
*   **Test with default system prompt**:
    *   `response_default = openai_llm.generate_response(user_prompt)`: Calls `generate_response` again, but this time only passes the `user_prompt`. The `system_prompt` will default to "You are an AI assistant" as defined in the method signature.
*   **Printing Responses**: The script then prints descriptive headers and the received responses for both scenarios.

**Critical Note on Asynchronous Execution in `__main__`**:
The `generate_response` method is defined as `async`. However, in the `if __name__ == "__main__":` block, the calls `openai_llm.generate_response(...)` are made directly without `await`. In standard Python execution, an `async` function call that is not `await`ed will return a coroutine object immediately and will **not** execute the asynchronous logic (including the API call) or yield a meaningful result. To correctly run an `async` function from a synchronous context (like the `__main__` block), it needs to be executed within an `asyncio` event loop, typically using `asyncio.run()`.

For example, the test block should ideally look like this to properly await the `async` calls:

```python
import asyncio
# ... (rest of the code)

if __name__ == "__main__":
    async def main_test(): # Define an async main function
        openai_llm = OpenAIConnector()
        user_prompt = "Give me a nice romantic poem."

        # Test with custom system prompt
        response_custom = await openai_llm.generate_response(user_prompt, system_prompt="Explain concepts in simple terms.")
        
        # Test with default system prompt
        response_default = await openai_llm.generate_response(user_prompt)

        print("\n=== OpenAI GPT-4o Response (Custom System Prompt) ===")
        print(response_custom)

        print("\n=== OpenAI GPT-4o Response (Default System Prompt) ===")
        print(response_default)

    asyncio.run(main_test()) # Run the async main function
```
Without this `asyncio.run()` wrapper and `await` keywords, the `generate_response` calls in the provided `__main__` block would likely return coroutine objects instead of actual responses, and the print statements would show representations of these objects, not the generated poems.
