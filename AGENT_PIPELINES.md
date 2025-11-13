# Plandex Agent Pipelines & Workflows

This document provides a comprehensive overview of all agent pipelines and workflows in Plandex, including detailed flow diagrams, data flows, and interactions between different components.

---

## Table of Contents

1. [Overview](#overview)
2. [Pipeline 1: Main Tell Pipeline](#pipeline-1-main-tell-pipeline)
3. [Pipeline 2: Auto-Context Pipeline](#pipeline-2-auto-context-pipeline)
4. [Pipeline 3: Planning Phase Pipeline](#pipeline-3-planning-phase-pipeline)
5. [Pipeline 4: Implementation Phase Pipeline](#pipeline-4-implementation-phase-pipeline)
6. [Pipeline 5: Build & Validation Pipeline](#pipeline-5-build--validation-pipeline)
7. [Pipeline 6: Apply & Execute Pipeline](#pipeline-6-apply--execute-pipeline)
8. [Pipeline 7: Chat Mode Pipeline](#pipeline-7-chat-mode-pipeline)
9. [Data Structures](#data-structures)
10. [File References](#file-references)

---

## Overview

Plandex uses a sophisticated multi-phase pipeline architecture to process user requests, break them down into tasks, implement them, validate the results, and execute commands. The system operates in an asynchronous, streaming manner with intelligent auto-continuation between phases.

### Key Concepts

- **Tell Mode:** Planning and implementing code changes
- **Chat Mode:** Conversational mode without file modifications
- **Auto-Context Mode:** Dynamic context loading during planning
- **Streaming:** Real-time output to client as AI generates responses
- **Auto-Continue:** Automatic progression between phases without user intervention
- **Build & Validation:** Multi-attempt validation with syntax checking and fixing

### Architecture Layers

```
┌──────────────────────────────────────────────────────────┐
│                    HTTP Handlers                          │
│  (TellPlanHandler, BuildPlanHandler, ApplyHandler)       │
└────────────────────┬─────────────────────────────────────┘
                     │
┌────────────────────▼─────────────────────────────────────┐
│              Model Layer (plan/)                          │
│  • State Management (ActivePlan)                          │
│  • Phase Resolution (Planning/Implementation)             │
│  • Prompt Assembly (Context + System Prompt)              │
│  • LLM Orchestration (doTellRequest)                      │
│  • Stream Processing (Chunk Parser)                       │
└────────────────────┬─────────────────────────────────────┘
                     │
┌────────────────────▼─────────────────────────────────────┐
│              Database Layer (db/)                         │
│  • ConvoMessages, Subtasks, Contexts                      │
│  • PlanBuilds, ConvoSummaries                             │
│  • CurrentPlanState                                       │
└──────────────────────────────────────────────────────────┘
```

---

## Pipeline 1: Main Tell Pipeline

**Purpose:** Orchestrate the entire user request from initial prompt to completion, handling planning, context loading, implementation, and validation.

**Entry Point:** `app/server/handlers/plans_exec.go:TellPlanHandler`

**Main Function:** `app/server/model/plan/tell_exec.go:execTellPlan()`

### ASCII Flow Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                  User Sends Tell Request                     │
│            (prompt, settings, auto-context flag)             │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────────┐
│  TellPlanHandler (HTTP)                                       │
│  • Authenticate user                                          │
│  • Parse TellPlanRequest                                      │
│  • Initialize API clients                                     │
└──────────────────────┬───────────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────────┐
│  modelPlan.Tell()                                             │
│  • Create ActivePlan (in-memory state)                        │
│  • Activate plan                                              │
│  • Spawn async execTellPlan() goroutine                       │
│  • Return immediately to client                               │
└──────────────────────┬───────────────────────────────────────┘
                       │ (async)
                       ▼
┌──────────────────────────────────────────────────────────────┐
│  execTellPlan(iteration=0)                                    │
│                                                               │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ 1. loadTellPlan() ──► Load from DB:                    │ │
│  │    • ConvoMessages (conversation history)              │ │
│  │    • Subtasks (task list with status)                  │ │
│  │    • Contexts (files, maps, pending contexts)          │ │
│  │    • ConvoSummaries (for token efficiency)             │ │
│  │    • CurrentPlanState (file states)                    │ │
│  └────────────────────────────────────────────────────────┘ │
│                       │                                       │
│                       ▼                                       │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ 2. resolveCurrentStage() ──► Determine:                │ │
│  │    • TellStage: Planning or Implementation             │ │
│  │    • PlanningPhase: Tasks or Context (if auto-context) │ │
│  │                                                         │ │
│  │    Logic:                                               │ │
│  │    If last_message.Role == User:                        │ │
│  │      └─ TellStage = Planning                            │ │
│  │         If AutoContext && hasContextMap:                │ │
│  │           └─ PlanningPhase = Context (architect)        │ │
│  │         Else:                                            │ │
│  │           └─ PlanningPhase = Tasks (planner)            │ │
│  │    Else (last_message.Role == Assistant):               │ │
│  │      If last_message.DidMakePlan:                       │ │
│  │        └─ TellStage = Implementation                    │ │
│  │      Else:                                               │ │
│  │        └─ TellStage = Planning                          │ │
│  └────────────────────────────────────────────────────────┘ │
│                       │                                       │
│                       ▼                                       │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ 3. formatModelContext() ──► Build context:             │ │
│  │    • Filter contexts by stage and token budget         │ │
│  │    • Planning: maps + manual contexts (cached)         │ │
│  │    • Implementation: smart filtered contexts           │ │
│  │    • Format into ExtendedChatMessageParts              │ │
│  │    • Apply prompt caching strategies                    │ │
│  └────────────────────────────────────────────────────────┘ │
│                       │                                       │
│                       ▼                                       │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ 4. getTellSysPrompt() ──► Generate system prompt:      │ │
│  │    Planning Stage:                                      │ │
│  │      • Tasks: GetPlanningPrompt()                       │ │
│  │      • Context: GetArchitectContextSummary()            │ │
│  │    Implementation Stage:                                │ │
│  │      • GetImplementationPrompt(currentSubtask)          │ │
│  └────────────────────────────────────────────────────────┘ │
│                       │                                       │
│                       ▼                                       │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ 5. resolvePromptMessage() ──► User prompt:             │ │
│  │    If iteration 0:                                      │ │
│  │      └─ User's actual prompt (wrapped with metadata)    │ │
│  │    Else (auto-continue):                                │ │
│  │      Planning: AutoContinuePlanningPrompt               │ │
│  │      Implementation: AutoContinueImplementationPrompt   │ │
│  └────────────────────────────────────────────────────────┘ │
│                       │                                       │
│                       ▼                                       │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ 6. doTellRequest() ──► Call LLM:                       │ │
│  │    • Assemble messages: [system, ...history, user]     │ │
│  │    • Stream response from LLM                           │ │
│  │    • Process chunks in real-time                        │ │
│  └────────────────────────────────────────────────────────┘ │
│                       │                                       │
└───────────────────────┼───────────────────────────────────────┘
                        │ (streaming)
                        ▼
┌──────────────────────────────────────────────────────────────┐
│  Stream Processing (via listeners)                            │
│                                                               │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ onChunk() ──► processChunk():                          │ │
│  │   • Parse <PlandexBlock> XML tags                       │ │
│  │   • Track file boundaries                               │ │
│  │   • Accumulate file content                             │ │
│  │   • Create Operations (file changes)                    │ │
│  │   • Stream to client in real-time                       │ │
│  └────────────────────────────────────────────────────────┘ │
│                       │                                       │
│                       ▼                                       │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ onFinish() ──► handleStreamFinished():                 │ │
│  │   • Store reply in DB (ConvoMessage)                    │ │
│  │   • Parse ### Tasks section if present                  │ │
│  │   • Create Subtasks in DB                               │ │
│  │   • Update message flags (DidMakePlan, etc.)            │ │
│  └────────────────────────────────────────────────────────┘ │
│                       │                                       │
│                       ▼                                       │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ willContinuePlan() ──► Auto-continue decision:         │ │
│  │                                                         │ │
│  │   If Planning Stage:                                    │ │
│  │     • Just created tasks? ──► Continue to Impl (YES)    │ │
│  │     • Context phase done? ──► Continue to Tasks (YES)   │ │
│  │     • New subtasks/context? ──► Continue (YES/NO)       │ │
│  │                                                         │ │
│  │   If Implementation Stage:                              │ │
│  │     • Current subtask done & more exist? ──► YES        │ │
│  │     • All subtasks done? ──► NO                         │ │
│  │                                                         │ │
│  │   Iteration limit (200) reached? ──► NO                 │ │
│  └────────────────────────────────────────────────────────┘ │
│                       │                                       │
└───────────────────────┼───────────────────────────────────────┘
                        │
                        ├──► YES: Loop back to execTellPlan(N+1)
                        │
                        ▼
                       NO
                        │
                        ▼
┌──────────────────────────────────────────────────────────────┐
│  Finalization                                                 │
│                                                               │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ Mark RepliesFinished = true                             │ │
│  └────────────────────────────────────────────────────────┘ │
│                       │                                       │
│                       ▼                                       │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ Check for pending builds:                               │ │
│  │   If builds pending:                                    │ │
│  │     └─ Set status → Building                            │ │
│  │     └─ Trigger Build Pipeline (async)                   │ │
│  │   Else:                                                  │ │
│  │     └─ active.Finish()                                   │ │
│  │     └─ Stream completion to client                       │ │
│  └────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────┘
```

### Data Flow

**Input (Iteration 0):**
- User prompt (string)
- Settings (AutoContext, ExecEnabled, BuildMode, etc.)
- Session context (API keys, OS details)

**Input (Iteration N > 0):**
- Auto-continue prompt (system-generated)
- Previous conversation state
- Current task information
- Updated contexts

**Output:**
- Stream of chunks to client (real-time)
- Database updates (messages, subtasks, contexts)
- Status updates (Replying → Building → Finished)

**Prompts Used:**
1. **Planning/Tasks:** `GetPlanningPrompt()` from `planning.go`
2. **Planning/Context:** `GetArchitectContextSummary()` from `architect_context.go`
3. **Implementation:** `GetImplementationPrompt()` from `implement.go`
4. **Auto-continue Planning:** `AutoContinuePlanningPrompt` from `user_prompt.go`
5. **Auto-continue Implementation:** `AutoContinueImplementationPrompt` from `user_prompt.go`

**Key Files:**
- `app/server/model/plan/tell_exec.go` - Main orchestrator (685 lines)
- `app/server/model/plan/tell_load.go` - Database loading (427 lines)
- `app/server/model/plan/tell_stage.go` - Phase resolution (105 lines)
- `app/server/model/plan/tell_stream_processor.go` - Chunk processing (400+ lines)
- `app/server/model/plan/tell_stream_finish.go` - Completion handling (277 lines)

---

## Pipeline 2: Auto-Context Pipeline

**Purpose:** Automatically analyze user's request and load relevant files/symbols from the codebase to provide context for planning or answering questions.

**Trigger:** When `AutoContext=true` in TellPlanRequest and a codebase map exists

**Model Used:** Architect model (configured separately, often Claude Sonnet)

### ASCII Flow Diagram

```
┌──────────────────────────────────────────────────────────────┐
│  User sends request with AutoContext=true                     │
│  "Add authentication to the user service"                     │
└──────────────────────┬───────────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────────┐
│  resolveCurrentStage()                                        │
│  • Last message is User message                               │
│  • AutoContext = true                                         │
│  • Codebase map exists and not empty                          │
│  • Not coming from context stage                              │
│  ──────────────────────────────────────────────────────────► │
│  Decision: PlanningPhase = Context                            │
└──────────────────────┬───────────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────────┐
│  formatModelContext() [for Context phase]                     │
│                                                               │
│  Includes in context:                                         │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ • Codebase map (overview of all files/symbols)         │ │
│  │ • Manual contexts (user-loaded files)                   │ │
│  │ • Pending contexts (from previous iterations)           │ │
│  │ • Git repository structure                               │ │
│  │                                                         │ │
│  │ Excludes:                                                │ │
│  │ • Auto-loaded contexts (from previous context phase)    │ │
│  └────────────────────────────────────────────────────────┘ │
└──────────────────────┬───────────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────────┐
│  getTellSysPrompt() [Context phase]                           │
│                                                               │
│  Tell Mode (IsChatOnly=false):                                │
│    Prompt: GetAutoContextTellPrompt()                         │
│    ┌────────────────────────────────────────────────────┐   │
│    │ Instructions:                                       │   │
│    │ • Create brief high-level overview (NOT full plan) │   │
│    │ • Continue with context loading in same response   │   │
│    │ • Output ### Categories and ### Files sections     │   │
│    │ • Follow interface/implementation chain rules      │   │
│    │ • Load utilities and dependencies                  │   │
│    └────────────────────────────────────────────────────┘   │
│                                                               │
│  Chat Mode (IsChatOnly=true):                                 │
│    Prompt: GetAutoContextChatPrompt()                         │
│    ┌────────────────────────────────────────────────────┐   │
│    │ Instructions:                                       │   │
│    │ • Assess what context is relevant                   │   │
│    │ • Be eager about loading (when in doubt, load)     │   │
│    │ • Mention what you need to check naturally         │   │
│    │ • Output context sections if needed                │   │
│    └────────────────────────────────────────────────────┘   │
└──────────────────────┬───────────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────────┐
│  doTellRequest() ──► LLM Call (Architect model)               │
│  User prompt: Original user request                           │
└──────────────────────┬───────────────────────────────────────┘
                       │ (streaming)
                       ▼
┌──────────────────────────────────────────────────────────────┐
│  Stream Processing                                            │
│                                                               │
│  Expected Output Format:                                      │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ [Brief architectural overview]                          │ │
│  │                                                         │ │
│  │ ### Categories                                          │ │
│  │ - Authentication                                        │ │
│  │ - User Management                                       │ │
│  │ - Database                                              │ │
│  │                                                         │ │
│  │ ### Files                                               │ │
│  │ Authentication:                                         │ │
│  │ `src/auth/interface.go` - Auth, AuthManager (main)     │ │
│  │ `src/auth/jwt.go` - JWTAuth, generateToken            │ │
│  │                                                         │ │
│  │ User Management:                                        │ │
│  │ `src/user/service.go` - UserService, GetUser          │ │
│  │ `src/user/model.go` - User struct                     │ │
│  │                                                         │ │
│  │ Database:                                               │ │
│  │ `src/db/connection.go` - Connect, GetDB               │ │
│  │                                                         │ │
│  │ <PlandexFinish/>                                        │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                               │
│  Parser extracts:                                             │
│    • Category names                                           │
│    • File paths (must exist in codebase map)                 │
│    • Symbol names for each file                              │
│    • Annotations (optional)                                   │
└──────────────────────┬───────────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────────┐
│  handleStreamFinished() [Context phase]                       │
│                                                               │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ 1. Parse context sections                               │ │
│  │    • Extract file paths from ### Files section          │ │
│  │    • Validate files exist in codebase map               │ │
│  │    • Extract requested symbols                          │ │
│  └────────────────────────────────────────────────────────┘ │
│                       │                                       │
│                       ▼                                       │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ 2. Load file contents                                   │ │
│  │    For each requested file:                             │ │
│  │      • Read file from repository                        │ │
│  │      • Extract requested symbols (if specified)         │ │
│  │      • Create Context record in DB                      │ │
│  │      • Mark as auto-loaded                              │ │
│  └────────────────────────────────────────────────────────┘ │
│                       │                                       │
│                       ▼                                       │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ 3. Update message flags                                 │ │
│  │    • WasContextStage = true                             │ │
│  │    • Store activated paths in ConvoMessage              │ │
│  └────────────────────────────────────────────────────────┘ │
│                       │                                       │
│                       ▼                                       │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ 4. willContinuePlan() Decision                          │ │
│  │    Context phase completed:                             │ │
│  │      ──► Continue = YES (auto-continue to next phase)   │ │
│  │                                                         │ │
│  │    Next phase:                                           │ │
│  │      • Tell Mode: Planning/Tasks phase                  │ │
│  │      • Chat Mode: Chat response with loaded context     │ │
│  └────────────────────────────────────────────────────────┘ │
└──────────────────────┬───────────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────────┐
│  Auto-continue to next iteration                              │
│  execTellPlan(iteration=N+1)                                  │
│                                                               │
│  Now with loaded contexts:                                    │
│    • Codebase map + manual contexts + auto-loaded contexts   │
│    • resolveCurrentStage() ──► Planning/Tasks or Chat        │
│    • Proceeds with full context available                    │
└──────────────────────────────────────────────────────────────┘
```

### Context Loading Rules

The architect model follows these rules when selecting context:

1. **Interface & Implementation Rule:**
   - Load both interface definitions and implementations
   - Example: `auth/interface.go` + `auth/jwt_impl.go`

2. **Reference Implementation Rule:**
   - Load existing similar features as reference
   - Example: Adding posts? Load existing comments implementation

3. **API Client Chain Rule:**
   - Load API interface + client implementation + types
   - Example: `api/interface.go` + `api/client.go` + `api/types.go`

4. **Database Chain Rule:**
   - Load model files + database helpers + similar operations
   - Example: `models/user.go` + `db/queries.go` + similar CRUD operations

5. **Utility Dependencies Rule:**
   - Load ALL utility files that might be needed
   - Example: Logging, error handling, validation utilities

### Data Flow

**Input:**
- User prompt describing the task
- Codebase map (file structure + symbols)
- Manual contexts (user-loaded files)
- Previous conversation history

**Prompts Used:**
1. **Tell Mode:** `GetAutoContextTellPrompt()` from `architect_context.go`
2. **Chat Mode:** `GetAutoContextChatPrompt()` from `architect_context.go`
3. **Shared Logic:** `GetAutoContextShared()` from `architect_context.go`

**Output:**
- List of loaded files with symbols
- Context records in database (marked as auto-loaded)
- Updated conversation state
- Auto-continuation to next phase

**Key Files:**
- `app/server/model/prompts/architect_context.go` - Context prompts (350+ lines)
- `app/server/model/plan/tell_stage.go` - Phase determination (105 lines)
- `app/server/model/plan/tell_context.go` - Context formatting (350+ lines)

---

## Pipeline 3: Planning Phase Pipeline

**Purpose:** Break down user's request into a structured list of subtasks with file dependencies.

**Trigger:** When in Planning stage and PlanningPhase = Tasks

**Model Used:** Planner model (configured separately, often Claude Sonnet)

### ASCII Flow Diagram

```
┌──────────────────────────────────────────────────────────────┐
│  Entering Planning/Tasks Phase                                │
│  (after auto-context or directly if AutoContext=false)        │
└──────────────────────┬───────────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────────┐
│  formatModelContext() [Planning Stage]                        │
│                                                               │
│  Includes (all CACHED for efficiency):                        │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ • Codebase map (if exists)                              │ │
│  │ • Manual contexts (user-loaded files)                   │ │
│  │ • Auto-loaded contexts (from auto-context phase)        │ │
│  │ • Pending contexts (files with pending changes)         │ │
│  │ • Current plan state (existing file contents)           │ │
│  │ • Git repository info (if git repo)                     │ │
│  │ • _apply.sh current state                               │ │
│  │ • _apply.sh execution history                           │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                               │
│  Note: Planning contexts are heavily cached to reduce        │
│        costs across multiple planning iterations             │
└──────────────────────┬───────────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────────┐
│  getTellSysPrompt() [Planning Stage]                          │
│                                                               │
│  System Prompt: GetPlanningPrompt()                           │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ Base Instructions:                                      │ │
│  │ • Identity: "You are Plandex, an AI assistant..."      │ │
│  │ • Determine if user has task or is chatting            │ │
│  │ • Create plan with numbered subtasks                   │ │
│  │ • Use ### Tasks section format                         │ │
│  │ • Include file dependencies with Uses: backtick lists  │ │
│  │ • Output <PlandexFinish/> after tasks                  │ │
│  │                                                         │ │
│  │ Conditional Additions:                                  │ │
│  │ • If AutoContext: Include auto-context instructions    │ │
│  │ • If ExecMode: Include _apply.sh instructions          │ │
│  │ • If IsUserDebug: Include debugging prompt             │ │
│  │ • If IsApplyDebug: Include _apply.sh debug prompt      │ │
│  │ • If IsGitRepo: Use .gitignore, else .plandexignore    │ │
│  │                                                         │ │
│  │ Additional Prompts:                                     │ │
│  │ • SharedPlanningImplementationPrompt (code quality)    │ │
│  │ • FileOpsPlanningPrompt (if applicable)                │ │
│  │ • ApplyScriptPlanningPrompt (if ExecMode)              │ │
│  │ • PlanningFlowControl (critical execution rules)       │ │
│  └────────────────────────────────────────────────────────┘ │
└──────────────────────┬───────────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────────┐
│  resolvePromptMessage() [Planning Stage]                      │
│                                                               │
│  If iteration 0:                                              │
│    User Prompt: GetWrappedPrompt()                            │
│    ┌────────────────────────────────────────────────────┐   │
│    │ Wrapper includes:                                   │   │
│    │ • Timestamp and OS details                          │   │
│    │ • Shared instructions                               │   │
│    │ • Stage-specific wrapper (Planning)                 │   │
│    │ • User's actual prompt                              │   │
│    └────────────────────────────────────────────────────┘   │
│                                                               │
│  Else (auto-continue):                                        │
│    User Prompt: AutoContinuePlanningPrompt                    │
│    ┌────────────────────────────────────────────────────┐   │
│    │ "Continue the plan according to your instructions  │   │
│    │  for the current stage (planning phase)."          │   │
│    └────────────────────────────────────────────────────┘   │
└──────────────────────┬───────────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────────┐
│  doTellRequest() ──► LLM Call (Planner model)                 │
└──────────────────────┬───────────────────────────────────────┘
                       │ (streaming)
                       ▼
┌──────────────────────────────────────────────────────────────┐
│  Stream Processing                                            │
│                                                               │
│  Expected Output Format:                                      │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ [High-level plan discussion]                            │ │
│  │                                                         │ │
│  │ ### Tasks                                               │ │
│  │                                                         │ │
│  │ 1. Create authentication interface                      │ │
│  │    - Define Auth interface with required methods       │ │
│  │    - Add JWT token generation method                   │ │
│  │    Uses: `src/auth/interface.go`                       │ │
│  │                                                         │ │
│  │ 2. Implement JWT authentication                         │ │
│  │    - Create JWTAuth struct implementing Auth           │ │
│  │    - Add token validation logic                        │ │
│  │    Uses: `src/auth/jwt.go`, `src/auth/interface.go`   │ │
│  │                                                         │ │
│  │ 3. Integrate auth into user service                     │ │
│  │    - Update UserService to use Auth                    │ │
│  │    - Add middleware for protected routes               │ │
│  │    Uses: `src/user/service.go`, `src/auth/interface.go`│
│  │                                                         │ │
│  │ 4. Install dependencies and test                        │ │
│  │    - Install JWT library                               │ │
│  │    - Run tests                                          │ │
│  │    Uses: `_apply.sh`                                    │ │
│  │                                                         │ │
│  │ <PlandexFinish/>                                        │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                               │
│  Parser extracts:                                             │
│    • Subtask titles and descriptions                         │
│    • File dependencies (Uses: paths in backticks)            │
│    • Special sections: ### Remove Tasks, ### Categories      │
└──────────────────────┬───────────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────────┐
│  handleStreamFinished() [Planning Stage]                      │
│                                                               │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ 1. Parse ### Tasks section                              │ │
│  │    • Extract numbered list of subtasks                  │ │
│  │    • Parse Uses: lines for file dependencies            │ │
│  │    • Validate file paths format                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                       │                                       │
│                       ▼                                       │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ 2. Create Subtasks in DB                                │ │
│  │    For each subtask:                                    │ │
│  │      • Create db.Subtask record                         │ │
│  │      • Store title, description                         │ │
│  │      • Store UsesFiles array                            │ │
│  │      • Set IsFinished = false                           │ │
│  │      • Set CreatedAt timestamp                          │ │
│  └────────────────────────────────────────────────────────┘ │
│                       │                                       │
│                       ▼                                       │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ 3. Handle ### Remove Tasks (if present)                 │ │
│  │    • Find subtasks matching exact names                 │ │
│  │    • Mark as deleted (soft delete)                      │ │
│  │    • Cannot remove completed subtasks                   │ │
│  └────────────────────────────────────────────────────────┘ │
│                       │                                       │
│                       ▼                                       │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ 4. Update ConvoMessage flags                            │ │
│  │    • DidMakePlan = true                                 │ │
│  │    • CurrentStage = Planning                            │ │
│  └────────────────────────────────────────────────────────┘ │
│                       │                                       │
│                       ▼                                       │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ 5. willContinuePlan() Decision                          │ │
│  │    If tasks were created:                               │ │
│  │      ──► Continue = YES                                 │ │
│  │    Else if tasks were modified:                         │ │
│  │      ──► Assess if more planning needed                 │ │
│  │                                                         │ │
│  │    Next stage: Implementation                           │ │
│  └────────────────────────────────────────────────────────┘ │
└──────────────────────┬───────────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────────┐
│  Auto-continue to Implementation                              │
│  execTellPlan(iteration=N+1, AutoContinuePrompt)              │
│                                                               │
│  • resolveCurrentStage() ──► TellStage = Implementation      │
│  • Load first incomplete subtask                             │
│  • Proceed to Implementation Phase                           │
└──────────────────────────────────────────────────────────────┘
```

### Data Flow

**Input:**
- User prompt (iteration 0) or AutoContinuePlanningPrompt
- Codebase map + all loaded contexts
- Existing conversation history
- Current plan state (if revising plan)
- Existing subtasks (if revising plan)
- _apply.sh state and history (if ExecMode)

**Prompts Used:**
1. **Main Planning:** `GetPlanningPrompt()` from `planning.go`
2. **User Prompt Wrapper:** `GetWrappedPrompt()` from `user_prompt.go`
3. **Auto-Continue:** `AutoContinuePlanningPrompt` from `user_prompt.go`
4. **Shared Quality Rules:** `SharedPlanningImplementationPrompt` from `planning.go`
5. **File Operations:** `FileOpsPlanningPrompt` from `file_ops.go` (if applicable)
6. **Execution:** `ApplyScriptPlanningPrompt` from `apply_exec.go` (if ExecMode)
7. **Flow Control:** `PlanningFlowControl` from `planning.go`

**Output:**
- List of Subtasks in database with:
  - Title and description
  - File dependencies (UsesFiles)
  - Completion status (IsFinished = false initially)
- Updated conversation message with DidMakePlan flag
- Auto-continuation to Implementation phase

**Key Files:**
- `app/server/model/prompts/planning.go` - Planning prompts (600+ lines)
- `app/server/model/plan/tell_stream_finish.go` - Task parsing (277 lines)
- `app/server/model/plan/tell_stage.go` - Stage resolution (105 lines)

---

## Pipeline 4: Implementation Phase Pipeline

**Purpose:** Execute subtasks one at a time, generating code blocks with <PlandexBlock> tags.

**Trigger:** When TellStage = Implementation and incomplete subtasks exist

**Model Used:** Coder model (configured separately, often Claude Sonnet)

### ASCII Flow Diagram

```
┌──────────────────────────────────────────────────────────────┐
│  Entering Implementation Stage                                │
│  (after planning phase completes)                             │
└──────────────────────┬───────────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────────┐
│  formatModelContext() [Implementation Stage]                  │
│                                                               │
│  Smart Context Filtering:                                     │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ Only include files relevant to CURRENT SUBTASK:        │ │
│  │ • Files in current subtask's Uses: list                │ │
│  │ • Files with pending changes (from previous tasks)     │ │
│  │ • Current plan state for those files                   │ │
│  │ • _apply.sh current state (if in Uses)                 │ │
│  │                                                         │ │
│  │ EXCLUDED from implementation context:                   │ │
│  │ • Codebase map (too large, not needed)                 │ │
│  │ • Files NOT in current subtask's Uses                  │ │
│  │ • Manual contexts not relevant to subtask              │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                               │
│  Token Budget: Smaller (~200k) vs Planning (~400k)           │
└──────────────────────┬───────────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────────┐
│  getTellSysPrompt() [Implementation Stage]                    │
│                                                               │
│  System Prompt: GetImplementationPrompt(currentSubtask)       │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ Instructions include:                                   │ │
│  │ • Use <PlandexBlock lang="..." path="...">            │ │
│  │ • File path labels: - file_path:                       │ │
│  │ • Reference comments: // ... existing code ...         │ │
│  │ • Removal comments: // Plandex: removed code           │ │
│  │ • Action Explanation Format (mandatory)                │ │
│  │ • CurrentSubtaskPrompt (do ONLY current task)          │ │
│  │ • MarkSubtaskDonePrompt (completion format)            │ │
│  │ • SharedPlanningImplementationPrompt (quality)         │ │
│  │ • FileOpsImplementationPrompt (if applicable)          │ │
│  │ • ApplyScriptImplementationPrompt (if ExecMode)        │ │
│  │                                                         │ │
│  │ Current Subtask Info:                                   │ │
│  │ • Subtask title and description                        │ │
│  │ • List of files to modify (Uses:)                      │ │
│  │ • Previous subtasks context                            │ │
│  └────────────────────────────────────────────────────────┘ │
└──────────────────────┬───────────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────────┐
│  resolvePromptMessage() [Implementation Stage]                │
│                                                               │
│  If iteration 0 (first subtask):                              │
│    • Empty prompt or AutoContinueImplementationPrompt         │
│                                                               │
│  Else (continuing subtasks):                                  │
│    • AutoContinueImplementationPrompt:                        │
│      "Continue with the next subtask"                         │
└──────────────────────┬───────────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────────┐
│  doTellRequest() ──► LLM Call (Coder model)                   │
└──────────────────────┬───────────────────────────────────────┘
                       │ (streaming)
                       ▼
┌──────────────────────────────────────────────────────────────┐
│  Stream Processing                                            │
│                                                               │
│  Expected Output Format:                                      │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ [Approach description]                                  │ │
│  │                                                         │ │
│  │ ### Action Explanation Format                           │ │
│  │ **Updating `src/auth/interface.go`**                   │ │
│  │ Type: add                                               │ │
│  │ Summary: Add Auth interface with JWT methods           │ │
│  │ Context: `package auth`, end of file                   │ │
│  │                                                         │ │
│  │ - src/auth/interface.go:                               │ │
│  │ <PlandexBlock lang="go" path="src/auth/interface.go">  │ │
│  │ package auth                                            │ │
│  │                                                         │ │
│  │ // ... existing code ...                               │ │
│  │                                                         │ │
│  │ type Auth interface {                                  │ │
│  │     GenerateToken(user User) (string, error)           │ │
│  │     ValidateToken(token string) (*User, error)         │ │
│  │ }                                                       │ │
│  │ </PlandexBlock>                                         │ │
│  │                                                         │ │
│  │ **[Current subtask name]** has been completed.          │ │
│  │ <PlandexFinish/>                                        │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                               │
│  processChunk() extracts:                                     │
│    • File operations (Create/Update/Delete)                  │
│    • File paths and content                                  │
│    • Language identifiers                                    │
│    • Completion markers                                      │
└──────────────────────┬───────────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────────┐
│  handleStreamFinished() [Implementation Stage]                │
│                                                               │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ 1. Store Operations in DB                               │ │
│  │    • Each file change → PlanOperation record            │ │
│  │    • Link to ConvoMessage and Subtask                   │ │
│  └────────────────────────────────────────────────────────┘ │
│                       │                                       │
│                       ▼                                       │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ 2. Check Subtask Completion                             │ │
│  │    • Parse completion marker (bolded task name)         │ │
│  │    • If marked done: Set IsFinished = true             │ │
│  │    • Update CompletedAt timestamp                       │ │
│  └────────────────────────────────────────────────────────┘ │
│                       │                                       │
│                       ▼                                       │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ 3. Create Build Records                                 │ │
│  │    • For each file with operations: PlanBuild          │ │
│  │    • Status: Pending                                    │ │
│  └────────────────────────────────────────────────────────┘ │
│                       │                                       │
│                       ▼                                       │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ 4. willContinuePlan() Decision                          │ │
│  │    • Current subtask done + more exist? ──► YES         │ │
│  │    • All subtasks done? ──► NO                          │ │
│  │    • Iteration limit (200) reached? ──► NO              │ │
│  └────────────────────────────────────────────────────────┘ │
└──────────────────────┬───────────────────────────────────────┘
                       │
                       ├──► YES: Loop to next subtask (N+1)
                       │
                       ▼ NO (all done)
┌──────────────────────────────────────────────────────────────┐
│  Mark RepliesFinished = true                                  │
│  Set Status → Building (if builds pending)                    │
│  Trigger Build Pipeline                                       │
└──────────────────────────────────────────────────────────────┘
```

**Prompts Used:**
1. `GetImplementationPrompt()` - `implement.go`
2. `ChangeExplanationPrompt` - `explanation_format.go`
3. `UpdateFormatPrompt` - `update_format.go`
4. `CurrentSubtaskPrompt` - `implement.go`
5. `MarkSubtaskDonePrompt` - `implement.go`

**Key Files:**
- `app/server/model/prompts/implement.go` (400+ lines)
- `app/server/model/plan/tell_stream_processor.go` (chunk parsing)
- `app/server/model/plan/tell_context.go` (smart filtering)

---

## Pipeline 5: Build & Validation Pipeline

**Purpose:** Apply proposed changes to files, validate syntax, and fix errors through multiple attempts.

**Trigger:** After Implementation phase completes with file operations

**Entry Point:** `BuildPlanHandler` or auto-triggered after Implementation

### ASCII Flow Diagram

```
┌──────────────────────────────────────────────────────────────┐
│  Build Trigger                                                │
│  (user clicks "Build" OR auto-build after Implementation)     │
└──────────────────────┬───────────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────────┐
│  modelPlan.Build(BuildParams)                                 │
│  • Load all pending PlanBuild records                         │
│  • Set ActivePlan status → Building                           │
│  • Queue builds for each file                                 │
└──────────────────────┬───────────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────────┐
│  For Each File: processBuild() [parallel goroutines]          │
│                                                               │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ Load Data:                                              │ │
│  │ • Original file (preBuildState)                         │ │
│  │ • Proposed changes (from Operations)                    │ │
│  │ • Proposed changes explanation                          │ │
│  │ • Conversation context                                  │ │
│  └────────────────────────────────────────────────────────┘ │
│                       │                                       │
│                       ▼                                       │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ buildValidateLoop (max 3 attempts)                      │ │
│  │                                                         │ │
│  │ Attempt 1: Apply Changes                                │ │
│  │   ┌───────────────────────────────────────────────┐   │ │
│  │   │ applyFileUpdates():                            │   │ │
│  │   │ • Parse proposed changes                       │   │ │
│  │   │ • Apply to original file                       │   │ │
│  │   │ • Replace reference comments with actual code  │   │ │
│  │   │ • Handle removal comments                      │   │ │
│  │   └───────────────────────────────────────────────┘   │ │
│  │                       │                                 │ │
│  │                       ▼                                 │ │
│  │   ┌───────────────────────────────────────────────┐   │ │
│  │   │ Syntax Validation (if applicable language):    │   │ │
│  │   │ • Parse file with tree-sitter                  │   │ │
│  │   │ • Check for syntax errors                      │   │ │
│  │   │ • Extract error messages                       │   │ │
│  │   └───────────────────────────────────────────────┘   │ │
│  │                       │                                 │ │
│  │                       ├──► Valid? ──► SUCCESS          │ │
│  │                       │                                 │ │
│  │                       ▼ Invalid                         │ │
│  │   ┌───────────────────────────────────────────────┐   │ │
│  │   │ Needs Verification Check:                      │   │ │
│  │   │ • Code removed/duplicated?                     │   │ │
│  │   │ • Ambiguous location?                          │   │ │
│  │   │ • Syntax errors?                               │   │ │
│  │   └───────────────────────────────────────────────┘   │ │
│  │                       │                                 │ │
│  │                       ▼ YES                             │ │
│  │   ┌───────────────────────────────────────────────┐   │ │
│  │   │ AI Validation Request:                         │   │ │
│  │   │ Prompt: GetValidationReplacementsXmlPrompt()   │   │ │
│  │   │ Model: Builder (validation specialist)         │   │ │
│  │   │                                                │   │ │
│  │   │ Provides:                                      │   │ │
│  │   │ • Original file with line numbers (pdx-)       │   │ │
│  │   │ • Proposed changes with line numbers (pdx-new-)│   │ │
│  │   │ • Diff of applied changes                      │   │ │
│  │   │ • Reasons for verification                     │   │ │
│  │   │ • Syntax errors (if any)                       │   │ │
│  │   └───────────────────────────────────────────────┘   │ │
│  │                       │                                 │ │
│  │                       ▼                                 │ │
│  │   ┌───────────────────────────────────────────────┐   │ │
│  │   │ AI Response:                                   │   │ │
│  │   │ • <PlandexCorrect/> ──► SUCCESS                │   │ │
│  │   │ OR                                             │   │ │
│  │   │ • <PlandexIncorrect/>                          │   │ │
│  │   │   + <PlandexReplacements>                      │   │ │
│  │   │     - Fixed code blocks                        │   │ │
│  │   │     - Exact old/new replacements               │   │ │
│  │   └───────────────────────────────────────────────┘   │ │
│  │                       │                                 │ │
│  │                       ▼ Incorrect                       │ │
│  │ Attempt 2: Apply AI Fixes + Use Stronger Model         │ │
│  │   • Apply replacement blocks from AI                   │ │
│  │   • Validate syntax again                              │ │
│  │   • If still failing, use stronger model (if available)│ │
│  │                       │                                 │ │
│  │                       ▼ Still failing                   │ │
│  │ Attempt 3: Final attempt with validation-only flag     │ │
│  │   • One more validation request                        │ │
│  │   • Use strongest available model                      │ │
│  │   • Accept result even if not perfect                  │ │
│  │                       │                                 │ │
│  │                       ▼                                 │ │
│  │ Return: Best attempt (may include errors)              │ │
│  └────────────────────────────────────────────────────────┘ │
│                       │                                       │
│                       ▼                                       │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ On File Completion:                                     │ │
│  │ • Write file to git repository                          │ │
│  │ • Git commit changes                                    │ │
│  │ • Update PlanBuild status → Completed                   │ │
│  │ • Stream build result to client                         │ │
│  └────────────────────────────────────────────────────────┘ │
└──────────────────────┬───────────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────────┐
│  When All Files Done: onFinishBuild()                         │
│  • Check allBuildsFinished()                                  │
│  • If yes + repliesFinished:                                  │
│    └─ active.Finish() (complete plan execution)              │
│  • Stream completion to client                                │
└──────────────────────────────────────────────────────────────┘
```

**Prompts Used:**
1. `GetValidationReplacementsXmlPrompt()` - `build_validation_replacements.go`
2. `CommentClassifierPrompt` - `build_helpers.go`
3. `WholeFilePrompt` - `build_whole_file.go` (fallback)

**Key Files:**
- `app/server/model/plan/build_exec.go` (200+ lines)
- `app/server/model/plan/build_validate_and_fix.go` (350+ lines)
- `app/server/syntax/structured_edits_apply.go` (tree-sitter)

---

## Pipeline 6: Apply & Execute Pipeline

**Purpose:** Apply built files to user's working directory and execute _apply.sh script.

**Trigger:** User runs `plandex apply` CLI command

**Entry Point:** CLI `MustApplyPlan()` function

### ASCII Flow Diagram

```
┌──────────────────────────────────────────────────────────────┐
│  User runs: plandex apply                                     │
└──────────────────────┬───────────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────────┐
│  MustApplyPlan(params) [CLI]                                  │
│                                                               │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ Pre-checks:                                             │ │
│  │ • Pending builds exist? ──► Prompt user to build first  │ │
│  │ • Get current plan state from server                    │ │
│  │ • Check for file conflicts                              │ │
│  └────────────────────────────────────────────────────────┘ │
└──────────────────────┬───────────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────────┐
│  Apply File Changes                                           │
│                                                               │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ For each file with changes:                             │ │
│  │   • Read built file from plan repo (git)                │ │
│  │   • Write to user's working directory                   │ │
│  │   • Create directories if needed                        │ │
│  │   • Handle file operations (moves, removals)            │ │
│  │   • Show diff to user (if enabled)                      │ │
│  └────────────────────────────────────────────────────────┘ │
└──────────────────────┬───────────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────────┐
│  Execute _apply.sh (if ExecMode && script exists)             │
│                                                               │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ Setup:                                                  │ │
│  │ • Create wrapper script with error handling            │ │
│  │ • Set up process group for signal handling             │ │
│  │ • Enable job control (set -m)                          │ │
│  │ • Inject 'set -e' for fail-fast behavior              │ │
│  └────────────────────────────────────────────────────────┘ │
│                       │                                       │
│                       ▼                                       │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ Execute:                                                │ │
│  │ • Run wrapper script in user's working directory       │ │
│  │ • Stream stdout/stderr to user in real-time            │ │
│  │ • Show command output as it runs                       │ │
│  │ • Monitor for 'plandex browser' command                │ │
│  │   └─ If found: Launch browser, capture console logs   │ │
│  └────────────────────────────────────────────────────────┘ │
│                       │                                       │
│                       ├──► SUCCESS ─┐                         │
│                       │             │                         │
│                       ▼ FAILURE     │                         │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ On Failure:                                             │ │
│  │ • Show error output                                     │ │
│  │ • Offer options:                                        │ │
│  │   1. Debug (send error to AI for fix)                  │ │
│  │   2. Auto-mode (auto-retry N times)                    │ │
│  │   3. Rollback (revert changes)                         │ │
│  │   4. Apply anyway (keep changes, ignore script error)  │ │
│  │                                                         │ │
│  │ If autoDebug > 0:                                       │ │
│  │   └─ Auto-retry up to autoDebug times                  │ │
│  │   └─ Send error + script to AI for debugging           │ │
│  │   └─ AI suggests fixes                                 │ │
│  │   └─ Apply fixes and re-run                            │ │
│  └────────────────────────────────────────────────────────┘ │
│                       │                                       │
└───────────────────────┼───────────────────────────────────────┘
                        │
                        ▼
┌──────────────────────────────────────────────────────────────┐
│  Post-Apply Actions                                           │
│                                                               │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ If success and autoCommit enabled:                      │ │
│  │   • Generate commit message from plan                   │ │
│  │   • Commit changes to user's git repo                   │ │
│  └────────────────────────────────────────────────────────┘ │
│                       │                                       │
│                       ▼                                       │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ If success and autoApply enabled:                       │ │
│  │   • Auto-continue plan with next user prompt            │ │
│  └────────────────────────────────────────────────────────┘ │
│                       │                                       │
│                       ▼                                       │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ Mark plan as applied in database                        │ │
│  │ Return to user                                          │ │
│  └────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────┘
```

**Script Generation Prompts:**
1. `ApplyScriptSharedPrompt` - Core instructions
2. `ApplyScriptPlanningPrompt` - Planning phase additions
3. `ApplyScriptImplementationPrompt` - Implementation phase additions
4. `ApplyPlanningDebugPrompt` - Debug mode prompt
5. `ExitCodePrompt` - Exit code handling

**Key Files:**
- `app/cli/plan_exec/apply_exec.go` (CLI-side)
- `app/server/model/prompts/apply_exec.go` (prompts, 600+ lines)

---

## Pipeline 7: Chat Mode Pipeline

**Purpose:** Enable conversational mode where AI discusses code without making changes.

**Trigger:** User sends prompt with `--chat` flag or `IsChatOnly=true`

**Model Used:** Chat model (often same as Planner)

### ASCII Flow Diagram

```
┌──────────────────────────────────────────────────────────────┐
│  User sends chat prompt                                       │
│  plandex tell --chat "How does auth work?"                    │
└──────────────────────┬───────────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────────┐
│  TellPlanHandler receives request                             │
│  • IsChatOnly = true                                          │
│  • AutoContext = true/false (user's choice)                   │
└──────────────────────┬───────────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────────┐
│  execTellPlan(iteration=0)                                    │
│                                                               │
│  resolveCurrentStage():                                       │
│  • If AutoContext + no context yet:                           │
│    └─ PlanningPhase = Context (load context first)           │
│  • Else:                                                       │
│    └─ TellStage = Chat (skip planning/implementation)        │
└──────────────────────┬───────────────────────────────────────┘
                       │
                       ├──► AutoContext? ──► YES ─┐
                       │                          │
                       ▼ NO                       ▼
┌─────────────────────────────────┐   ┌──────────────────────┐
│  Skip to Chat Response          │   │  Load Context First  │
└──────────────────┬──────────────┘   │  (See Pipeline 2)    │
                   │                   └──────────┬───────────┘
                   │                              │
                   │  ◄───────────────────────────┘
                   │  (after context loaded, auto-continue)
                   ▼
┌──────────────────────────────────────────────────────────────┐
│  getTellSysPrompt() [Chat Mode]                               │
│                                                               │
│  System Prompt: GetChatSysPrompt()                            │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ Instructions:                                           │ │
│  │ • You are Plandex (AI assistant)                        │ │
│  │ • This is chat-only mode                                │ │
│  │ • Do NOT create or modify files                         │ │
│  │ • Do NOT output <PlandexBlock> tags                     │ │
│  │ • Do NOT make formal plans                              │ │
│  │ • CAN discuss code, suggest approaches, explain         │ │
│  │ • CAN include code snippets for explanation             │ │
│  │ • CAN reference files from context                      │ │
│  │ • Mention "switch to tell mode" if user wants changes   │ │
│  │                                                         │ │
│  │ ExecMode awareness:                                     │ │
│  │ • If enabled: Discuss both files and commands           │ │
│  │ • If disabled: Focus on files, mention exec needed     │ │
│  └────────────────────────────────────────────────────────┘ │
└──────────────────────┬───────────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────────┐
│  doTellRequest() ──► LLM Call (Chat model)                    │
│  User prompt: Original chat question                          │
└──────────────────────┬───────────────────────────────────────┘
                       │ (streaming)
                       ▼
┌──────────────────────────────────────────────────────────────┐
│  Stream Processing                                            │
│                                                               │
│  Expected Output Format:                                      │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ [Natural conversational response]                       │ │
│  │                                                         │ │
│  │ Authentication in this codebase works through the      │ │
│  │ `Auth` interface defined in `src/auth/interface.go`.   │ │
│  │                                                         │ │
│  │ Here's the key interface:                              │ │
│  │ ```go                                                   │ │
│  │ type Auth interface {                                  │ │
│  │     GenerateToken(user User) (string, error)           │ │
│  │     ValidateToken(token string) (*User, error)         │ │
│  │ }                                                       │ │
│  │ ```                                                     │ │
│  │                                                         │ │
│  │ The JWT implementation (`src/auth/jwt.go`) handles     │ │
│  │ token generation and validation...                      │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                               │
│  Parser:                                                      │
│    • No special parsing (regular markdown)                   │
│    • No file operations extracted                            │
│    • No subtasks created                                     │
└──────────────────────┬───────────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────────┐
│  handleStreamFinished() [Chat Mode]                           │
│  • Store reply in conversation history                        │
│  • No builds triggered                                        │
│  • No subtasks created                                        │
│  • No auto-continuation (unless user asks follow-up)          │
│  • Stream completion to client                                │
└──────────────────────────────────────────────────────────────┘
```

**Switching to Tell Mode:**
```
User: "Actually, can you implement that?"
AI: "I'd be happy to help implement that! Would you like to
     switch to tell mode so I can create files and make changes?"

User: "plandex tell 'implement authentication'"
[Now in Tell Mode, follows Pipeline 1]
```

**Prompts Used:**
1. `GetChatSysPrompt()` - `chat.go`
2. `GetWrappedChatOnlyPrompt()` - `user_prompt.go`

**Key Files:**
- `app/server/model/prompts/chat.go` (100 lines)
- `app/server/model/plan/tell_exec.go` (chat branch)

---

## Data Structures

### ActivePlan
In-memory state tracker for running plans:
```go
type ActivePlan struct {
  PlanId, Branch        string
  Status                PlanStatus
  CurrentReplyContent   string
  Contexts              []*Context
  ActiveBuilds          map[string]*ActiveBuild
  RepliesFinished       bool
}
```

### ConvoMessage
Database record for conversation messages:
```go
type ConvoMessage struct {
  Id, PlanId            string
  Role                  string (User/Assistant)
  Message               string
  DidMakePlan           bool
  CurrentStage          CurrentStage
  WasContextStage       bool
  ActivatedContextPaths []string
}
```

### Subtask
Database record for plan tasks:
```go
type Subtask struct {
  Id, PlanId, Title     string
  Description           string
  UsesFiles             []string
  IsFinished            bool
  StartedAt, CompletedAt *time.Time
}
```

### PlanBuild
Database record for build tracking:
```go
type PlanBuild struct {
  Id, PlanId, FilePath  string
  Status                BuildStatus
  Error                 string
  NumRetries            int
}
```

---

## File References

### Core Orchestration
- `app/server/model/plan/tell_exec.go` (685 lines) - Main pipeline
- `app/server/model/plan/tell_load.go` (427 lines) - Database loading
- `app/server/model/plan/tell_stage.go` (105 lines) - Phase resolution
- `app/server/model/plan/tell_stream_processor.go` (400+ lines) - Streaming
- `app/server/model/plan/tell_stream_finish.go` (277 lines) - Completion

### Prompt Definitions
- `app/server/model/prompts/planning.go` (600+ lines) - Planning prompts
- `app/server/model/prompts/implement.go` (400+ lines) - Implementation prompts
- `app/server/model/prompts/architect_context.go` (350+ lines) - Context prompts
- `app/server/model/prompts/chat.go` (100 lines) - Chat prompts
- `app/server/model/prompts/apply_exec.go` (600+ lines) - _apply.sh prompts

### Build & Validation
- `app/server/model/plan/build_exec.go` (200+ lines) - Build orchestration
- `app/server/model/plan/build_validate_and_fix.go` (350+ lines) - Validation
- `app/server/model/prompts/build_validation_replacements.go` - Validation prompts
- `app/server/syntax/structured_edits_apply.go` - Tree-sitter validation

### HTTP Handlers
- `app/server/handlers/plans_exec.go` - TellPlanHandler, BuildPlanHandler

### CLI
- `app/cli/plan_exec/apply_exec.go` - Apply & Execute

---

## Summary

Plandex's pipeline architecture provides a sophisticated, multi-phase approach to AI-assisted coding:

1. **Main Tell Pipeline** orchestrates the entire workflow with intelligent auto-continuation
2. **Auto-Context Pipeline** dynamically loads relevant code for context
3. **Planning Phase** breaks down tasks into manageable subtasks
4. **Implementation Phase** executes tasks one at a time with smart context filtering
5. **Build & Validation** ensures code quality through multi-attempt validation
6. **Apply & Execute** safely applies changes and runs commands
7. **Chat Mode** provides conversational assistance without file modifications

Each pipeline is carefully designed with specific prompts, data flows, and error handling to ensure reliable, high-quality code generation.

