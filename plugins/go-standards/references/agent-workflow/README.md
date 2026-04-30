# Multi-Agent Development Workflow

## Overview

AI-assisted development workflow separating **intelligent planning** (Opus) from **mechanical execution** (Haiku).

**CRITICAL**: All operations MUST happen via Task tool agents. The main context (Opus) is ONLY for:
- Reading user requests
- Launching appropriate agents via Task tool
- Reviewing agent results
- Communicating with the user

## Core Principle

```
┌─────────────────────────────────────────────────────────────────┐
│                    MAIN CONTEXT (Opus)                      │
│                                                                 │
│  - Receives user requests                                       │
│  - Launches agents via Task tool                                │
│  - Reviews results                                              │
│  - NEVER writes code directly                                   │
│  - NEVER runs tests directly                                    │
│  - NEVER fixes errors directly                                  │
│                                                                 │
│  "I orchestrate, agents execute"                                │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    AGENTS (via Task tool)                       │
│                                                                 │
│  Planning Agent (Opus) ──► Writes code in story                 │
│  Implementation Agent (Haiku) ──► Copies code to files          │
│  Fix Agent (Sonnet/Opus) ──► Fixes errors in story              │
│  Code Review Agent (Haiku) ──► Verifies implementation          │
│                                                                 │
│  All communication between agents: via story file               │
└─────────────────────────────────────────────────────────────────┘
```

## Agent Types

| Agent | Model | Purpose | Input | Output |
|-------|-------|---------|-------|--------|
| **Planning** | Opus | Write complete code in story | Story (draft) | Story (ready) |
| **Implementation** | Haiku | Copy code from story to files | Story (ready) | Files + Story (review/in_progress) |
| **Fix** | Sonnet/Opus | Fix errors in story code | Story (in_progress + errors) | Story (ready) |
| **Code Review** | Haiku | Verify implementation | Story (review) | Story (done) or issues |

## Complete Workflow

```
┌──────────────────┐
│   USER           │
│   Creates story  │
│   (draft)        │
└────────┬─────────┘
         │
         ▼
┌──────────────────────────────────────┐
│   PLANNING AGENT (Opus via Task)     │  ← Task tool invocation
│   Writes complete code in story      │     model: "opus"
│   Status: draft → ready              │
└────────┬─────────────────────────────┘
         │
         ▼
┌──────────────────┐
│   USER REVIEW    │
│   Reviews code   │
│   in story file  │
└────────┬─────────┘
         │
         ▼
┌──────────────────────────────────────┐
│   IMPLEMENTATION AGENT (Haiku)       │  ← Task tool invocation
│   Copies code from story to files    │     model: "haiku"
│   Runs verification                  │
│   Status: ready → in_progress/review │
└────────┬─────────────────────────────┘
         │
         ├── If PASS ─────────────────────┐
         │                                │
         ▼                                │
┌────────────────────────┐                │
│   If FAIL              │                │
│   Errors in story      │                │
└────────┬───────────────┘                │
         │                                │
         ▼                                │
┌──────────────────────────────────────┐  │
│   FIX AGENT (Sonnet/Opus via Task)   │  │  ← Task tool invocation
│   Reads errors from story            │  │     model: "sonnet" or "opus"
│   Fixes code IN STORY FILE           │  │
│   Status: in_progress → ready        │  │
└────────┬─────────────────────────────┘  │
         │                                │
         └── Loop back to Implementation ─┘
                                          │
                                          ▼
┌──────────────────────────────────────┐
│   CODE REVIEW AGENT (Haiku via Task) │  ← Task tool invocation
│   Verifies files match story         │     model: "haiku"
│   Status: review → done              │
└────────┬─────────────────────────────┘
         │
         ├── APPROVED → done
         │
         └── NEEDS_CHANGES → Fix Agent → Implementation Agent
```

## Critical Rules

### 1. Main Context NEVER Does Work

```
❌ WRONG (in main context):
- Reading code files to fix errors
- Writing/editing actual code files
- Running go test or golangci-lint
- Making code changes based on errors

✓ RIGHT (in main context):
- Launch Planning Agent to write code
- Launch Implementation Agent to copy code
- Launch Fix Agent when errors occur
- Launch Code Review Agent to verify
```

### 2. All Communication Via Story File

Agents don't communicate directly. They read/write the story file:

```
Planning Agent ──writes──► Story File ──read by──► Implementation Agent
                               │
Implementation Agent ──writes errors──► Story File ──read by──► Fix Agent
                               │
Fix Agent ──writes fixes──► Story File ──read by──► Implementation Agent
```

### 3. Fix Agent vs Main Context

**When Implementation Agent reports errors:**

```
❌ WRONG:
Main context reads errors and starts editing code directly

✓ RIGHT:
Main context launches Fix Agent via Task tool
Fix Agent reads errors from story
Fix Agent fixes code IN THE STORY FILE
Main context launches Implementation Agent again
```

### 4. Session Isolation

Each agent runs in its own context (Task tool). This ensures:
- Clean context for each operation
- No pollution of main context
- Clear separation of concerns
- Predictable behavior

## Task Tool Invocations

### Planning Agent

```
Task tool call:
- subagent_type: "systems-programming:golang-pro"
- model: "opus"
- prompt: |
    You are a PLANNING AGENT for story NNN-feature-name.

    YOUR JOB: Write COMPLETE, PRODUCTION-READY code in the story file.

    PROCESS:
    1. Read story: SERVICE_PATH/documentation/stories/NNN-feature-name.md
    2. Explore codebase to understand patterns
    3. Write complete code for each file in story
    4. Write complete tests for each file
    5. Set status to "ready"

    RULES:
    - Write COMPLETE code, no TODOs
    - Include ALL imports
    - Write ALL tests
    - Follow Clean Architecture
    - Do NOT create actual files

    Work directory: SERVICE_PATH/
```

### Implementation Agent

```
Task tool call:
- subagent_type: "systems-programming:golang-pro"
- model: "haiku"
- prompt: |
    You are an IMPLEMENTATION AGENT for story NNN-feature-name.

    YOUR ONLY JOB: Copy code from story to files.

    RULES:
    - DO NOT modify code
    - DO NOT add anything
    - DO NOT fix errors
    - ONLY copy code blocks to file paths

    PROCESS:
    1. Read story: SERVICE_PATH/documentation/stories/NNN-feature-name.md
    2. For each "#### File: `path`" - copy code block to that path
    3. Run: golangci-lint run ./... && go test ./...
    4. If PASS: set status to "review"
    5. If FAIL: write errors to "Issues Found", keep status "in_progress"

    Work directory: SERVICE_PATH/
```

### Fix Agent

```
Task tool call:
- subagent_type: "systems-programming:golang-pro"
- model: "sonnet"
- prompt: |
    You are a FIX AGENT for story NNN-feature-name.

    Implementation failed with errors. Your job:
    1. Read story: SERVICE_PATH/documentation/stories/NNN-feature-name.md
    2. Find "Issues Found" section
    3. Fix ALL errors in the code blocks IN THE STORY FILE
    4. Document fixes in "Fixes Applied" section
    5. Set status to "ready"

    RULES:
    - ONLY edit story file, NOT actual code files
    - Fix ALL errors
    - Do NOT add new features
    - Do NOT run tests

    Work directory: SERVICE_PATH/
```

### Code Review Agent

```
Task tool call:
- subagent_type: "code-documentation:code-reviewer"
- model: "haiku"
- prompt: |
    Review implementation for story NNN-feature-name.

    VERIFY:
    1. All files from story exist
    2. Contents match story exactly
    3. Tests pass
    4. Coverage meets thresholds

    OUTPUT:
    - APPROVED: Set status to "done"
    - NEEDS_CHANGES: List issues
```

## Error Handling Flow

When Implementation Agent fails:

```
┌─────────────────────────────────────────┐
│ Implementation Agent reports:           │
│ "Verification FAILED"                   │
│ "Issues documented in story"            │
└─────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────┐
│ Main Context sees failure               │
│                                         │
│ ❌ WRONG: Start fixing code directly    │
│ ✓ RIGHT: Launch Fix Agent               │
└─────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────┐
│ Fix Agent (via Task tool):              │
│ - Reads errors from story               │
│ - Fixes code IN STORY FILE              │
│ - Sets status to "ready"                │
└─────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────┐
│ Main Context launches Implementation    │
│ Agent again (new Task tool call)        │
└─────────────────────────────────────────┘
```

## Directory Structure

```
service/
└── documentation/
    └── stories/
        ├── 001-initial-setup.md
        ├── 002-add-user-auth.md
        └── ...
```

## Story Lifecycle

| Status | Meaning | Set By |
|--------|---------|--------|
| `draft` | Story created, needs planning | User |
| `planning` | Planning Agent writing code | Planning Agent |
| `ready` | Code in story, awaiting implementation | Planning/Fix Agent |
| `in_progress` | Implementation failed, needs fix | Implementation Agent |
| `review` | Implementation passed, awaiting review | Implementation Agent |
| `done` | Complete | Code Review Agent |

## Quick Reference

| When this happens... | Do this... |
|----------------------|------------|
| User creates story (draft) | Launch Planning Agent |
| Planning complete (ready) | User reviews, then Launch Implementation Agent |
| Implementation passes (review) | Launch Code Review Agent |
| Implementation fails (in_progress) | Launch Fix Agent |
| Fix complete (ready) | Launch Implementation Agent again |
| Code review passes (done) | Story complete |
| Code review fails | Launch Fix Agent |

## Anti-Patterns to Avoid

### DON'T: Fix Errors in Main Context

```
❌ "I see the error. Let me fix this import..."
   [Main context starts editing files]

✓ "Implementation failed. Launching Fix Agent..."
   [Task tool call to Fix Agent]
```

### DON'T: Run Tests in Main Context

```
❌ "Let me run the tests to see what's failing..."
   [Main context runs go test]

✓ "Implementation Agent will run tests and report results."
   [Task tool call to Implementation Agent]
```

### DON'T: Skip the Story File

```
❌ "The fix is simple, I'll just edit the file directly..."
   [Main context edits actual code files]

✓ "Fix Agent will update the story, then Implementation Agent will copy."
   [Task tool calls preserve the workflow]
```

## Related Documentation

- [Story Template](story-template.md) - Story file format
- [Planning Guide](planning-guide.md) - Planning Agent instructions
- [Implementation Guide](implementation-guide.md) - Implementation Agent instructions
- [Fix Guide](fix-guide.md) - Fix Agent instructions

## Summary

| Role | Context | Model | Does |
|------|---------|-------|------|
| Orchestrator | Main | Opus | Launches agents, reviews results |
| Planner | Task | Opus | Writes code in story |
| Implementer | Task | Haiku | Copies code to files |
| Fixer | Task | Sonnet/Opus | Fixes errors in story |
| Reviewer | Task | Haiku | Verifies implementation |

**All work happens in agents. Main context only orchestrates.**
