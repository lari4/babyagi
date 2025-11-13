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

## 4. Code Generation

### 4.1 Generate Function Code from Specification

**Purpose:** Generates complete, working Python function code based on a function specification, using context from dependencies and similar functions.

**Location:** `babyagi/functionz/packs/drafts/code_writing_functions.py:233-338`

**Used in function:** `generate_function_code(function, context)`

**Prompt:**
```python
prompt = f"""
You are an expert Python programmer. Your task is to write detailed and working code for the following function based on the context provided. Do not provide placeholder code, but rather do your best like you are the best senior engineer in the world and provide the best code possible. DO NOT PROVIDE PLACEHOLDER CODE.

Function details:

Name: {function['name']}
Description: {function['description']}
Input parameters: {function['input_parameters']}
Output parameters: {function['output_parameters']}
Dependencies: {function['dependencies']}
Imports: {function['imports']}

Overall context:

{context}

Dependency code:

{dependency_code}

Code from functions with similar imports:

{similar_imports_functions_code}

Please provide the function details in JSON format, following this structure:

{{
  "function_name": "<function_name>",
  "metadata": {{
    "description": "<function_description>",
    "input_parameters": {function['input_parameters']},
    "output_parameters": {function['output_parameters']}
  }},
  "code": "<function_code_as_string>",
  "imports": {function['imports']},
  "dependencies": {function['dependencies']},
  "key_dependencies": [],
  "triggers": []
}}

**Example JSON Output:**

{{
  "function_name": "example_function",
  "metadata": {{
    "description": "An example function.",
    "input_parameters": [{{"name": "param1", "type": "str"}}],
    "output_parameters": [{{"name": "result", "type": "str"}}]
  }},
  "code": "<complete function code goes here>",
  "imports": ["os"],
  "dependencies": [],
  "key_dependencies": [],
  "triggers": []
}}

Provide the JSON output only, without any additional text. Do not provide placeholder code, but write complete code that is ready to run and provide the expected output.

Now, please provide the JSON output for the function '{function['name']}'.
"""
```

**Input Variables:**
- `function`: Dictionary with function specification (name, description, parameters, dependencies, imports)
- `context`: Overall context about the task
- `dependency_code`: Code of functions this function depends on
- `similar_imports_functions_code`: Code of other functions using similar imports

**Expected Output:** JSON object with complete function implementation ready to execute

---

### 4.2 Framework System Prompt (Used across generate_function.py)

**Purpose:** Provides consistent guidelines for code generation adhering to the functionz framework.

**Location:** `babyagi/functionz/packs/drafts/generate_function.py` (used in multiple functions)

**System Prompt:**
```python
system_prompt = """
You are an AI designed to help developers write Python functions using the functionz framework. Every function you generate must adhere to the following rules:

Function Registration: All functions must be registered with the functionz framework using the @babyagi.register_function() decorator. Each function can include metadata, dependencies, imports, and key dependencies.

Basic Function Registration Example:

def function_name(param1, param2):
    # function logic here
    return result

Metadata and Dependencies: When writing functions, you may include optional metadata (such as descriptions) and dependencies. Dependencies can be other functions or secrets (API keys, etc.).

Import Handling: Manage imports by specifying them in the decorator as dictionaries with 'name' and 'lib' keys. Include these imports within the function body.

Secret Management: When using API keys or authentication secrets, reference the stored key with globals()['key_name'].

Error Handling: Functions should handle errors gracefully, catching exceptions if necessary.

General Guidelines: Use simple, clean, and readable code. Follow the structure and syntax of the functionz framework. Ensure proper function documentation via metadata.
"""
```

**Usage:** This system prompt is used across multiple functions in `generate_function.py` to maintain consistency in code generation

---

### 4.3 Analyze Internal Functions for Reuse

**Purpose:** Analyzes existing functions to identify which can be reused directly and which should be used as references.

**Location:** `babyagi/functionz/packs/drafts/generate_function.py:33-136`

**Used in function:** `analyze_internal_functions(description, existing_functions, intermediate_steps)`

**Prompt:**
```python
display_prompt = f"""You are an assistant helping a developer build a function using the functionz framework.

The user has provided the following function description: {description}

The current available functions are listed below. Please specify if any of these functions can be used directly (for reuse), or if any should be referenced while building the new function. Return your response as structured JSON.

Available Functions:
{existing_functions}
"""
```

**Input Variables:**
- `description`: User's description of the function to generate
- `existing_functions`: List of all available functions in the system

**Expected Output:** JSON object with `reusable_functions` (list) and `reference_functions` (list)

---

### 4.4 Generate Final Function Code with Full Context

**Purpose:** Generates complete function code using all gathered information including reusable functions, reference functions, and API contexts.

**Location:** `babyagi/functionz/packs/drafts/generate_function.py:509-680`

**Used in function:** `generate_final_function_code(description, reusable_function_code, reference_function_code, api_contexts, intermediate_steps)`

**Prompt:**
```python
final_prompt = f"""{system_prompt}

The user wants to create a function with the following description: {description}.

You have the following internal reusable functions:
{json.dumps(reusable_function_code)}

You have the following internal reference functions:
{json.dumps(reference_function_code)}

You have the following context on the necessary external APIs and their usage:
{json.dumps(api_contexts)}

Generate a complete function using the functionz framework that adheres to the provided guidelines and utilizes the specified internal and external functions. Ensure the function is registered with the correct metadata, dependencies, and includes all relevant imports.

Provide the function details in a structured format including:
1. Function name
2. Complete function code (do not the @babyagi.register_function decorator)
3. Metadata (description)
4. Imports
5. Dependencies
6. Key dependencies
7. Triggers (if any)
"""
```

**Input Variables:**
- `description`: User's function description
- `reusable_function_code`: Code of functions that can be reused
- `reference_function_code`: Code of functions to use as reference
- `api_contexts`: Documentation and usage information for external APIs

**Expected Output:** JSON object with complete function implementation including all metadata

---

## 5. API Analysis

### 5.1 Determine Required External APIs

**Purpose:** Analyzes a function description to identify what external APIs (including SDKs or libraries) will be needed.

**Location:** `babyagi/functionz/packs/drafts/generate_function.py:176-269`

**Used in function:** `determine_required_external_apis(description, intermediate_steps)`

**Prompt:**
```python
prompt_for_apis = f"""You are an assistant analyzing function requirements.

The user has provided the following function description: {description}.

Identify if this function will require external APIs (including SDKs or libraries). If so, return a structured JSON with a list of external APIs, their purposes, and any relevant endpoints."""
```

**Input Variables:**
- `description`: User's description of the function to generate

**Expected Output:** JSON object with API details including name, purpose, and endpoints

---

### 5.2 Select Relevant URLs from Search Results

**Purpose:** Evaluates search results to identify the most relevant documentation URLs for an API.

**Location:** `babyagi/functionz/packs/drafts/generate_function.py:371-402`

**Used in function:** `handle_api_documentation(api_name, description, intermediate_steps)`

**Prompt:**
```python
link_selection_prompt = f"""You are given the following search results for the query "{search_query}":
{json.dumps(search_results)}

Which links seem most relevant for obtaining Python API documentation? Return them as a structured JSON list of URLs."""
```

**Input Variables:**
- `search_query`: The search query used (e.g., "OpenAI API documentation python")
- `search_results`: Results from SerpAPI search

**Expected Output:** JSON object with `selected_urls` list containing relevant documentation URLs

---

### 5.3 Extract Relevant API Information from Documentation

**Purpose:** Extracts relevant API methods, endpoints, and usage patterns from scraped documentation, determining if more information is needed.

**Location:** `babyagi/functionz/packs/drafts/generate_function.py:438-501`

**Used in function:** `handle_api_documentation(api_name, description, intermediate_steps)` (within scraping loop)

**Prompt:**
```python
extraction_prompt = f"""The user wants to create a function described as follows: {description}.
You have accumulated the following relevant API information so far:
{accumulated_info}

You have just scraped the following new API documentation:
{json.dumps(scrape_result)}

Based on the new information, extract any additional relevant API methods, endpoints, and usage patterns needed to implement the user's function. Indicate whether more information is required by setting 'requires_more_info' to true or false. If any other URLs should be scraped for further information, include them in the 'additional_urls' field."""
```

**Input Variables:**
- `description`: User's function description
- `accumulated_info`: Information extracted from previous documentation pages
- `scrape_result`: Content scraped from the current documentation page

**Expected Output:** JSON object with `relevant_info` (string), `additional_urls` (list), and `requires_more_info` (boolean)

**Note:** This prompt uses recursive exploration - the agent will continue scraping additional URLs until `requires_more_info` is false.

---

## 6. Parameter Extraction

### 6.1 Extract Function Parameters from User Input (code_writing_functions)

**Purpose:** Extracts the required parameters from user input to match a function's parameter signature.

**Location:** `babyagi/functionz/packs/drafts/code_writing_functions.py:415-508`

**Used in function:** `extract_function_parameters(user_input, function_name)`

**Prompt:**
```python
prompt = f"""
You are an expert assistant. The user wants to execute the following function:

Function code:
{function['code']}

Function description:
{function['metadata'].get('description', '')}

Function parameters:
{function['metadata'].get('input_parameters', [])}

The user has provided the following input:
"{user_input}"

Your task is to extract the required parameters from the user's input and provide them in JSON format that matches the function's parameters.

Provide your answer in the following JSON format:
{{
  "parameters": {{
    "param1": value1,
    "param2": value2,
    ...
  }}
}}

Ensure that the parameters match the function's required input parameters.

Examples:

Example 1:

Function code:
def add_numbers(a, b):
    return a + b

Function parameters:
[{{"name": "a", "type": "int"}}, {{"name": "b", "type": "int"}}]

User input: "Add 5 and 3"

Response:
{{
  "parameters": {{
    "a": 5,
    "b": 3
  }}
}}

Example 2:

Function code:
def greet_user(name):
    return f"Hello, {{name}}!"

Function parameters:
[{{"name": "name", "type": "str"}}]

User input: "Say hello to Alice"

Response:
{{
  "parameters": {{
    "name": "Alice"
  }}
}}

Now, using the function provided and the user's input, extract the parameters and provide the JSON response.
"""
```

**Input Variables:**
- `user_input`: The user's natural language input
- `function`: Dictionary containing function code, description, and parameter specifications

**Expected Output:** JSON object with a `parameters` key containing a dictionary of parameter names and values

---

### 6.2 Generate Function Parameters (choose_or_create_function)

**Purpose:** Generates appropriate parameter values for a function based on user input and function signature.

**Location:** `babyagi/functionz/packs/drafts/choose_or_create_function.py:143-189`

**Used in function:** `choose_or_create_function(user_input)` (Step 4)

**Prompt:**
```python
param_prompt = f"""
The user has provided the following input:
\"{user_input}\"

The function to execute is:
{function_info.get('code', '')}

Generate a JSON object with a single key "parameters" that contains the parameters required by the function, filled in appropriately based on the user's input.

Return only the JSON object, with no additional text.
"""
```

**System Prompt:**
```python
system_prompt = "You are an assistant that provides only JSON-formatted data, with no additional text."
```

**Input Variables:**
- `user_input`: The user's request
- `function_info`: Dictionary containing function code and metadata

**Expected Output:** JSON object with `parameters` key containing function arguments as key-value pairs

---

## 7. Agent Systems

### 7.1 ReAct Agent System Prompt

**Purpose:** Provides the system prompt for a reasoning and action agent that uses chain-of-thought techniques to solve tasks using available functions.

**Location:** `babyagi/functionz/packs/drafts/react_agent.py:72-81`

**Used in function:** `react_agent(input_text)`

**System Prompt:**
```python
system_prompt = (
    "You are an AI assistant that uses a chain-of-thought reasoning process to solve tasks. "
    "Let's think step by step to solve the following problem. "
    "You have access to the following functions which you can use to complete the task. "
    "Explain your reasoning in detail, including any functions you use and their outputs. "
    "At the end of your reasoning, provide the final answer after 'Answer:'. "
    "Before finalizing, review your reasoning for any errors or inconsistencies. "
    "Avoid repeating function calls with the same arguments you've already tried. "
    "Here is the history of function calls you have made so far: {{function_call_history}}"
)
```

**Dynamic Updates:**
- The `{{function_call_history}}` placeholder is replaced with a formatted string of previous function calls
- Format: `"- {function_name} with arguments {arguments} produced output: {output}"`
- Initially set to "None" if no calls have been made

**Input Variables:**
- `input_text`: The user's task or question
- `function_call_history`: List of dictionaries containing function_name, arguments, and output

**Key Features:**
- Chain-of-thought reasoning
- Iterative function calling (max 5 iterations)
- Function call deduplication (prevents calling the same function with same arguments twice)
- Self-review mechanism
- Access to all functions in the database via tool calling

**Expected Behavior:**
- The agent iteratively reasons about the problem
- Calls functions as needed via LiteLLM's tool calling interface
- Accumulates reasoning and function results
- Provides a final answer after "Answer:"

---

## 8. Chat Systems

### 8.1 Chat with Functions System Prompt

**Purpose:** Provides a simple system prompt for a chat assistant that can call functions during conversation.

**Location:** `babyagi/functionz/packs/default/function_calling_chat.py:42-44`

**Used in function:** `chat_with_functions(chat_history, available_function_names)`

**System Prompt:**
```python
chat_context = [
    {"role": "system", "content": "You are a helpful assistant."}
]
```

**Key Features:**
- Simple, straightforward assistant prompt
- Function calling via LiteLLM's tool interface
- Two-stage process: initial response, then function execution, then final response
- Supports maintaining chat history across multiple turns

**Input Variables:**
- `chat_history`: List of previous messages with role and content
- `available_function_names`: List or comma-separated string of function names to make available

**Workflow:**
1. Builds chat context from history
2. Fetches function definitions from database
3. Calls LLM with tools parameter
4. If functions are called, executes them
5. Calls LLM again with function results
6. Returns final assistant response

**Expected Output:** Natural language response from the assistant, potentially incorporating function call results

---

## Summary

This documentation covers **all AI prompts** used in the BabyAGI framework, organized into 8 thematic categories:

1. **Description Generation** - Automated function documentation
2. **Function Selection** - Choosing relevant functions for tasks
3. **Task Planning & Breakdown** - Decomposing complex tasks
4. **Code Generation** - Creating function implementations
5. **API Analysis** - Identifying and documenting external APIs
6. **Parameter Extraction** - Converting natural language to function parameters
7. **Agent Systems** - ReAct-style reasoning agents
8. **Chat Systems** - Conversational interfaces with function calling

### Key Patterns Across Prompts:

- **JSON-structured outputs** for reliable parsing
- **Few-shot examples** to guide LLM behavior
- **Iterative refinement** with retry loops on parse failures
- **Context accumulation** for multi-step processes
- **Function reuse emphasis** to leverage existing code
- **Modular design** following microservice principles

### Models Used:
- Primary: `gpt-4o-mini` (fast, cost-effective)
- Agent: `gpt-4-turbo` (for complex reasoning)
- Embeddings: `text-embedding-ada-002`

---

**Document Version:** 1.0
**Last Updated:** 2025
**Framework:** BabyAGI (functionz framework)
