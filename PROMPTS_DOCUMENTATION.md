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
