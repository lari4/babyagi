# BabyAGI Prompts Documentation

This document contains all AI prompts used in the BabyAGI framework, grouped by their thematic purpose.

## Table of Contents

1. [Description Generation](#1-description-generation)
2. [Function Selection](#2-function-selection)
3. [Task Planning & Breakdown](#3-task-planning--breakdown)
4. [Code Generation](#4-code-generation)
5. [API Analysis](#5-api-analysis)
6. [Parameter Extraction](#6-parameter-extraction)
7. [Agent Systems](#7-agent-systems)
8. [Chat Systems](#8-chat-systems)

---

## 1. Description Generation

### 1.1 Function Description Writer

**Purpose:** Generates a concise and clear description for a Python function based on its code.

**Location:** `babyagi/functionz/packs/default/ai_functions.py:20-27`

**Used in function:** `description_writer(function_code: str) -> str`

**Prompt:**
```python
prompt = (
    f"Provide a concise and clear description for the following Python function:\n\n"
    f"{function_code}\n\n"
    f"Description:"
)
```

**Input Variables:**
- `function_code`: The complete source code of the Python function

**Expected Output:** A concise, clear description of what the function does

---

## 2. Function Selection

### 2.1 Choose Function from List

**Purpose:** Determines which functions from the available function list are most relevant to a given user input or task description.

**Location:** `babyagi/functionz/packs/default/ai_functions.py:254-262`

**Used in function:** `choose_function(prompt: str) -> str`

**Prompt:**
```python
prompt = (
    f"Which functions are most relevant to the following input? It could be ones to use or look at as reference to build a new one:\n\n"
    f"{prompt}\n\n"
    f"Functions:{functions}"
)
```

**Input Variables:**
- `prompt`: User's input or task description
- `functions`: List of all available functions

**Expected Output:** A response indicating which functions are relevant, either for direct use or as reference

---

### 2.2 Check Existing Functions for Task Match

**Purpose:** Analyzes available functions to determine if any existing function perfectly fulfills the user's request.

**Location:** `babyagi/functionz/packs/drafts/code_writing_functions.py:19-57`

**Used in function:** `check_existing_functions(user_input)`

**Prompt:**
```python
prompt = f"""
You are an expert software assistant. The user has provided the following request:

"{user_input}"

Below is a list of available functions with their descriptions:

{function_descriptions}

Determine if any of the existing functions perfectly fulfill the user's request. If so, return the name of the function.

Provide your answer in the following JSON format:
{{
  "function_found": true or false,
  "function_name": "<name of the function if found, else null>"
}}

Examples:

Example 1:
User input: "Calculate the sum of two numbers"
Functions: [{{"name": "add_numbers", "description": "Adds two numbers"}}]
Response:
{{
  "function_found": true,
  "function_name": "add_numbers"
}}

Example 2:
User input: "Translate text to French"
Functions: [{{"name": "add_numbers", "description": "Adds two numbers"}}]
Response:
{{
  "function_found": false,
  "function_name": null
}}

Now, analyze the user's request and provide the JSON response.
"""
```

**Input Variables:**
- `user_input`: The user's request or task description
- `function_descriptions`: List of dictionaries containing function names and descriptions

**Expected Output:** JSON object with `function_found` (boolean) and `function_name` (string or null)

---

### 2.3 Function Decision for Choose or Create

**Purpose:** Decides whether to use an existing function or generate a new one based on user input.

**Location:** `babyagi/functionz/packs/drafts/choose_or_create_function.py:46-73`

**Used in function:** `choose_or_create_function(user_input: str)`

**System Prompt:**
```python
system_prompt = """
You are an assistant that helps decide whether an existing function can fulfill a user's request or if a new function needs to be created.

Please analyze the user's input and the list of available functions.

Return your decision in the following JSON format:

{
    "use_existing_function": true or false,
    "function_name": "name of the existing function" (if applicable),
    "function_description": "description of the function to generate" (if applicable)
}

Provide only the JSON response, without any additional text.
"""
```

**Decision Prompt:**
```python
decision_prompt = f"""
The user has provided the following input:
\"{user_input}\"

Available Functions:
{existing_functions}
"""
```

**Input Variables:**
- `user_input`: The user's request
- `existing_functions`: List of available functions with descriptions

**Expected Output:** JSON object with `use_existing_function` (boolean), `function_name` (optional string), and `function_description` (optional string)

---

## 3. Task Planning & Breakdown

### 3.1 Break Down Task into Smaller Functions

**Purpose:** Decomposes a complex user task into smaller, reusable functions following a microservice-inspired architecture.

**Location:** `babyagi/functionz/packs/drafts/code_writing_functions.py:77-148`

**Used in function:** `break_down_task(user_input)`

**Prompt:**
```python
prompt = f"""
You are an expert software assistant helping to break down a user's request into smaller functions for a microservice-inspired architecture. The system is designed to be modular, with each function being small and designed optimally for potential future reuse.

When breaking down the task, consider the following:

- Each function should be as small as possible and do one thing well.
- Use existing functions where possible. You have access to functions such as 'gpt_call', 'find_similar_function', and others in our function database.
- Functions can depend on each other. Use 'dependencies' to specify which functions a function relies on.
- Functions should include appropriate 'imports' if external libraries are needed.
- Provide the breakdown as a list of functions, where each function includes its 'name', 'description', 'input_parameters', 'output_parameters', 'dependencies', and 'code' (just a placeholder or brief description at this stage).
- Make sure descriptions are detailed so an engineer could build it to spec.
- Every sub function you create should be designed to be reusable by turning things into parameters, vs hardcoding them.

User request:

"{user_input}"

Provide your answer in JSON format as a list of functions. Each function should have the following structure:

{{
  "name": "function_name",
  "description": "Brief description of the function",
  "input_parameters": [{{"name": "param1", "type": "type1"}}, ...],
  "output_parameters": [{{"name": "output", "type": "type"}}, ...],
  "dependencies": ["dependency1", "dependency2", ...],
  "imports": ["import1", "import2", ...],
  "code": "Placeholder or brief description"
}}

Example:

[
  {{
      "name": "process_data",
      "description": "Processes input data",
      "input_parameters": [{{"name": "data", "type": "str"}}],
      "output_parameters": [{{"name": "processed_data", "type": "str"}}],
      "dependencies": [],
      "imports": [],
      "code": "Placeholder for process_data function"
  }},
  {{
      "name": "analyze_data",
      "description": "Analyzes processed data",
      "input_parameters": [{{"name": "processed_data", "type": "str"}}],
      "output_parameters": [{{"name": "analysis_result", "type": "str"}}],
      "dependencies": ["process_data"],
      "imports": [],
      "code": "Placeholder for analyze_data function"
  }}
]

Now, provide the breakdown for the user's request.
"""
```

**Input Variables:**
- `user_input`: The user's complex task description

**Expected Output:** JSON array of function specifications with names, descriptions, parameters, dependencies, and imports

---

### 3.2 Decide Imports and External APIs

**Purpose:** Determines what Python imports and external APIs are needed for a set of functions.

**Location:** `babyagi/functionz/packs/drafts/code_writing_functions.py:150-218`

**Used in function:** `decide_imports_and_apis(context)`

**Prompt:**
```python
prompt = f"""
You are an expert software assistant helping to decide what imports and external APIs are needed for a set of functions based on the context provided.

Context:

{context}

Existing standard Python imports:

{list(existing_imports)}

Determine the libraries (imports) and external APIs needed for these functions. Separate standard Python libraries from external libraries or APIs.

Provide your answer in the following JSON format:

{{
  "standard_imports": ["import1", "import2", ...],
  "external_imports": ["external_import1", "external_import2", ...],
  "external_apis": ["api1", "api2", ...],
  "documentation_needed": [
      {{"name": "external_import1", "type": "import" or "api"}},
      ...
  ]
}}

Note: 'documentation_needed' should include any external imports or APIs for which documentation should be looked up.

Example:

{{
  "standard_imports": ["os", "json"],
  "external_imports": ["requests"],
  "external_apis": ["SerpAPI"],
  "documentation_needed": [
      {{"name": "requests", "type": "import"}},
      {{"name": "SerpAPI", "type": "api"}}
  ]
}}

Now, analyze the context and provide the JSON response.
"""
```

**Input Variables:**
- `context`: Information about the functions being created
- `existing_imports`: Set of imports already used in the system

**Expected Output:** JSON object with categorized imports and APIs, including documentation needs

---
