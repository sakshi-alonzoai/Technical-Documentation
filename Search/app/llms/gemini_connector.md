# Documentation for `gemini_connector.py`

This document provides comprehensive technical documentation for the `GeminiConnector` Python script, detailing its purpose, structure, functions, and execution flow.

---

## Technical Documentation: `GeminiConnector`

### 1. High-Level Overview

This Python script defines a `GeminiConnector` class, which acts as an interface to Google's Gemini Generative AI models. It extends a `BaseLLMConnector` (presumably from `app.llms.base`) and provides a standardized way to initialize a Gemini model and generate textual responses based on user prompts and optional system instructions. The connector loads its API key from environment variables, ensuring secure handling of credentials. It also includes a `__main__` block for direct testing of its functionality.

### 2. Dependencies and Imports

The script relies on the following external libraries and internal modules:

*   **`google.generativeai as genai`**: This is the official Google Generative AI Python client library, used to interact with Gemini models.
*   **`os`**: Python's standard library module for interacting with the operating system, primarily used here to retrieve environment variables.
*   **`from dotenv import load_dotenv`**: This function from the `python-dotenv` library is used to load environment variables from a `.env` file into `os.environ`.
*   **`from app.llms.base import BaseLLMConnector`**: An internal module that presumably defines an abstract base class or a common interface for various Large Language Model (LLM) connectors. `GeminiConnector` inherits from this class.
*   **`from app.constants import LLMModels`**: An internal module that likely contains enumerated constants, such as model names (e.g., `GEMINI_FLASH`) and environment variable keys (e.g., `GEMINI_API_KEY`).

### 3. Environment Variable Loading

```python
# Load environment variables
load_dotenv()
```

*   **`load_dotenv()`**: This line calls the `load_dotenv()` function. Its purpose is to read key-value pairs from a `.env` file (if present in the current directory or parent directories) and add them to the system's environment variables. This is crucial for securely managing sensitive information like API keys, keeping them out of the source code.

### 4. Class: `GeminiConnector`

The `GeminiConnector` class is the core component of this script, responsible for managing the connection and interaction with Google's Gemini models.

```python
class GeminiConnector(BaseLLMConnector):
    """
    Gemini Connector, inheriting from BaseLLMConnector.
    """
```

*   **`class GeminiConnector(BaseLLMConnector):`**: Defines a new class `GeminiConnector` that inherits from `BaseLLMConnector`. This implies that `GeminiConnector` must implement any abstract methods defined in `BaseLLMConnector` and adheres to a common interface for LLM connectors.
*   **Docstring**: Provides a brief description of the class's purpose.

#### 4.1. `__init__` Method

The constructor for the `GeminiConnector` class, responsible for setting up the Gemini API client.

```python
    def __init__(self, model: str | None = LLMModels.GEMINI_FLASH.value):
        """
        Initializes the Gemini connector.
        """
        self.model = model
        print("Initializing GeminiConnector with model:", self.model)
        print("API API API",LLMModels.GEMINI_API_KEY.value)
        self.api_key = os.getenv(LLMModels.GEMINI_API_KEY.value)
        print("API new Key:", self.api_key)
        if not self.api_key:
            raise ValueError("API key is missing. Make sure it's set in the .env file or environment variables.")

        super().__init__(self.api_key)

        # Proper API configuration
        genai.configure(api_key=self.api_key)

        # Create the model
        self.model_client = genai.GenerativeModel(self.model)
```

*   **`def __init__(self, model: str | None = LLMModels.GEMINI_FLASH.value):`**:
    *   Initializes a new instance of `GeminiConnector`.
    *   **`model` (str | None)**: The name of the Gemini model to use (e.g., "gemini-pro", "gemini-flash").
    *   **Default Value**: `LLMModels.GEMINI_FLASH.value`. This indicates that `GEMINI_FLASH` is the default model if none is explicitly provided during instantiation. `LLMModels` is an enum-like object from `app.constants`.
*   **`self.model = model`**: Stores the chosen model name as an instance variable.
*   **`print(...)`**: Debugging statements to show which model is being initialized and what environment variable key is being sought for the API key.
*   **`self.api_key = os.getenv(LLMModels.GEMINI_API_KEY.value)`**:
    *   Retrieves the Gemini API key from the environment variables.
    *   `LLMModels.GEMINI_API_KEY.value` is expected to be the string key (e.g., "GEMINI_API_KEY") used to store the actual API key.
    *   `os.getenv()` safely retrieves the value, returning `None` if the variable is not set.
*   **`print("API new Key:", self.api_key)`**: Another debugging statement to print the API key (or `None` if not found). In a production environment, printing API keys directly is generally discouraged for security reasons.
*   **`if not self.api_key:`**:
    *   **Error Handling**: Checks if the API key was successfully retrieved.
    *   **`raise ValueError(...)`**: If the API key is missing, a `ValueError` is raised, instructing the user to set it in the `.env` file or environment variables. This ensures the connector cannot be used without proper authentication.
*   **`super().__init__(self.api_key)`**:
    *   Calls the constructor of the parent class (`BaseLLMConnector`).
    *   This usually handles common initialization tasks or sets common attributes, such as storing the API key or performing basic validation, that are shared across all LLM connectors.
*   **`genai.configure(api_key=self.api_key)`**:
    *   This is a crucial step for configuring the `google.generativeai` client.
    *   It globally sets the API key that the `genai` library will use for subsequent API calls.
*   **`self.model_client = genai.GenerativeModel(self.model)`**:
    *   Creates an instance of `genai.GenerativeModel`. This object represents the specific Gemini model (e.g., "gemini-flash") that will be used for generating content.
    *   The `model_client` object is then stored as an instance variable for later use by the `generate_response` method.

#### 4.2. `generate_response` Method

This asynchronous method is responsible for sending a prompt to the configured Gemini model and returning its response.

```python
    async def generate_response(self, user_prompt: str, system_prompt: str = "You are an AI assistant",
                          temperature: float = 0.7):
        """
        Generates a response using Gemini.

        Args:
            system_prompt (str): Instructions to guide the assistant's behavior.
            user_prompt (str): The actual input query from the user.
            temperature (float): Controls randomness (default: 0.7).

        Returns:
            str: The generated response from Gemini.
        """
        try:
            response = self.model_client.generate_content(
                contents=[{"role": "user", "parts": [user_prompt]}],
                generation_config={"temperature": temperature}
            )
            return response.text
        except Exception as e:
            raise RuntimeError(f"Gemini API Error: {str(e)}")
```

*   **`async def generate_response(...)`**:
    *   Declares an asynchronous method. This means it can be `await`-ed when called within an `async` function, allowing other tasks to run concurrently while waiting for the API response.
*   **Parameters**:
    *   **`user_prompt` (str)**: The primary input text from the user for which a response is to be generated. This is a mandatory parameter.
    *   **`system_prompt` (str)**: Instructions given to the AI to guide its behavior or persona.
        *   **Default Value**: `"You are an AI assistant"`.
        *   **Note**: For Gemini models using `generate_content`, the system prompt is typically integrated into the conversation history or handled implicitly rather than a separate `system_prompt` parameter as seen in some other LLMs (e.g., OpenAI). The current implementation passes only the `user_prompt` in `contents`. If `system_prompt` were to be actively used by Gemini via `generate_content`, it would typically need to be included as part of the `contents` list, perhaps as a "user" role followed by an "model" role, or if Gemini's API supports a dedicated system message, it would be passed differently. As written, the `system_prompt` parameter is defined but not directly utilized in the `generate_content` call's `contents` argument.
    *   **`temperature` (float)**: Controls the creativity or randomness of the generated response.
        *   **Default Value**: `0.7`. Higher values (e.g., 1.0) make the output more random; lower values (e.g., 0.2) make it more deterministic and focused.
*   **Docstring**: Clearly explains the method's purpose, arguments, and return value.
*   **`try...except` block**:
    *   **`response = self.model_client.generate_content(...)`**:
        *   Makes the actual API call to the Gemini model.
        *   **`contents=[{"role": "user", "parts": [user_prompt]}]`**: This is the main input for the model. It's structured as a list of dictionaries, where each dictionary represents a turn in a conversation. Here, it's a single turn from the "user" containing the `user_prompt`.
        *   **`generation_config={"temperature": temperature}`**: Configures parameters for the generation process, in this case, the `temperature`.
    *   **`return response.text`**: The `generate_content` method returns a response object. `response.text` extracts the generated text content from this object.
    *   **`except Exception as e:`**:
        *   Catches any exceptions that might occur during the API call (e.g., network issues, invalid API key, rate limits, model errors).
        *   **`raise RuntimeError(f"Gemini API Error: {str(e)}")`**: Reraises a `RuntimeError` with a more descriptive message, including the original error, making it easier to debug issues related to the Gemini API.

### 5. Main Execution Block (`if __name__ == "__main__":`)

This block demonstrates how to use the `GeminiConnector` when the script is run directly.

```python
# === Test the GeminiConnector when running this script directly ===
if __name__ == "__main__":
    gemini_llm = GeminiConnector()

    user_prompt = "Give me a nice romantic poem."

    response_custom = gemini_llm.generate_response(user_prompt, system_prompt="Explain concepts in simple terms.")
    response_default = gemini_llm.generate_response(user_prompt)

    print("\n=== Gemini Response (Custom System Prompt) ===")
    print(response_custom)

    print("\n=== Gemini Response (Default System Prompt) ===")
    print(response_default)
```

*   **`if __name__ == "__main__":`**: This standard Python construct ensures that the code inside this block only runs when the script is executed directly (not when imported as a module into another script).
*   **`gemini_llm = GeminiConnector()`**:
    *   Creates an instance of `GeminiConnector`.
    *   Since no `model` argument is provided, it will use the default model specified in the `__init__` method (e.g., `LLMModels.GEMINI_FLASH.value`).
*   **`user_prompt = "Give me a nice romantic poem."`**: Defines a sample `user_prompt` to send to the model.
*   **`response_custom = gemini_llm.generate_response(user_prompt, system_prompt="Explain concepts in simple terms.")`**:
    *   Calls the `generate_response` method with the `user_prompt` and a custom `system_prompt`.
    *   **Note**: As discussed in the `generate_response` section, the `system_prompt` parameter is currently not directly used in the `generate_content` call within this implementation for Gemini's API. The model's behavior might not be influenced by this specific `system_prompt` argument as it's passed here.
*   **`response_default = gemini_llm.generate_response(user_prompt)`**:
    *   Calls `generate_response` again, but this time without explicitly providing a `system_prompt`. It will use the default `system_prompt` defined in the method signature (`"You are an AI assistant"`).
*   **`print(...)`**: Prints headers and the generated responses to the console for both calls, allowing comparison of outputs (though the custom system prompt might not have an effect due to its current non-utilization in `generate_content`).

### 6. Configuration and Environment Variables

To run this script successfully, the following environment variable must be set:

*   **`GEMINI_API_KEY`**: This variable, whose name is presumably defined by `LLMModels.GEMINI_API_KEY.value`, must contain your Google Gemini API key.

You can set this in a `.env` file in the same directory as the script:

```dotenv
GEMINI_API_KEY="YOUR_GEMINI_API_KEY_HERE"
```

Replace `"YOUR_GEMINI_API_KEY_HERE"` with your actual API key obtained from Google AI Studio or Google Cloud.

### 7. Important Implementation Details

*   **Inheritance**: The class inherits from `BaseLLMConnector`, suggesting a broader framework for integrating various LLMs. This promotes code reusability and a consistent API across different models.
*   **API Key Management**: Utilizes `python-dotenv` and `os.getenv` for secure and flexible handling of API keys, preventing them from being hardcoded in the script.
*   **Asynchronous Operation**: The `generate_response` method is `async`, which is a best practice for I/O-bound operations like API calls. This allows the application to remain responsive while waiting for the LLM to process requests.
*   **Error Handling**: Basic error handling is in place for missing API keys during initialization and for API call failures during response generation, raising descriptive `ValueError` and `RuntimeError` exceptions, respectively.
*   **Gemini API Interaction**: The script correctly uses `genai.configure()` for global API key setup and `genai.GenerativeModel().generate_content()` for interacting with the model.
*   **System Prompt Handling**: As noted, the `system_prompt` parameter in `generate_response` is defined but not directly included in the `contents` list passed to `generate_content`. For Gemini, system instructions are often integrated differently (e.g., by structuring the conversation history or using specific model tunings). If the intent is for the `system_prompt` to actively guide the Gemini model's behavior, the `generate_content` call's `contents` argument might need adjustment (e.g., by adding an initial "system" or "user" role message that frames the AI's persona).
