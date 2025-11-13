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

### ReviseSubtasksPrompt

**Purpose:** Instructions for modifying, updating, or removing existing subtasks.

**Location:** `app/server/model/prompts/planning.go`

**Description:** Provides guidelines for how to revise an existing plan by:
- Adding new subtasks using '### Tasks' section
- Modifying existing unfinished subtasks (by creating a subtask with the same exact name)
- Removing subtasks using '### Remove Tasks' section
- Never removing tasks that have already been finished

**Key Rules:**
- New subtasks use '### Tasks' section with numbered format
- Modified subtasks must have exact same name as original
- Removed subtasks use '### Remove Tasks' section with bullet list
- Cannot modify or remove completed subtasks

**Example Format:**
```
### Remove Tasks
- Task name
- Task name

### Tasks
1. Updated task (same name as before)
Uses: `file.go`

2. New task
Uses: `new_file.go`
```

### CombineSubtasksPrompt

**Purpose:** Guidelines for grouping related subtasks together intelligently.

**Location:** `app/server/model/prompts/planning.go`

**Description:** Instructions on how to combine multiple small steps into cohesive subtasks:
- Combine closely related changes into single subtask if manageable
- Size subtasks to be completable in a single response
- Use bullet points to break up multi-step subtasks
- Avoid making tasks too granular or too large
- Group file operations of the same type

**Good vs Poor Examples:**
- **Poor:** Separate tasks for creating file, adding schema, adding methods
- **Better:** Single task "Create product model with core functionality" with bulleted steps
- **Poor:** Single massive task with many unrelated features
- **Better:** Separate focused tasks for each major feature area

### SharedPlanningImplementationPrompt

**Purpose:** Shared guidelines for both planning and implementation phases.

**Location:** `app/server/model/prompts/planning.go`

**Description:** Common instructions that apply to both planning and implementation:
- Code must be robust, complete, and production-ready
- Include proper error handling, logging, security best practices
- Code organization: prefer smaller focused files over large monolithic files
- Task planning: group related file changes into cohesive subtasks
- Focus only on what user requested, no extra features
- Always use open source libraries when appropriate
- Consider latest context state when continuing plans
- Work from latest version of files and context

**Key Principles:**
- Multiple smaller files > large monolithic files (code organization)
- Single subtask can modify multiple files if tightly coupled (task planning)
- Use well-maintained, widely-used libraries with permissive licenses
- Don't modify existing code unless necessary for user's request
- Always work from latest state of context and files

### CurrentSubtaskPrompt

**Purpose:** Reminds AI to implement only the current subtask.

**Location:** `app/server/model/prompts/implement.go`

**Description:** Critical instruction that enforces:
- Implement ONLY the current subtask in this response
- Do NOT move on to next subtask
- Must complete ALL steps before marking done
- Must mark as done or in-progress before ending response

### MarkSubtaskDonePrompt

**Purpose:** Instructions for marking tasks as completed or in-progress.

**Location:** `app/server/model/prompts/implement.go`

**Description:** Defines the exact format for marking subtask status:

**Mark as Done:**
```
**[task name]** has been completed.
<PlandexFinish/>
```

**Mark as In-Progress:**
```
The task is not yet complete. I will continue working on it in the next response.
<PlandexFinish/>
```

**Key Requirements:**
- Must mark every task as done or in-progress
- Can only mark done if ALL steps implemented with code blocks
- Must use exact task name (bolded)
- Must output `<PlandexFinish/>` and immediately end response
- Special rules for .gitignore files (only add, never remove entries)

---

## 3. Implementation Phase Prompts

### GetImplementationPrompt

**Purpose:** Main prompt for the implementation phase where AI writes code to complete tasks.

**Location:** `app/server/model/prompts/implement.go`

**Function Signature:** `GetImplementationPrompt(task string) string`

**Description:** Core implementation prompt that instructs the AI on:
- Describing the approach before writing code
- Using `<PlandexBlock>` tags with `lang` and `path` attributes
- File path labels format: `- file_path:`
- Reference comments for updating existing files: `// ... existing code ...`
- Removal comments: `// Plandex: removed code`
- Including entire files when creating new files
- Never using placeholders or incomplete implementations

**Key Code Block Format:**
```
- src/main.rs:
<PlandexBlock lang="rust" path="src/main.rs">
// code here
</PlandexBlock>
```

**Critical Rules:**
- ALWAYS use file path label before `<PlandexBlock>` tag
- ALWAYS include both `lang` and `path` attributes
- NO line numbers in code blocks
- NO triple backticks, only `<PlandexBlock>` tags
- For updates: show only changing code + context
- For new files: include entire file
- NO placeholders like "// implement functionality here"
- Combine multiple changes to same file in SINGLE code block

**Supported Languages:** 165+ language identifiers (see `ValidLangIdentifiers`)

**Update vs Create:**
- **Updating:** Use reference comments, show only changes + anchors
- **Creating:** Include entire file, no reference comments

