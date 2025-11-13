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
