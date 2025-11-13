# BabyAGI Agent Pipelines Documentation

This document describes all agent workflows, pipelines, and data flow patterns in the BabyAGI framework.

## Table of Contents

1. [process_user_input Pipeline](#1-process_user_input-pipeline)
2. [generate_function_from_description Pipeline](#2-generate_function_from_description-pipeline)
3. [choose_or_create_function Pipeline](#3-choose_or_create_function-pipeline)
4. [react_agent Pipeline](#4-react_agent-pipeline)
5. [chat_with_functions Pipeline](#5-chat_with_functions-pipeline)

---

## 1. process_user_input Pipeline

**Location:** `babyagi/functionz/packs/drafts/code_writing_functions.py:510-538`

**Purpose:** Main orchestration pipeline that processes user input, determines if existing functions can handle it, or generates new functions if needed, then executes the appropriate function.

### Pipeline Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                     process_user_input                          │
│                                                                 │
│  Input: user_input (string)                                    │
└──────────────────┬──────────────────────────────────────────────┘
                   │
                   ▼
┌──────────────────────────────────────────────────────────────────┐
│ STEP 1: Check Existing Functions                                │
│                                                                  │
│  Function: check_existing_functions(user_input)                 │
│  Prompt: "Check Existing Functions for Task Match"              │
│                                                                  │
│  Input:                                                          │
│    - user_input: User's request                                 │
│    - function_descriptions: All available functions             │
│                                                                  │
│  Output: {                                                       │
│    "function_found": boolean,                                   │
│    "function_name": string | null                               │
│  }                                                               │
└──────────────────┬───────────────────────────────────────────────┘
                   │
                   ▼
              ┌────┴────┐
              │ Decision │
              └────┬────┘
                   │
        ┌──────────┴──────────┐
        │                     │
        ▼ YES                 ▼ NO
┌───────────────────┐  ┌──────────────────────────────────────────┐
│ Use Existing      │  │ STEP 2: Break Down Task                 │
│ Function          │  │                                          │
│                   │  │ Function: break_down_task(user_input)   │
│ function_name =   │  │ Prompt: "Break Down Task into Smaller   │
│ result['name']    │  │          Functions"                      │
└─────────┬─────────┘  │                                          │
          │            │ Input: user_input                        │
          │            │                                          │
          │            │ Output: [                                │
          │            │   {                                      │
          │            │     "name": "function_name",             │
          │            │     "description": "...",                │
          │            │     "input_parameters": [...],           │
          │            │     "output_parameters": [...],          │
          │            │     "dependencies": [...],               │
          │            │     "imports": [...],                    │
          │            │     "code": "placeholder"                │
          │            │   },                                     │
          │            │   ...                                    │
          │            │ ]                                        │
          │            └──────────────────┬───────────────────────┘
          │                               │
          │                               ▼
          │            ┌──────────────────────────────────────────┐
          │            │ STEP 3: Generate Functions               │
          │            │                                          │
          │            │ Function: generate_functions(            │
          │            │    function_breakdown, context)          │
          │            │                                          │
          │            │ For each function in breakdown:          │
          │            │   - Check if similar function exists     │
          │            │   - If not, call create_function()       │
          │            │                                          │
          │            │ Sub-pipeline per function:               │
          │            │   1. decide_imports_and_apis()           │
          │            │   2. generate_function_code()            │
          │            │   3. add_new_function()                  │
          │            │                                          │
          │            │ Output: Functions created in database    │
          │            │                                          │
          │            │ function_name = first function in        │
          │            │                 breakdown                │
          │            └──────────────────┬───────────────────────┘
          │                               │
          └───────────────┬───────────────┘
                          │
                          ▼
┌──────────────────────────────────────────────────────────────────┐
│ STEP 4: Extract Parameters                                       │
│                                                                  │
│ Function: extract_function_parameters(user_input, function_name)│
│ Prompt: "Extract Function Parameters from User Input"           │
│                                                                  │
│ Input:                                                           │
│   - user_input: Original user request                           │
│   - function_name: Name of function to execute                  │
│   - function_code: Code of the function                         │
│   - function_metadata: Parameter specs                          │
│                                                                  │
│ Output: {                                                        │
│   "parameters": {                                               │
│     "param1": value1,                                           │
│     "param2": value2                                            │
│   }                                                              │
│ }                                                                │
└──────────────────┬───────────────────────────────────────────────┘
                   │
                   ▼
┌──────────────────────────────────────────────────────────────────┐
│ STEP 5: Execute Function                                         │
│                                                                  │
│ Function: run_final_function(function_name, **parameters)       │
│                                                                  │
│ Input:                                                           │
│   - function_name: Name of function to execute                  │
│   - **parameters: Extracted parameters                          │
│                                                                  │
│ Output: Result from executing the function                      │
└──────────────────┬───────────────────────────────────────────────┘
                   │
                   ▼
              ┌─────────┐
              │ Return  │
              │ Output  │
              └─────────┘
```

### Data Flow Summary

1. **Input:** User's natural language request
2. **Check Existing:** Determine if existing function matches
3. **Branch:**
   - If match found → Use existing function
   - If no match → Break down task → Generate new functions
4. **Extract Parameters:** Convert user input to function parameters
5. **Execute:** Run the function with parameters
6. **Output:** Return function result

### Key Functions Called

- `check_existing_functions()` - Function selection
- `break_down_task()` - Task decomposition
- `generate_functions()` - Function generation orchestrator
  - `find_similar_function()` - Similarity check
  - `create_function()` - Function creation
    - `decide_imports_and_apis()` - Dependency analysis
    - `generate_function_code()` - Code generation
    - `add_new_function()` - Database insertion
- `extract_function_parameters()` - Parameter extraction
- `run_final_function()` - Execution

---

## 2. generate_function_from_description Pipeline

**Location:** `babyagi/functionz/packs/drafts/generate_function.py:739-839`

**Purpose:** Advanced function generation pipeline that analyzes existing functions, determines required APIs, scrapes documentation, and generates complete working code.

### Pipeline Flow

```
┌──────────────────────────────────────────────────────────────────┐
│              generate_function_from_description                  │
│                                                                  │
│  Input: description (string)                                    │
│  Output: {                                                       │
│    "intermediate_steps": [...],                                 │
│    "final_code": string,                                        │
│    "added_to_database": boolean                                 │
│  }                                                               │
└──────────────────┬───────────────────────────────────────────────┘
                   │
                   ▼
┌──────────────────────────────────────────────────────────────────┐
│ STEP 1: Fetch Existing Functions                                │
│                                                                  │
│ Function: fetch_existing_functions(description)                 │
│                                                                  │
│ Actions:                                                         │
│   - Call display_functions_wrapper()                            │
│   - Get all functions from database                             │
│                                                                  │
│ Output: {                                                        │
│   "existing_functions": string (all functions),                 │
│   "intermediate_steps": [...]                                   │
│ }                                                                │
└──────────────────┬───────────────────────────────────────────────┘
                   │
                   ▼
┌──────────────────────────────────────────────────────────────────┐
│ STEP 2: Analyze Internal Functions                              │
│                                                                  │
│ Function: analyze_internal_functions(                           │
│     description, existing_functions, intermediate_steps)        │
│                                                                  │
│ Prompt: "Analyze Internal Functions for Reuse"                  │
│ Uses: Framework System Prompt                                   │
│                                                                  │
│ Input:                                                           │
│   - description: User's function description                    │
│   - existing_functions: All available functions                 │
│                                                                  │
│ Output: {                                                        │
│   "reusable_functions": [list of function names],              │
│   "reference_functions": [list of function names],             │
│   "intermediate_steps": [...]                                   │
│ }                                                                │
└──────────────────┬───────────────────────────────────────────────┘
                   │
                   ▼
┌──────────────────────────────────────────────────────────────────┐
│ STEP 3: Fetch Function Codes                                    │
│                                                                  │
│ Function: fetch_function_codes(function_names, intermediate_steps)│
│                                                                  │
│ Called twice:                                                    │
│   1. For reusable_functions                                     │
│   2. For reference_functions                                    │
│                                                                  │
│ For each function:                                               │
│   - Call get_function_wrapper(func_name)                        │
│   - Extract code                                                │
│                                                                  │
│ Output: {                                                        │
│   "function_codes": {                                           │
│     "func_name1": "code...",                                    │
│     "func_name2": "code...",                                    │
│     ...                                                          │
│   },                                                             │
│   "intermediate_steps": [...]                                   │
│ }                                                                │
└──────────────────┬───────────────────────────────────────────────┘
                   │
                   ▼
┌──────────────────────────────────────────────────────────────────┐
│ STEP 4: Determine Required External APIs                        │
│                                                                  │
│ Function: determine_required_external_apis(                     │
│     description, intermediate_steps)                            │
│                                                                  │
│ Prompt: "Determine Required External APIs"                      │
│ Uses: Framework System Prompt                                   │
│                                                                  │
│ Input: description                                               │
│                                                                  │
│ Output: {                                                        │
│   "external_apis": [                                            │
│     {                                                            │
│       "name": "API name",                                       │
│       "purpose": "Why it's needed",                             │
│       "endpoints": [                                            │
│         {                                                        │
│           "method": "GET/POST/etc",                             │
│           "url": "endpoint URL",                                │
│           "description": "..."                                  │
│         }                                                        │
│       ]                                                          │
│     }                                                            │
│   ],                                                             │
│   "intermediate_steps": [...]                                   │
│ }                                                                │
└──────────────────┬───────────────────────────────────────────────┘
                   │
                   ▼
┌──────────────────────────────────────────────────────────────────┐
│ STEP 5: Handle API Documentation (for each API)                 │
│                                                                  │
│ Function: handle_api_documentation(                             │
│     api_name, description, intermediate_steps)                  │
│                                                                  │
│ Sub-step 5.1: Search for Documentation                          │
│   - Call serpapi_search_v2(query)                               │
│   - Query: "{api_name} API documentation python"               │
│                                                                  │
│ Sub-step 5.2: Select Relevant URLs                              │
│   Prompt: "Select Relevant URLs from Search Results"            │
│   Output: selected_urls list                                    │
│                                                                  │
│ Sub-step 5.3: Recursive Documentation Scraping                  │
│   WHILE selected_urls AND requires_more_info:                   │
│     - Pop URL from selected_urls                                │
│     - Call scrape_website(url)                                  │
│     - Extract info with LLM:                                    │
│       Prompt: "Extract Relevant API Information"                │
│       Input: accumulated_info + new scrape_result               │
│       Output: {                                                  │
│         "relevant_info": string,                                │
│         "additional_urls": [urls],                              │
│         "requires_more_info": boolean                           │
│       }                                                          │
│     - Accumulate relevant_info                                  │
│     - Add additional_urls to selected_urls                      │
│     - Continue if requires_more_info == true                    │
│   END WHILE                                                      │
│                                                                  │
│ Output: {                                                        │
│   "api_contexts": [accumulated documentation strings],          │
│   "intermediate_steps": [...]                                   │
│ }                                                                │
└──────────────────┬───────────────────────────────────────────────┘
                   │
                   ▼
┌──────────────────────────────────────────────────────────────────┐
│ STEP 6: Generate Final Function Code                            │
│                                                                  │
│ Function: generate_final_function_code(                         │
│     description,                                                │
│     reusable_function_code,                                     │
│     reference_function_code,                                    │
│     api_contexts,                                               │
│     intermediate_steps)                                         │
│                                                                  │
│ Prompt: "Generate Final Function Code with Full Context"        │
│ Uses: Framework System Prompt                                   │
│                                                                  │
│ Input:                                                           │
│   - description: Original function description                  │
│   - reusable_function_code: {func_name: code}                  │
│   - reference_function_code: {func_name: code}                 │
│   - api_contexts: [documentation strings]                       │
│                                                                  │
│ Processing:                                                      │
│   - May chunk large prompts (100k chars, 10k overlap)          │
│   - Combines results from multiple chunks                       │
│   - Deduplicates imports, dependencies, etc.                    │
│                                                                  │
│ Output: {                                                        │
│   "combined_final": {                                           │
│     "name": "function_name",                                    │
│     "code": "complete function code",                           │
│     "metadata": {...},                                          │
│     "imports": [...],                                           │
│     "dependencies": [...],                                      │
│     "key_dependencies": [...],                                  │
│     "triggers": [...]                                           │
│   },                                                             │
│   "intermediate_steps": [...]                                   │
│ }                                                                │
└──────────────────┬───────────────────────────────────────────────┘
                   │
                   ▼
┌──────────────────────────────────────────────────────────────────┐
│ STEP 7: Add Function to Database                                │
│                                                                  │
│ Function: add_function_to_database(                             │
│     combined_final, intermediate_steps)                         │
│                                                                  │
│ Actions:                                                         │
│   - Call add_new_function() with all function details           │
│   - Register in database                                        │
│                                                                  │
│ Output: {                                                        │
│   "success": boolean,                                           │
│   "intermediate_steps": [...]                                   │
│ }                                                                │
└──────────────────┬───────────────────────────────────────────────┘
                   │
                   ▼
              ┌─────────┐
              │ Return  │
              │ Results │
              └─────────┘
```

### Data Flow Summary

1. **Input:** Function description (what the function should do)
2. **Analyze Context:**
   - Fetch all existing functions
   - Identify reusable and reference functions
   - Get their code
3. **API Discovery:**
   - Determine needed external APIs
   - Search for documentation
   - Recursively scrape and extract relevant info
4. **Code Generation:**
   - Combine all context (reusable functions, references, API docs)
   - Generate complete working code
   - Handle large contexts via chunking
5. **Database Registration:**
   - Add function to database
   - Make available for future use
6. **Output:** Complete function with metadata and success status

### Key Features

- **Recursive Documentation Exploration:** Continues scraping until enough info gathered
- **Context-aware Generation:** Uses similar functions as templates
- **Chunking Support:** Handles large contexts by breaking into chunks
- **Comprehensive Tracking:** All steps logged in intermediate_steps

### Key Functions Called

- `fetch_existing_functions()` - Database query
- `analyze_internal_functions()` - Function analysis with LLM
- `fetch_function_codes()` - Code retrieval
- `determine_required_external_apis()` - API identification with LLM
- `handle_api_documentation()` - Documentation scraping orchestrator
  - `serpapi_search_v2()` - Search API
  - `scrape_website()` - Web scraping
  - Multiple LLM calls for URL selection and info extraction
- `generate_final_function_code()` - Code generation with LLM
- `add_function_to_database()` - Database insertion

---

## 3. choose_or_create_function Pipeline

**Location:** `babyagi/functionz/packs/drafts/choose_or_create_function.py:18-207`

**Purpose:** Unified pipeline that decides whether to use an existing function or generate a new one, then executes it with generated parameters.

### Pipeline Flow

```
┌──────────────────────────────────────────────────────────────────┐
│                  choose_or_create_function                       │
│                                                                  │
│  Input: user_input (string)                                     │
│  Output: {                                                       │
│    "intermediate_steps": [...],                                 │
│    "execution_result": any                                      │
│  }                                                               │
└──────────────────┬───────────────────────────────────────────────┘
                   │
                   ▼
┌──────────────────────────────────────────────────────────────────┐
│ STEP 1: Fetch Existing Functions                                │
│                                                                  │
│ Function: display_functions_wrapper()                           │
│                                                                  │
│ Actions:                                                         │
│   - Get all functions from database                             │
│   - Log to intermediate_steps                                   │
│                                                                  │
│ Output: existing_functions (list)                               │
└──────────────────┬───────────────────────────────────────────────┘
                   │
                   ▼
┌──────────────────────────────────────────────────────────────────┐
│ STEP 2: Decide - Use Existing or Create New                     │
│                                                                  │
│ LLM Call with FunctionDecision Pydantic model                   │
│                                                                  │
│ System Prompt:                                                   │
│   "You are an assistant that helps decide whether an existing   │
│    function can fulfill a user's request or if a new function   │
│    needs to be created."                                        │
│                                                                  │
│ User Prompt:                                                     │
│   User input: {user_input}                                      │
│   Available Functions: {existing_functions}                     │
│                                                                  │
│ Output (JSON):                                                   │
│   {                                                              │
│     "use_existing_function": boolean,                           │
│     "function_name": string (if using existing),                │
│     "function_description": string (if creating new)            │
│   }                                                              │
└──────────────────┬───────────────────────────────────────────────┘
                   │
                   ▼
              ┌────┴────┐
              │ Branch  │
              └────┬────┘
                   │
        ┌──────────┴──────────┐
        │                     │
        ▼ use_existing=true   ▼ use_existing=false
┌───────────────────┐  ┌──────────────────────────────────────────┐
│ Use Existing      │  │ STEP 2b: Generate New Function          │
│                   │  │                                          │
│ function_name =   │  │ Call: generate_function_from_description(│
│ decision.name     │  │         function_description)            │
│                   │  │                                          │
└─────────┬─────────┘  │ This triggers the complete 7-step        │
          │            │ generate_function_from_description        │
          │            │ pipeline (see Pipeline #2)                │
          │            │                                          │
          │            │ Output:                                   │
          │            │   - Function created in database          │
          │            │   - intermediate_steps appended           │
          │            │                                          │
          │            │ Extract function_name from:               │
          │            │   - gen_result["function_name"], or       │
          │            │   - Parse from gen_result["final_code"]   │
          │            │     using regex: def\s+([a-zA-Z_]...)    │
          │            └──────────────────┬───────────────────────┘
          │                               │
          └───────────────┬───────────────┘
                          │
                          ▼
┌──────────────────────────────────────────────────────────────────┐
│ STEP 3: Get Function Info                                       │
│                                                                  │
│ Function: get_function_wrapper(function_name)                   │
│                                                                  │
│ Actions:                                                         │
│   - Retrieve function details from database                     │
│   - Get code, metadata, parameters                              │
│                                                                  │
│ Output: function_info dict                                      │
└──────────────────┬───────────────────────────────────────────────┘
                   │
                   ▼
┌──────────────────────────────────────────────────────────────────┐
│ STEP 4: Generate Function Parameters                            │
│                                                                  │
│ LLM Call with FunctionParameters Pydantic model                 │
│                                                                  │
│ System Prompt:                                                   │
│   "You are an assistant that provides only JSON-formatted data, │
│    with no additional text."                                    │
│                                                                  │
│ User Prompt:                                                     │
│   User input: {user_input}                                      │
│   Function code: {function_info['code']}                        │
│   Generate parameters as JSON with "parameters" key             │
│                                                                  │
│ Output (JSON):                                                   │
│   {                                                              │
│     "parameters": {                                             │
│       "param1": value1,                                         │
│       "param2": value2,                                         │
│       ...                                                        │
│     }                                                            │
│   }                                                              │
└──────────────────┬───────────────────────────────────────────────┘
                   │
                   ▼
┌──────────────────────────────────────────────────────────────────┐
│ STEP 5: Execute Function                                        │
│                                                                  │
│ Function: execute_function_wrapper(function_name, **parameters) │
│                                                                  │
│ Actions:                                                         │
│   - Execute function with generated parameters                  │
│   - Handle any execution errors                                 │
│   - Log result to intermediate_steps                            │
│                                                                  │
│ Output: execution_result (function return value)                │
└──────────────────┬───────────────────────────────────────────────┘
                   │
                   ▼
              ┌─────────┐
              │ Return  │
              │ Results │
              └─────────┘
```

### Data Flow Summary

1. **Input:** User's natural language request
2. **Fetch Context:** Get all available functions
3. **Decision:** LLM decides whether to use existing or create new
4. **Branch:**
   - **Use Existing:** Get function name from decision
   - **Create New:** Run full `generate_function_from_description` pipeline
5. **Get Function:** Retrieve function details from database
6. **Generate Parameters:** LLM converts user input to function parameters
7. **Execute:** Run function with parameters
8. **Output:** Execution result and intermediate steps

### Key Differences from process_user_input

| Feature | choose_or_create_function | process_user_input |
|---------|--------------------------|-------------------|
| **Decision Making** | Single LLM call | Structured check |
| **New Function Generation** | Calls `generate_function_from_description` | Calls `break_down_task` + `generate_functions` |
| **Code Generation Approach** | Advanced (API docs, scraping) | Simpler (context-based) |
| **Complexity** | Higher (7-step sub-pipeline) | Lower (inline generation) |
| **Best For** | Production use, complex APIs | Quick prototyping |

### Key Features

- **Unified Decision Point:** Single LLM call determines path
- **Reuses Advanced Pipeline:** Leverages `generate_function_from_description` for high-quality code
- **Name Extraction Fallback:** Uses regex if function name not in response
- **End-to-End Execution:** Goes from request to result

### Key Functions Called

- `display_functions_wrapper()` - Fetch all functions
- LLM with `FunctionDecision` model - Decision making
- `generate_function_from_description()` - New function generation (if needed)
- `get_function_wrapper()` - Function retrieval
- LLM with `FunctionParameters` model - Parameter generation
- `execute_function_wrapper()` - Function execution

---

## 4. react_agent Pipeline

**Location:** `babyagi/functionz/packs/drafts/react_agent.py:11-193`

**Purpose:** Iterative reasoning and action agent that uses chain-of-thought to solve tasks by planning, executing functions, and reflecting on results.

### Pipeline Flow

```
┌──────────────────────────────────────────────────────────────────┐
│                         react_agent                              │
│                                                                  │
│  Input: input_text (string) - task or question                  │
│  Output: full_response (string) - reasoning + answer            │
└──────────────────┬───────────────────────────────────────────────┘
                   │
                   ▼
┌──────────────────────────────────────────────────────────────────┐
│ INITIALIZATION                                                   │
│                                                                  │
│ 1. Get all available functions:                                 │
│    - Call get_all_functions_wrapper()                           │
│    - Extract function names                                     │
│                                                                  │
│ 2. Build tools array:                                           │
│    For each function:                                            │
│      - Call get_function_wrapper(func_name)                     │
│      - Map Python types to JSON schema                          │
│      - Create tool definition for LiteLLM                       │
│                                                                  │
│ 3. Initialize state:                                            │
│    - function_call_history = []                                 │
│    - iteration = 0                                              │
│    - max_iterations = 5                                         │
│    - full_reasoning_path = ""                                   │
│                                                                  │
│ 4. Set up chat context:                                         │
│    [                                                             │
│      {                                                           │
│        "role": "system",                                        │
│        "content": Chain-of-Thought System Prompt                │
│                   (includes function_call_history)              │
│      },                                                          │
│      {                                                           │
│        "role": "user",                                          │
│        "content": input_text                                    │
│      }                                                           │
│    ]                                                             │
└──────────────────┬───────────────────────────────────────────────┘
                   │
                   ▼
┌──────────────────────────────────────────────────────────────────┐
│ ITERATIVE LOOP (max 5 iterations)                               │
│                                                                  │
│ WHILE iteration < max_iterations:                               │
│                                                                  │
│   ┌──────────────────────────────────────────────────────────┐ │
│   │ STEP 1: Update System Prompt with History               │ │
│   │                                                          │ │
│   │ If function_call_history not empty:                     │ │
│   │   Format history as:                                    │ │
│   │   "- {func_name} with args {args} → {output}"          │ │
│   │ Else:                                                    │ │
│   │   "None"                                                 │ │
│   │                                                          │ │
│   │ Replace {{function_call_history}} placeholder           │ │
│   └──────────────────┬───────────────────────────────────────┘ │
│                      │                                          │
│                      ▼                                          │
│   ┌──────────────────────────────────────────────────────────┐ │
│   │ STEP 2: Call LLM with Tools                             │ │
│   │                                                          │ │
│   │ litellm.completion(                                     │ │
│   │   model="gpt-4-turbo",                                  │ │
│   │   messages=chat_context,                                │ │
│   │   tools=tools,                                          │ │
│   │   tool_choice="auto",                                   │ │
│   │   max_tokens=1500,                                      │ │
│   │   temperature=0.7                                       │ │
│   │ )                                                        │ │
│   │                                                          │ │
│   │ Output: response_message                                │ │
│   └──────────────────┬───────────────────────────────────────┘ │
│                      │                                          │
│                      ▼                                          │
│   ┌──────────────────────────────────────────────────────────┐ │
│   │ STEP 3: Process Response                                │ │
│   │                                                          │ │
│   │ - Append response_message to chat_context               │ │
│   │ - Append to full_reasoning_path                         │ │
│   │ - Extract tool_calls from response                      │ │
│   └──────────────────┬───────────────────────────────────────┘ │
│                      │                                          │
│                      ▼                                          │
│              ┌───────┴────────┐                                 │
│              │ tool_calls?    │                                 │
│              └───────┬────────┘                                 │
│                      │                                          │
│        ┌─────────────┴─────────────┐                           │
│        │ YES                       │ NO                        │
│        ▼                           ▼                           │
│   ┌────────────────────┐  ┌────────────────────────┐          │
│   │ For each tool_call:│  │ No functions to call   │          │
│   │                    │  │ Task complete          │          │
│   │ STEP 4a: Check     │  │ BREAK LOOP             │          │
│   │ Deduplication      │  └────────────────────────┘          │
│   │                    │                                        │
│   │ Is (func_name,     │                                        │
│   │     args) already  │                                        │
│   │     in history?    │                                        │
│   │                    │                                        │
│   │ ┌─YES────┬───NO─┐ │                                        │
│   │ ▼        ▼       │ │                                        │
│   │ Return   Execute │ │                                        │
│   │ "already func via│ │                                        │
│   │ called"  wrapper │ │                                        │
│   │          ↓       │ │                                        │
│   │       Add to     │ │                                        │
│   │       history    │ │                                        │
│   │          ↓       │ │                                        │
│   │  Build response  │ │                                        │
│   │  message         │ │                                        │
│   └────────┬─────────┘ │                                        │
│            │           │                                        │
│            ▼           │                                        │
│   ┌────────────────────┴───────────────────────┐              │
│   │ STEP 4b: Append Tool Response to Context   │              │
│   │                                             │              │
│   │ chat_context.append({                      │              │
│   │   "tool_call_id": id,                      │              │
│   │   "role": "tool",                          │              │
│   │   "name": func_name,                       │              │
│   │   "content": response                      │              │
│   │ })                                          │              │
│   │                                             │              │
│   │ Update full_reasoning_path                 │              │
│   └─────────────────────┬───────────────────────┘              │
│                         │                                      │
│                         ▼                                      │
│                   ┌──────────┐                                 │
│                   │ CONTINUE │                                 │
│                   │ LOOP     │                                 │
│                   └──────────┘                                 │
│                                                                 │
│   iteration += 1                                               │
│                                                                 │
│ END WHILE                                                       │
└──────────────────┬───────────────────────────────────────────────┘
                   │
                   ▼
┌──────────────────────────────────────────────────────────────────┐
│ FINALIZATION                                                     │
│                                                                  │
│ 1. Extract final answer:                                        │
│    If "Answer:" in response_message.content:                    │
│      final_answer = text after "Answer:"                        │
│    Else:                                                         │
│      final_answer = entire response_message.content             │
│                                                                  │
│ 2. Format function calls history:                               │
│    For each call in function_call_history:                      │
│      "Function '{name}' called with {args}, → {output}"         │
│                                                                  │
│ 3. Build full response:                                         │
│    "Full Reasoning Path:\n{full_reasoning_path}\n\n"            │
│    "Functions Used:\n{function_calls_str}\n\n"                  │
│    "Final Answer:\n{final_answer}"                              │
└──────────────────┬───────────────────────────────────────────────┘
                   │
                   ▼
              ┌─────────┐
              │ Return  │
              │ Response│
              └─────────┘
```

### Data Flow Summary

1. **Input:** Task or question in natural language
2. **Setup:**
   - Fetch all available functions from database
   - Build tool definitions for LLM
   - Initialize tracking variables
3. **Iterative Reasoning Loop (up to 5 iterations):**
   - **Think:** LLM reasons about the problem
   - **Act:** LLM decides to call functions or provide answer
   - **Observe:** Function results fed back to LLM
   - **Reflect:** LLM incorporates results into reasoning
4. **Deduplication:** Prevents calling same function with same args twice
5. **Termination:** Loop ends when LLM provides answer or max iterations reached
6. **Output:** Complete reasoning path + function calls + final answer

### Key Features

- **Chain-of-Thought Reasoning:** LLM explicitly explains its thinking
- **Iterative Planning:** Can call multiple functions across iterations
- **Function Call History:** Maintains and shares history with LLM
- **Deduplication:** Prevents redundant function calls
- **Self-Correction:** Can reflect on errors and try different approaches
- **Full Transparency:** Returns complete reasoning path

### Loop Termination Conditions

1. **LLM decides task is complete** (no tool_calls in response)
2. **Max iterations reached** (5 iterations)
3. **Error occurs** (caught and returned in response)

### Example Flow

```
User: "What's 5 factorial divided by 2?"

Iteration 1:
  Think: "I need to calculate 5 factorial first"
  Act: calculate_factorial(5)
  Observe: 120

Iteration 2:
  Think: "Now I need to divide 120 by 2"
  Act: divide(120, 2)
  Observe: 60

Iteration 3:
  Think: "I have the answer"
  Act: None
  Answer: "5 factorial divided by 2 equals 60"
  → Loop terminates
```

### Key Functions Called

- `get_all_functions_wrapper()` - Fetch available functions
- `get_function_wrapper()` - Get function details for tool building
- `execute_function_wrapper()` - Execute selected functions
- `litellm.completion()` - LLM calls with tool support

### Comparison with Other Pipelines

| Feature | react_agent | process_user_input | choose_or_create |
|---------|------------|-------------------|-----------------|
| **Approach** | Iterative reasoning | One-shot execution | Decision then execute |
| **Function Calls** | Multiple, sequential | Single | Single |
| **Reasoning** | Explicit CoT | Implicit | Implicit |
| **Adaptability** | High (can adjust plan) | Low | Medium |
| **Best For** | Complex multi-step tasks | Simple tasks | Production automation |

---
