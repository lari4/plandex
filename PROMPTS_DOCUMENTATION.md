# Plandex AI Prompts Documentation

This document provides a comprehensive overview of all AI prompts used in the Plandex application. Each prompt is categorized by its purpose and includes a detailed description of its functionality.

---

## Table of Contents

1. [System Identity](#1-system-identity)
2. [Planning Phase Prompts](#2-planning-phase-prompts)
3. [Implementation Phase Prompts](#3-implementation-phase-prompts)
4. [Context Management Prompts](#4-context-management-prompts)
5. [Chat Mode Prompts](#5-chat-mode-prompts)
6. [Code Formatting Prompts](#6-code-formatting-prompts)
7. [File Operations Prompts](#7-file-operations-prompts)
8. [Execution & Script Prompts](#8-execution--script-prompts)
9. [Naming Prompts](#9-naming-prompts)
10. [Summary & Description Prompts](#10-summary--description-prompts)
11. [Validation Prompts](#11-validation-prompts)
12. [Execution Status Prompts](#12-execution-status-prompts)
13. [Missing File Prompts](#13-missing-file-prompts)

---

## 1. System Identity

### Identity

**Purpose:** Establishes the core identity of the Plandex AI assistant.

**Location:** `app/server/model/prompts/planning.go`

**Description:** This is the fundamental identity statement that defines Plandex as an AI programming and system administration assistant. It is used as a prefix in planning prompts to establish the AI's role.

```
You are Plandex, an AI programming and system administration assistant.
```

---

## 2. Planning Phase Prompts

### GetPlanningPrompt

**Purpose:** Main prompt for the planning phase where the AI breaks down user tasks into subtasks.

**Location:** `app/server/model/prompts/planning.go`

**Function Signature:** `GetPlanningPrompt(params CreatePromptParams) string`

**Description:** This is the core planning prompt that instructs the AI on how to:
- Determine if the user has a task or is just chatting
- Create a plan with numbered subtasks
- Use the '### Tasks' section format
- Include file dependencies with 'Uses:' lists
- Output `<PlandexFinish/>` tag after tasks
- Handle auto-context mode vs manual context
- Handle execution mode (can run commands via _apply.sh)
- Consider git repository status
- Break down complex tasks into manageable subtasks

**Key Features:**
- Adapts based on auto-context mode (loads context automatically)
- Supports execution mode for running commands
- Handles git repositories differently (creates .gitignore vs .plandexignore)
- Enforces strict formatting for task lists
- Prevents code implementation during planning (planning only)

**Parameters:**
- `AutoContext` - Whether auto-context mode is enabled
- `ExecMode` - Whether execution mode is enabled
- `IsUserDebug` - Debug mode for user commands
- `IsApplyDebug` - Debug mode for _apply.sh script
- `IsGitRepo` - Whether the project is a git repository
- `ContextTokenLimit` - Maximum tokens for context

**Example Output Format:**
```
### Tasks

1. Create a new file called 'game_logic.h'
   - This file will be used to define the 'updateGameLogic' function
   - This file will be created in the 'src' directory
   Uses: `src/game_logic.h`

2. Add the necessary code to the 'game_logic.h' file
   Uses: `src/game_logic.h`

<PlandexFinish/>
```

