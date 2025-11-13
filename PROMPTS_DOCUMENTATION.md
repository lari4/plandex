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

---

## 4. Context Management Prompts

### GetArchitectContextSummary

**Purpose:** Instructions for the architect role to assess and load relevant context.

**Location:** `app/server/model/prompts/architect_context.go`

**Function Signature:** `GetArchitectContextSummary(tokenLimit int) string`

**Description:** Two-phase process for context management:

**Phase 1 - Context (Current Phase):**
- Examine user's request and available codebase information
- Determine relevant context for next phase
- List categories and files needed
- End with `<PlandexFinish/>`

**Phase 2 - Response (Next Phase):**
- System incorporates selected context
- Create plan (tell mode) or provide answer (chat mode)
- Implementation happens only in Phase 2

**Key Instructions:**
1. Assess if have enough information (if not, ask user)
2. Create high-level architectural overview/plan
3. Output context sections if needed:
   - `### Categories` - list of context categories
   - `### Files` - grouped by category with relevant symbols
   - Files MUST be from codebase map or pending changes
   - Order files by importance/relevance
4. Output `<PlandexFinish/>` immediately after

**Critical Rules:**
- Do NOT write any code or implementation
- Do NOT create tasks or plans
- ONLY include files from codebase map or pending changes
- Never guess file paths or include hypothetical files

### GetAutoContextTellPrompt

**Purpose:** Tell mode context loading instructions for auto-context phase.

**Location:** `app/server/model/prompts/architect_context.go`

**Function Signature:** `GetAutoContextTellPrompt(params CreatePromptParams) string`

**Description:** Instructions for loading context automatically in tell mode:
- Create brief high-level overview (NOT final plan)
- Adapt length to project size and complexity
- Continue with context loading in same response
- Output categories and files if context needed
- Follow interface/implementation, API chain, database chain rules

**Context Loading Rules:**
1. **Interface & Implementation Rule:** Load both interface and implementation files
2. **Reference Implementation Rule:** Load existing similar features as reference
3. **API Client Chain Rule:** Load API interface + client implementation
4. **Database Chain Rule:** Load model files + helpers + similar DB operations
5. **Utility Dependencies Rule:** Load ALL utility files that might be needed

**Finding Relevant Context:**
- Look for naming patterns (similar prefixes/suffixes)
- Find feature groupings (all related files)
- Follow file relationships (interface, test, helper files)

### GetAutoContextChatPrompt

**Purpose:** Chat mode context loading instructions for auto-context phase.

**Location:** `app/server/model/prompts/architect_context.go`

**Function Signature:** `GetAutoContextChatPrompt(params CreatePromptParams) string`

**Description:** Instructions for assessing and loading context in chat mode:
- Assess which context is relevant to user's question/message
- Be eager about loading context (if in doubt, load it)
- Mention what you need to check naturally
- Output categories and files in same format as tell mode

**When to Load:**
- Specific files needed from codebase map
- Related files would help answer
- Need to understand implementations or dependencies

**Output Format:**
```
### Categories
- Category 1
- Category 2

### Files
Category 1:
`file1.go` - symbol1, symbol2, symbol3 (annotation)
`file2.go` - symbol1, symbol2

Category 2:
`file3.js` - symbol1, symbol2

<PlandexFinish/>
```

---

## 5. Chat Mode Prompts

### GetChatSysPrompt

**Purpose:** System prompt for chat-only mode interactions.

**Location:** `app/server/model/prompts/chat.go`

**Function Signature:** `GetChatSysPrompt(params CreatePromptParams) string`

**Description:** Defines behavior for chat mode where AI discusses code without making changes.

**Key Capabilities:**
- Engage in technical discussion about code
- Provide explanations and answer questions
- Include code snippets for explanation
- Reference files from context
- Help debug issues
- Suggest approaches and discuss trade-offs
- Evaluate implementation strategies

**Cannot Do:**
- Create or modify any files
- Output formal implementation code blocks
- Make formal plans using "### Tasks"
- Load context multiple times consecutively
- Switch to implementation mode without user request

**Execution Mode Handling:**
- If enabled: Can discuss both file changes and command execution
- If disabled: Focus on file updates, mention execution mode needed for commands

**Context Handling:**
- Auto-context: Continue conversation with loaded context
- Manual: Ask user to provide needed files

**Transition to Tell Mode:**
- Use exact phrase "switch to tell mode" to give user option
- Remind users they can use tell mode for actual changes

---

## 6. Code Formatting Prompts

### ChangeExplanationPrompt

**Purpose:** Defines the Action Explanation Format for all code changes.

**Location:** `app/server/model/prompts/explanation_format.go`

**Description:** Mandatory format that MUST precede every code block.

**Format for Updating Existing Files:**
```
**Updating `[file path]`**
Type: [add|prepend|append|replace|remove|overwrite]
Summary: [brief description]
Replace: [lines to replace/remove] (for replace/remove only)
Context: [surrounding code structures]
Preserve: [symbols to preserve] (for overwrite only)
```

**Type Definitions:**
- **add:** Insert new code within file (NO existing code changed/removed)
- **prepend:** Insert at start of file
- **append:** Insert at end of file
- **replace:** Replace existing code with new code
- **remove:** Remove existing code with nothing new
- **overwrite:** Replace entire file

**Multiple Changes to Same File:**
```
**Updating `file.go`**
Change 1.
  Type: remove
  Summary: Remove unused functions
  Replace: lines 25-85
  Context: Located between `funcA` and `funcB`

Change 2.
  Type: append
  Summary: Add new function at end
  Context: Last structure is `finalFunc`
```

**Format for Creating New Files:**
```
**Creating `[file path]`**
Type: new file
Summary: [brief description of new file]
```

**Critical Context Field Rules:**
- Symbols/structures in Context MUST NOT be modified
- They serve as ANCHORS to locate the change
- All Context symbols MUST appear in code block
- Anchors MUST be immediately adjacent to change
- Failure to include Context symbols = CRITICAL ERROR

**Symbol Format:**
- List symbols in comma-separated list with backticks
- Use ONLY symbol name, NOT full signature
- Example: `funcName` not `func funcName(params) returns`

### UpdateFormatPrompt

**Purpose:** Detailed instructions for updating existing files with reference comments.

**Location:** `app/server/model/prompts/update_format.go`

**Description:** Comprehensive guide on using reference comments and updating files correctly.

**Reference Comment Rules:**
- ONLY use `// ... existing code ...` (with appropriate comment symbol)
- NEVER use any other variations
- Must be on their own lines (never inline)
- Must match indentation of original file
- Must include trailing commas/syntax where necessary
- Must not have multiple references in a row with no code between

**For File Types Without Comments (JSON, etc.):**
- Still use `// ... existing code ...`
- Required for proper structure demonstration

**Removal Comments:**
- Use `// Plandex: removed code` (with appropriate comment symbol)
- Replaces the code being removed
- Do NOT include the removed code in block
- Must be on own line with sufficient context

**Critical Rules:**
- Show ONLY changing code + necessary context
- Include enough anchors to locate change unambiguously
- Match original indentation EXACTLY
- Preserve code order from original file
- ALWAYS include `// ... existing code ...` at start (unless prepending)
- ALWAYS include `// ... existing code ...` at end (unless appending)

**Anchor Requirements:**
When inserting code, must include:
- Anchors from original file immediately before change
- Anchors from original file immediately after change
- Enough structure to show nesting level clearly

### UpdateFormatAdditionalExamples

**Purpose:** Extended examples of correct vs incorrect update formatting.

**Location:** `app/server/model/prompts/update_format.go`

**Description:** Provides concrete examples of:
1. Adding new route (incorrect: replacing instead of inserting)
2. Adding method to class (incorrect: ambiguous insertion)
3. Adding configuration section (incorrect: lost context)
4. Multiple changes to same file (incorrect: multiple blocks vs correct: single block)

**Key Lessons:**
- Always show surrounding context that will be preserved
- Make insertion points unambiguous by showing adjacent code
- Never remove existing functionality unless explicitly instructed
- Use reference comments properly to indicate preserved sections
- Show enough context to understand code structure

---

## 7. File Operations Prompts

### FileOpsPlanningPrompt

**Purpose:** Instructions for planning file operations (move, remove, reset).

**Location:** `app/server/model/prompts/file_ops.go`

**Description:** Explains special subtasks for file operations on files in context or with pending changes.

**Available Operations:**
- **Move:** Relocate or rename files
- **Remove:** Delete files
- **Reset:** Clear pending changes for files

**Important Notes:**
1. Can ONLY operate on files in context or with pending changes
2. Cannot create new directories (created automatically)
3. Cannot move file to path already in context/pending (would overwrite)
4. After move: further updates go to NEW location
5. After remove: further updates require creating NEW file

**When to Use:**
- Usually NOT used in initial plan implementation
- Mainly for revising plans with pending changes
- User specifically asks to move/remove files

### FileOpsImplementationPrompt

**Purpose:** Detailed format and rules for implementing file operations.

**Location:** `app/server/model/prompts/file_ops.go`

**Description:** Defines exact syntax for three types of file operation sections.

**Move Files Section:**
```
### Move Files
- `source/path.tsx` → `dest/path.tsx`
- `components/button.tsx` → `pages/button.tsx`
<EndPlandexFileOps/>
```

**Rules:**
- Each line starts with dash (-)
- Paths wrapped in backticks
- Use → (Unicode arrow, NOT ->)
- Can only move individual files (not directories)
- Source MUST be in context or pending
- Dest MUST NOT already exist
- Dest directory created automatically if needed

**Remove Files Section:**
```
### Remove Files
- `components/page.tsx`
- `layouts/header.tsx`
<EndPlandexFileOps/>
```

**Rules:**
- Each line starts with dash (-)
- Paths wrapped in backticks
- Can only remove individual files
- All paths MUST be in context or pending

**Reset Changes Section:**
```
### Reset Changes
- `components/page.tsx`
- `layouts/header.tsx`
<EndPlandexFileOps/>
```

**Rules:**
- Each line starts with dash (-)
- Paths wrapped in backticks
- Can only reset files WITH pending changes
- Clears pending changes, returns to original state

**All Sections Must:**
- End with `<EndPlandexFileOps/>` tag
- Follow exact format (no comments/extra text)
- Only operate on files in context/pending

---

## 8. Execution & Script Prompts

### ApplyScriptSharedPrompt

**Purpose:** Core instructions for the `_apply.sh` script and command execution.

**Location:** `app/server/model/prompts/apply_exec.go`

**Description:** Comprehensive guide for using the special `_apply.sh` file to execute commands on user's machine.

**Core Concepts:**
- Executed EXACTLY ONCE after ALL files created/updated
- Script accumulates commands during plan
- Resets to empty after successful execution
- Runs in root directory of plan

**Core Restrictions - MUST NOT:**
- Create files/directories (use code blocks instead)
- Use for file operations on files in context
- Include shebang lines or error handling (handled externally)
- Give script execution privileges (handled externally)
- Tell users to run script (runs automatically)
- Use separate script files unless absolutely necessary

**Safety & Security:**
- Only make strictly necessary changes
- Prefer local over global changes (npm install --save-dev vs --global)
- Only modify files/directories in plan root directory
- Avoid user prompts (make reasonable defaults)
- Don't run malicious/harmful commands

**Keep It Lightweight:**
- Script should be simple and focused
- Offload complex logic to separate files
- Don't include large content blocks
- Use straightforward bash (avoid fancy constructs)
- Make portable across Unix-like systems

**Dependencies and Tools:**
1. **Context-based Assumptions:**
   - Make reasonable assumptions based on OS, files, project structure
   - Don't install Node.js for existing Node project
   - Don't install Go for Go project, Python for Python project, etc.

2. **Checking for Tools:**
   - Check if tool installed before using
   - Install missing tools or exit with clear error

3. **Dependency Management:**
   - Don't install dependencies already in project
   - Only install new dependencies for new features

**Browser & Web Server Commands:**
- Launch browser with `plandex browser [urls...]` command
- CRITICAL: Run server in background with `&` before launching browser
- Add sleep to allow server startup
- Handle cleanup with `||` block for browser command failures
- Use uncommon ports (3400+) to avoid conflicts
- Try multiple ports if default is in use

**Command Preservation:**
- ALL commands persist until successful application
- Each update ADDS/MODIFIES, never removes
- After success, script resets to empty
- History of previous scripts provided in prompt

### ApplyScriptPlanningPrompt

**Purpose:** Planning-specific instructions for _apply.sh during task breakdown.

**Location:** `app/server/model/prompts/apply_exec.go`

**Description:** Guides how to plan tasks involving command execution.

**Natural Command Hierarchy:**
1. Install required packages/dependencies
2. Run build commands
3. Run test/execution commands

**Good Practices:**
- Write dependency installations close to related subtasks
- Group related commands together
- Commands affecting whole project appear only ONCE
- Update existing commands rather than duplicate

**Bad Practices to Avoid:**
- Don't write same command multiple times
- Don't create separate subtasks for single commands
- Don't duplicate installations

**Example Good Structure:**
```
1. Add authentication feature
   - Update auth-related files
   - Write to _apply.sh: npm install auth-package

2. Build and run
   - Write to _apply.sh: npm run build, npm start
```

**Critical Requirement:**
IMMEDIATELY BEFORE '### Tasks', MUST output '### Commands' section assessing whether commands needed in _apply.sh.

**### Commands Section Format:**
```
### Commands

The _apply.sh script is empty. I'll add commands to build and run the code.
I'll add this step to the plan.

### Tasks
[... tasks including _apply.sh subtask ...]
```

### ApplyScriptImplementationPrompt

**Purpose:** Implementation-specific instructions for writing to _apply.sh.

**Location:** `app/server/model/prompts/apply_exec.go`

**Description:** Detailed guide for actually implementing _apply.sh updates.

**Code Block Format:**
```
- _apply.sh:
<PlandexBlock lang="bash" path="_apply.sh">
# commands here
</PlandexBlock>
```

**CRITICAL Rules:**
- ALWAYS include file path label exactly as shown
- ALWAYS use `lang="bash"` in <PlandexBlock> tag
- NO lines between path and opening tag
- If empty: follow "creating new file" instructions
- If not empty: follow "updating existing file" instructions

**Command Output:**
- DO NOT hide or filter command output
- Show all command output (don't use --quiet, etc.)
- Let commands display their natural output

**Script Organization:**
- Written defensively to fail gracefully
- Organized logically with similar commands grouped
- Commented only when necessary
- Include logging ONLY for errors and long operations

**Tool Checking Example:**
```bash
if ! command -v tool > /dev/null; then
    echo "Error: tool is not installed"
    exit 1
fi
```

**Dependency Installation Example:**
```bash
npm install --save-dev \
    package1 \
    package2 \
    package3
```

**Web Server with Browser Example:**
```bash
# Find available port
export PORT=3400
while ! nc -z localhost $PORT && [ $PORT -lt 3410 ]; do
  export PORT=$((PORT + 1))
done

# Build and start in background
npm run build
npm start &
SERVER_PID=$!

# Wait for server
sleep 3

# Launch browser
plandex browser http://localhost:$PORT || {
   kill $SERVER_PID
   exit 1
}
wait $SERVER_PID
```

**Process Management:**
- If running multiple processes, handle partial failures
- Store PIDs, wait on all processes
- If any fail, kill all and exit with error code
- No need for `trap`, `setsid`, `disown` (handled by wrapper)

---

## 9. Naming Prompts

### SysPlanName / SysPlanNameXml

**Purpose:** Generate short lowercase names for plans.

**Location:** `app/server/model/prompts/name.go`

**Description:** AI creates concise plan names for the content.

**Format:**
- Short lowercase file name
- Use dashes as word separators
- No spaces, numbers, or special characters
- 2-3 words max (1-2 if possible)
- Shorten and abbreviate where possible

**Example:** `add-auth-system`

**Output Formats:**
- **XML:** `<planName>add-auth-system</planName>`
- **JSON Function:** Call `namePlan` with `{"planName": "add-auth-system"}`

### SysPipedDataName / SysPipedDataNameXml

**Purpose:** Generate names for piped command output.

**Location:** `app/server/model/prompts/name.go`

**Description:** Create names for data piped into context. Try to guess what command produced it.

**Format:** Same as plan names (short, lowercase, dashes)

**Example:** `git-status`

**Output Formats:**
- **XML:** `<name>git-status</name>`
- **JSON Function:** Call `namePipedData` with `{"name": "git-status"}`

### SysNoteName / SysNoteNameXml

**Purpose:** Generate names for arbitrary text notes.

**Location:** `app/server/model/prompts/name.go`

**Description:** Create names for text notes added to context.

**Format:** Same as plan names (short, lowercase, dashes)

**Example:** `meeting-notes`

**Output Formats:**
- **XML:** `<name>meeting-notes</name>`
- **JSON Function:** Call `nameNote` with `{"name": "meeting-notes"}`

---

## 10. Summary & Description Prompts

### PlanSummary

**Purpose:** Summarize the entire conversation and plan state.

**Location:** `app/server/model/prompts/summary.go`

**Description:** Creates append-only summary of plan evolution over time.

**Structure:**
1. **User's Messages Summary:**
   - Focus on tasks given
   - Reflect latest version of each task
   - Omit obsolete information
   - Condense while retaining meaning

2. **Discussion & Accomplishments:**
   - Key decisions made
   - Major changes or updates
   - Significant challenges identified
   - Important requirements established

3. **Latest Messages & Next Steps:**
   - What's been done recently
   - Next steps discussed
   - Action items identified

**Key Principles:**
- Do NOT include code in summary
- Explain in words what was done and what needs doing
- Treat as append-only record
- Keep information from existing summary
- Add new information from latest messages

### SysDescribe / SysDescribeXml

**Purpose:** Generate commit messages from plans.

**Location:** `app/server/model/prompts/describe.go`

**Description:** Turn AI's plan into structured commit message.

**Output Format:**
- **XML:** `<commitMsg>Add user authentication system with JWT support</commitMsg>`
- **JSON Function:** Call `describePlan` with `{"commitMsg": "..."}`

**Qualities:**
- Good, succinct commit message
- Describes proposed changes clearly
- Follows conventional commit style

### SysPendingResults

**Purpose:** Summarize pending changes into one-line commit message.

**Location:** `app/server/model/prompts/describe.go`

**Description:** Takes list of pending change descriptions and creates one-line summary suitable for commit title.

**Output:** ONLY the one-line title, nothing else.

---

## 11. Validation Prompts

### GetValidationReplacementsXmlPrompt

**Purpose:** Validate if changes were applied correctly and fix if needed.

**Location:** `app/server/model/prompts/build_validation_replacements.go`

**Function Signature:** `GetValidationReplacementsXmlPrompt(params ValidationPromptParams) (string, int)`

**Description:** Checks if proposed changes were correctly applied to original file.

**Input Provided:**
- Original file with line numbers (prefixed `pdx-`)
- Proposed changes explanation
- Proposed changes with line numbers (prefixed `pdx-new-`)
- Diff of applied changes
- Reasons for verification (ambiguous location, code removed, duplicated)
- Syntax errors (if any)

**Validation Criteria - ONLY assess:**
- Changes applied at correct location and nesting
- All specified additions/modifications included
- No unintended changes to surrounding code
- No accidentally removed or duplicated code
- Syntax errors (if previously specified)

**Do NOT evaluate:**
- Code quality
- Missing imports
- Unused variables
- Best practices
- Potential bugs

**Output Format:**

**If Correct:**
```
## Evaluate Diff
[reasoning about why changes are correct]

<PlandexCorrect/>
<PlandexFinish/>
```

**If Incorrect:**
```
## Evaluate Diff
[reasoning about what went wrong]

<PlandexIncorrect/>

<PlandexComments>
[classify each comment as reference or not]
</PlandexComments>

<PlandexReplacements>
  <Replacement>
    <Old>
      pdx-42: func someFunction() {
      pdx-43:   code here
      pdx-44: }
    </Old>
    <New>
      func someFunction() {
        corrected code here
      }
    </New>
  </Replacement>
</PlandexReplacements>
```

**Critical Rules:**
- `<Old>` must include exact original code with `pdx-` line numbers
- `<New>` must contain corrected code WITHOUT line numbers
- NO reference comments in `<New>` (replace with actual code)
- Replacements ordered by position in file
- NO overlapping replacements

### CommentClassifierPrompt

**Purpose:** Evaluate which comments are reference comments vs actual code comments.

**Location:** `app/server/model/prompts/build_helpers.go`

**Description:** Analyzes comments in proposed updates to classify them.

**Reference Comment Examples:**
- `// ... existing code...`
- `# Existing code...`
- `/* ... */`
- `// rest of the function...`
- `<!-- rest of div tag -->`

**Output Format:**
```
<PlandexComments>
pdx-new-1: // ... existing code ...
Evaluation: refers to code that starts transaction
Reference: true

pdx-new-5: // verify user permission
Evaluation: describes the change being made
Reference: false
</PlandexComments>
```

**Rules:**
- List EVERY comment from proposed updates
- Include line number with `pdx-new-` prefix
- Evaluate if refers to code left out of updates
- State whether it's a reference comment
- Reference comments can exist in non-comment file types (JSON, etc.)

### WholeFilePrompt

**Purpose:** Output entire merged file with all changes applied.

**Location:** `app/server/model/prompts/build_whole_file.go`

**Description:** Merge original file with proposed updates to create complete result.

**Key Instructions:**
- Output ENTIRE merged file
- Replace ALL reference comments with actual code from original
- File must be syntactically and semantically correct
- All code structures properly balanced
- Use `<PlandexWholeFile>` element

**Output Format:**
```
<PlandexWholeFile>
package main

import "logger"

function main() {
  logger.info("Hello, world!");
  exec()
}
</PlandexWholeFile>
```

**Critical Rules:**
- NO line numbers in output
- NO reference comments in output
- Output ENTIRE file no matter how long
- No other text except file content
- No triple backticks, only `<PlandexWholeFile>` tags
- Do NOT remove/change code not in proposed updates
- End after `</PlandexWholeFile>`, no text after

---

## 12. Execution Status Prompts

### SysExecStatusFinishedSubtask / SysExecStatusFinishedSubtaskXml

**Purpose:** Evaluate if a subtask was fully implemented.

**Location:** `app/server/model/prompts/exec_status.go`

**Description:** Analyzes AI's implementation messages to determine if current task is complete.

**Evaluation Criteria:**
- Examine current message and possibly previous messages
- Task only complete if ALL necessary code changes implemented
- No remaining TODO placeholders
- No partial implementations

**Consider:**
- If AI stated task complete, but also check actual implementation
- Don't validate code quality, only completeness
- Task finished if all items implemented (even if imperfect)
- Task NOT finished only if significant step missing

**Output Format:**

**XML:**
```
<subtaskStatus>
<reasoning>Task is complete - all required code changes implemented with no placeholders</reasoning>
<subtaskFinished>true</subtaskFinished>
</subtaskStatus>
```

**JSON Function:** Call `didFinishSubtask` with:
```json
{
  "reasoning": "Task is complete - all required code changes...",
  "subtaskFinished": true
}
```

**Subtask States:**
- `true`: All necessary code changes completed, no placeholders
- `false`: Significant steps missing or unexplained placeholders remain

---

## 13. Missing File Prompts

### GetSkipMissingFilePrompt

**Purpose:** Instruct AI to skip generating specific file.

**Location:** `app/server/model/prompts/missing_file.go`

**Function Signature:** `GetSkipMissingFilePrompt(path string) string`

**Description:** Tells AI to not generate content for specified file and continue with plan.

**Behavior:**
- Skip this file entirely
- Continue with remaining tasks/subtasks
- Don't repeat previous message content
- If no remaining tasks, stop there

**Example:** "You *must not* generate content for the file `config.json`. Skip this file and continue..."

### GetMissingFileContinueGeneratingPrompt

**Purpose:** Continue generating large file across multiple responses.

**Location:** `app/server/model/prompts/missing_file.go`

**Function Signature:** `GetMissingFileContinueGeneratingPrompt(path string) string`

**Description:** For large files that exceed single response, continue from where left off.

**Critical Rules:**
- Continue EXACTLY where previous message left off
- Do NOT produce any other output before continuing
- Do NOT repeat previous message content
- Do NOT duplicate last line before continuing
- Do NOT include opening `<PlandexBlock>` tag (already included)
- Start immediately with code that belongs in file
- MUST include closing `</PlandexBlock>` tag when done
- Then continue with plan if remaining tasks

**Example Instruction:** "Continue generating the file 'large_file.py'. Continue EXACTLY where you left off. DO NOT INCLUDE THE FILE PATH OR OPENING TAG..."

---

## Summary

This documentation covers all major AI prompts used in Plandex, organized by their functional purpose:

1. **System Identity** - Core AI role definition
2. **Planning Phase** - Breaking down tasks into subtasks
3. **Implementation Phase** - Writing code to complete tasks
4. **Context Management** - Loading relevant code files
5. **Chat Mode** - Conversational mode without file changes
6. **Code Formatting** - Exact format for code changes and explanations
7. **File Operations** - Moving, removing, resetting files
8. **Execution & Scripts** - Running commands via _apply.sh
9. **Naming** - Generating names for plans, data, notes
10. **Summary & Description** - Creating summaries and commit messages
11. **Validation** - Verifying changes applied correctly
12. **Execution Status** - Checking task completion
13. **Missing Files** - Handling skipped or large files

Each prompt is carefully designed to guide the AI through specific phases of the plan-code-apply workflow, ensuring consistent, high-quality output while maintaining safety and user control.

