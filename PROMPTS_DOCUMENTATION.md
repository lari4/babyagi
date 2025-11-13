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
