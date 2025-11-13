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

