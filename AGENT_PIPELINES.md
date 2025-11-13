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
