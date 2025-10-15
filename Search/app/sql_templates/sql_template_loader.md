# Documentation for `sql_template_loader.py`

This document provides comprehensive technical documentation for the `SQLTemplateLoader` Python script.

---

## Table of Contents

1.  [High-Level Overview](#1-high-level-overview)
2.  [Dependencies and Imports](#2-dependencies-and-imports)
3.  [Class: `SQLTemplateLoader`](#3-class-sqltemplateloader)
    *   [Class Docstring](#class-docstring)
    *   [Attributes](#attributes)
    *   [Method: `__init__`](#method-__init__)
    *   [Method: `load_template`](#method-load_template)
4.  [Main Execution Flow](#4-main-execution-flow)
5.  [Implementation Details and Considerations](#5-implementation-details-and-considerations)
    *   [Template Directory Structure](#template-directory-structure)
    *   [Eager vs. Lazy Loading](#eager-vs-lazy-loading)
    *   [Caching Mechanism](#caching-mechanism)
    *   [Error Handling](#error-handling)
    *   [Debug Output](#debug-output)

---

## 1. High-Level Overview

The `SQLTemplateLoader` script defines a utility class designed to manage SQL template files. Its primary purpose is to abstract away the file system operations required to load SQL query templates. It offers functionality to load templates from a specified directory, cache them in memory for faster retrieval, and provide them on demand to other parts of an application. The class supports both eager loading (all templates loaded at initialization) and lazy loading (templates loaded on first request if not already cached).

## 2. Dependencies and Imports

The script relies on the following standard Python modules and internal application components:

*   **`os`**:
    *   **Purpose**: Provides a way of using operating system dependent functionality. In this script, it's used for listing directory contents (`os.listdir`), checking file existence (`os.path.exists`), and constructing platform-independent file paths (`os.path.join`).
*   **`app.constants.Qualifiers`**:
    *   **Purpose**: This import suggests the script is part of a larger application where `Qualifiers` might define various constants or enumeration types. Although imported, `Qualifiers` is not explicitly used within the provided `SQLTemplateLoader` class. It's likely intended for future use or is a remnant from a template/boilerplate.

## 3. Class: `SQLTemplateLoader`

The `SQLTemplateLoader` class is the core component of this script, responsible for handling SQL template management.

### Class Docstring

```python
class SQLTemplateLoader:
    """
    A utility class that manages loading and caching of SQL template files.
    
    This class provides functionality to load SQL templates from a specified directory,
    cache them in memory, and retrieve them on demand. It supports lazy loading of
    templates that weren't loaded during initialization.
    
    Attributes:
        template_dir (str): Directory path containing SQL template files. Defaults to 'app/sql_templates'.
        templates (dict): Cache storing loaded SQL templates, mapping query_type to template content.
    """
```
**Explanation**:
The docstring clearly outlines the class's role: managing, loading, and caching SQL template files. It emphasizes its support for both initial eager loading and subsequent lazy loading. It also describes the two main instance attributes: `template_dir` (the path where templates are stored) and `templates` (the in-memory cache for the loaded templates).

### Attributes

*   **`template_dir` (str)**:
    *   **Description**: Stores the file system path to the directory where SQL template files are located.
    *   **Default Value**: `app/sql_templates`
    *   **Usage**: Used by `__init__` to load initial templates and by `load_template` to locate files for lazy loading.
*   **`templates` (dict)**:
    *   **Description**: An in-memory dictionary that acts as a cache for SQL templates.
    *   **Structure**: Keys are `query_type` (derived from the SQL filename without the `.sql` extension), and values are the string contents of the respective SQL template files.
    *   **Usage**: Populated during initialization (`__init__`) and by `load_template` upon first access of an uncached template. Retrieved by `load_template`.

### Method: `__init__`

```python
    def __init__(self, template_dir="app/sql_templates"):
        """
        Initialize the SQLTemplateLoader with a template directory.
        
        Args:
            template_dir (str): Path to directory containing SQL template files.
                              Defaults to 'app/sql_templates'.
                              
        On initialization, loads all SQL files from the template directory into memory.
        """
        self.template_dir = template_dir
        self.templates = {}
        
        # Eagerly load all SQL templates during initialization for faster subsequent access
        for filename in os.listdir(self.template_dir):
            if filename.endswith('.sql'):
                query_type = filename[:-4]  # Strip .sql extension to get template type
                template_path = os.path.join(self.template_dir, filename)
                with open(template_path, "r") as file:
                    self.templates[query_type] = file.read()
```
**Explanation**:
The constructor initializes a new instance of `SQLTemplateLoader`.

*   **`self, template_dir="app/sql_templates"`**:
    *   Defines the method. `self` refers to the instance of the class.
    *   `template_dir` is an argument specifying the path to the SQL templates. It has a default value of `'app/sql_templates'`, allowing for flexible instantiation.
*   **`self.template_dir = template_dir`**:
    *   Assigns the provided `template_dir` argument to the instance's `template_dir` attribute. This stores the path for later use.
*   **`self.templates = {}`**:
    *   Initializes an empty dictionary for the `templates` attribute. This dictionary will serve as the in-memory cache for SQL templates.
*   **`for filename in os.listdir(self.template_dir):`**:
    *   This loop iterates through all files and directories found within the `template_dir`. This is the start of the *eager loading* process.
*   **`if filename.endswith('.sql'):`**:
    *   Filters the listed items, processing only those that have a `.sql` file extension, indicating they are SQL template files.
*   **`query_type = filename[:-4]`**:
    *   Extracts the base name of the SQL file (e.g., `'get_user'` from `'get_user.sql'`). This `query_type` will serve as the key in the `self.templates` cache. The `[:-4]` slices the string to remove the last four characters (i.e., `.sql`).
*   **`template_path = os.path.join(self.template_dir, filename)`**:
    *   Constructs the full, absolute path to the SQL template file by joining the `template_dir` and the `filename` using `os.path.join`, which handles operating system-specific path separators.
*   **`with open(template_path, "r") as file:`**:
    *   Opens the SQL template file specified by `template_path` in read mode (`"r"`). The `with` statement ensures the file is properly closed even if errors occur.
*   **`self.templates[query_type] = file.read()`**:
    *   Reads the entire content of the SQL file and stores it as a string in the `self.templates` dictionary, using `query_type` as the key. This effectively caches the template content in memory.

### Method: `load_template`

```python
    def load_template(self, query_type):
        """
        Load and return an SQL template for the specified query type.
        
        This method first checks the in-memory cache for the template. If not found,
        attempts to load it from the filesystem. Templates are cached after loading
        to improve performance of subsequent requests.
        
        Args:
            query_type (str): The type of query template to load (filename without .sql extension)
            
        Returns:
            str: The contents of the SQL template file
            
        Raises:
            FileNotFoundError: If the template file doesn't exist in the template directory
        """
        if query_type not in self.templates:
            template_path = os.path.join(self.template_dir, f"{query_type}.sql")
            if not os.path.exists(template_path):
                raise FileNotFoundError(f"SQL template not found: {template_path}")
            
            with open(template_path, "r") as file:
                self.templates[query_type] = file.read()

        # Debug print statement to log template loading
        print(f"\n=== Loaded SQL Template for {query_type} ===\n{self.templates[query_type]}\n")

        return self.templates[query_type]
```
**Explanation**:
The `load_template` method provides the primary interface for retrieving SQL templates. It implements a cache-first strategy, falling back to file system loading if a template is not found in memory.

*   **`def load_template(self, query_type):`**:
    *   Defines the method. `self` refers to the instance.
    *   `query_type` is the string identifier for the desired SQL template (e.g., `'get_user'`).
*   **`if query_type not in self.templates:`**:
    *   This is the cache check. It verifies if the requested `query_type` already exists as a key in the `self.templates` dictionary (meaning it was either eagerly loaded or previously lazy-loaded). If the template is *not* in the cache, the code proceeds to load it from the file system (lazy loading).
*   **`template_path = os.path.join(self.template_dir, f"{query_type}.sql")`**:
    *   If the template is not in the cache, this line constructs the expected full file path for the template. It combines the `template_dir` with the `query_type` and appends the `.sql` extension.
*   **`if not os.path.exists(template_path):`**:
    *   Checks if the constructed `template_path` actually points to an existing file on the file system using `os.path.exists`.
*   **`raise FileNotFoundError(f"SQL template not found: {template_path}")`**:
    *   If the file does not exist, a `FileNotFoundError` is raised with a descriptive message, indicating that the requested template could not be found. This is critical for robust error handling.
*   **`with open(template_path, "r") as file:`**:
    *   If the file exists, it's opened in read mode (`"r"`).
*   **`self.templates[query_type] = file.read()`**:
    *   The content of the SQL file is read and stored in the `self.templates` cache, making it available for subsequent requests without needing to hit the file system again.
*   **`print(f"\n=== Loaded SQL Template for {query_type} ===\n{self.templates[query_type]}\n")`**:
    *   This is a debug print statement. It outputs a clear message to the console indicating which template was loaded and displays its full content. This is useful during development and debugging to confirm templates are being loaded correctly. This message will print regardless if the template was loaded eagerly or lazily, or from the cache.
*   **`return self.templates[query_type]`**:
    *   Finally, the method returns the content of the SQL template corresponding to the `query_type`, whether it was retrieved from the initial eager load, a lazy load, or directly from the cache.

## 4. Main Execution Flow

The provided script does not contain a `if __name__ == "__main__":` block. This indicates that the `SQLTemplateLoader` class is designed to be imported and used as a module by other parts of a larger application rather than being run directly as a standalone script.

To use this class, another script would typically:

1.  Import `SQLTemplateLoader`.
2.  Instantiate the class, optionally providing a custom `template_dir`.
3.  Call the `load_template` method with the desired `query_type` to retrieve SQL template strings.

**Example Usage (Conceptual)**:

```python
# In another_module.py

from my_app.template_loader import SQLTemplateLoader # Assuming path to this file

# Initialize the loader (templates in 'app/sql_templates' are loaded eagerly)
sql_loader = SQLTemplateLoader()

try:
    # Load a specific template
    user_query_template = sql_loader.load_template("get_user_by_id")
    print("User Query:", user_query_template)

    # Load another template (may be from cache or lazily loaded)
    insert_product_template = sql_loader.load_template("insert_product")
    print("Insert Product Query:", insert_product_template)

    # Attempt to load a non-existent template
    non_existent_template = sql_loader.load_template("update_order_status")

except FileNotFoundError as e:
    print(f"Error loading SQL template: {e}")

```

## 5. Implementation Details and Considerations

### Template Directory Structure

The script expects SQL template files to reside in a specific directory (defaulting to `app/sql_templates`). Each SQL file should represent a distinct query type, and its filename (without the `.sql` extension) will serve as the identifier (`query_type`) for retrieving that template.

**Example Directory Structure**:

```
app/
├── constants.py
├── sql_templates/
│   ├── get_user.sql
│   ├── insert_order.sql
│   └── update_product_stock.sql
└── template_loader.py (contains SQLTemplateLoader class)
```

In this structure:
*   `'get_user'` would be the `query_type` for `get_user.sql`.
*   `'insert_order'` would be the `query_type` for `insert_order.sql`.

### Eager vs. Lazy Loading

*   **Eager Loading**: During `SQLTemplateLoader` initialization (`__init__`), all `.sql` files found in `template_dir` are read into the `self.templates` cache. This ensures that frequently used templates are immediately available without any disk I/O on first access.
*   **Lazy Loading**: The `load_template` method implements lazy loading. If a requested `query_type` is not found in the cache (e.g., it's a new template added after initialization or the `template_dir` was changed), the method attempts to load it directly from the file system and then caches it for future use.

This combination optimizes for both startup performance (for essential templates) and memory efficiency (for less frequently used templates or dynamic template sets).

### Caching Mechanism

The `self.templates` dictionary acts as a simple in-memory cache. Once an SQL template is loaded (either eagerly or lazily), its content is stored in this dictionary. Subsequent requests for the same `query_type` will retrieve the template directly from the cache, avoiding redundant file system reads and improving performance. The cache persists for the lifetime of the `SQLTemplateLoader` instance.

### Error Handling

The `load_template` method explicitly checks for the existence of a requested template file on the file system. If `os.path.exists()` returns `False`, a `FileNotFoundError` is raised with a clear message indicating the missing template's expected path. This allows calling code to gracefully handle situations where templates are not found.

### Debug Output

The `load_template` method includes a `print` statement that logs the loaded template's `query_type` and its full content to standard output. While useful for debugging and development, this should typically be removed or replaced with a proper logging mechanism (e.g., Python's `logging` module) in production environments to avoid cluttering console output and to allow for configurable log levels.
