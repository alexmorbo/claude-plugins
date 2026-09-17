# Multi-Agent Development Workflow

## Overview

AI-assisted development workflow separating **deciding** (Opus) from
**building** (Sonnet), with verification wired into both.

The story file carries intent and contracts. The repository carries the code.
Neither duplicates the other.

## Core Principle

```
┌─────────────────────────────────────────────────────────────────┐
│                    THE STORY IS A CONTRACT                      │
│                                                                 │
│  Planning decides:  scope, signatures, schemas, invariants,     │
│                     acceptance criteria, verification commands  │
│                                                                 │
│  Implementation:    writes bodies and tests in the repo,        │
│                     compiles, lints, tests, fixes what breaks   │
│                                                                 │
│  Code that never compiled is not a plan - it is a guess.        │
│  A story describing code that does not exist is worse than      │
│  no story at all.                                               │
└─────────────────────────────────────────────────────────────────┘
```

## Why Not Put the Code in the Story

The older version of this workflow had planning write complete, production-ready
code into the story, with a cheap model copying it into files. That is no longer
the recommended shape, for four independent reasons:

1. **No feedback.** Code written into Markdown never meets `go build`. Planning
   improves outcomes when it encodes feedback; a plan of untested code encodes
   none - and Go's compiler is the cheapest, most precise reviewer available.
2. **Drift.** Implementation adjusts the code to make things pass. The story is
   not adjusted. What is left is a confident document describing a codebase that
   does not exist.
3. **Reviewability.** The plan is the high-leverage review surface precisely
   because it is shorter than the code. A plan containing the whole
   implementation is longer than the code *and* has no tooling behind it.
4. **The cheap tier is not a transcriber.** A smaller model today is a full
   agent with tools and a verification loop. Using it to copy Markdown wastes
   what it is actually good at and adds a parsing step that can silently corrupt
   the result.

## Agent Roles

| Agent | Tier | Purpose | Input | Output |
|-------|------|---------|-------|--------|
| **Planning** | Opus | Write scope, contracts, acceptance criteria | Story (draft) | Story (ready) |
| **Implementation** | Sonnet | Write code in repo, verify, fix | Story (ready) | Code + Story (review) |
| **Mechanical edits** | Haiku | Single-file edits, codemods, renames | Explicit instruction | Code |
| **Review** | fresh context | Verify diff against acceptance criteria | Story (review) + diff | Story (done) or issues |
| **Reconcile** | Opus | Resolve story/code divergence | Diverged story + repo | Story (ready/done) |

Review is never performed by the agent that wrote the code - an author
reviewing itself is reliably too generous.

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
│   PLANNING AGENT (Opus)              │
│   Scope, contracts, acceptance,      │
│   steps, verification commands       │
│   Status: draft → ready              │
└────────┬─────────────────────────────┘
         │
         ▼
┌──────────────────┐
│   USER REVIEW    │
│   Reviews the    │
│   contracts      │
└────────┬─────────┘
         │
         ▼
┌──────────────────────────────────────┐
│   IMPLEMENTATION AGENT (Sonnet)      │
│   Writes code in the repository      │
│   ┌────────────────────────────────┐ │
│   │ edit → build → lint → test     │ │
│   │   ▲                      │     │ │
│   │   └──── fix ◄────────────┘     │ │
│   └────────────────────────────────┘ │
│   Status: ready → in_progress → review│
└────────┬─────────────────────────────┘
         │
         ├── Contract wrong? ──► fix story FIRST, record it, continue
         │
         ▼
┌──────────────────────────────────────┐
│   REVIEW AGENT (fresh context)       │
│   Diff vs acceptance criteria        │
│   Status: review → done              │
└────────┬─────────────────────────────┘
         │
         ├── APPROVED → done
         │
         └── NEEDS_CHANGES → back to Implementation
```

## Critical Rules

### 1. The Verification Loop Is Not Optional

Every change is compiled, linted and tested before it counts as done. Wire it
into hooks so it happens without being asked:

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          { "type": "command", "command": "${CLAUDE_PROJECT_DIR}/.claude/hooks/go-check.sh" }
        ]
      }
    ],
    "Stop": [
      {
        "hooks": [
          { "type": "command", "command": "${CLAUDE_PROJECT_DIR}/.claude/hooks/go-test.sh" }
        ]
      }
    ]
  }
}
```

`PostToolUse` fires after every file edit (`gofmt`, `golangci-lint`); `Stop`
fires before the turn ends (`go build`, `go test`). A hook exiting with code
**2** returns its stderr to the model as feedback. Nothing here depends on the
model remembering to check its own work.

### 2. Story and Code Never Diverge Silently

When implementation finds a contract is wrong:

```
❌ WRONG: change the code, leave the story describing the old design
✓ RIGHT: fix the story, record it under Contract Corrections, then the code
```

Small, local corrections are made by the implementer. Divergence spanning
several stories or a whole phase goes to the Reconcile Agent.

### 3. Don't Get Green by Weakening the Check

```
❌ Skipping a test, deleting an assertion, disabling a linter
✓ Fixing the code, or explaining why the expectation itself was wrong
```

A green suite bought this way is worse than a red one: it removes the signal.

### 4. Scope Is Bounded by the Story

The story's OUT list is binding. Work that looks adjacent and cheap belongs in
its own story - otherwise the diff outgrows the review and the acceptance
criteria stop covering it.

### 5. Prefer Semantic Navigation

Where `gopls` is available as an MCP server (`gopls mcp`, v0.20+), use its
definition/references/implementations tools instead of text search. It is
compiler-accurate and far cheaper in context than reading whole files.

## Task Tool Invocations

### Planning Agent

```
Task tool call:
- subagent_type: "systems-programming:golang-pro"
- model: "opus"
- prompt: |
    You are a PLANNING AGENT for story NNN-feature-name.

    YOUR JOB: write scope, contracts and acceptance criteria in the story.

    PROCESS:
    1. Read story: SERVICE_PATH/documentation/stories/NNN-feature-name.md
    2. Explore the codebase to understand existing patterns
    3. Write Scope (explicit IN and OUT)
    4. Write Contracts: signatures, schemas, error shapes, wire formats,
       invariants in prose
    5. Write Steps, each ending in a verification command
    6. Write checkable acceptance criteria
    7. Mark anything unknown as [NEEDS CLARIFICATION] - never guess
    8. Set status to "ready"

    RULES:
    - NO function bodies, NO full test suites, NO migrations
    - Name test cases that must be covered; do not write the suite
    - Do NOT create or modify code files

    Work directory: SERVICE_PATH/
```

### Implementation Agent

```
Task tool call:
- subagent_type: "systems-programming:golang-pro"
- model: "sonnet"
- prompt: |
    You are an IMPLEMENTATION AGENT for story NNN-feature-name.

    YOUR JOB: implement the contracts in the repository, verified.

    PROCESS:
    1. Read story: SERVICE_PATH/documentation/stories/NNN-feature-name.md
    2. Implement step by step, in the story's order
    3. After EVERY step: go build ./... && golangci-lint run ./... &&
       go test -race -count=1 ./...
    4. Fix what the tooling reports
    5. If a CONTRACT is wrong: fix the story first, record it under
       "Contract Corrections", then continue
    6. Update Implementation Notes and Files Changed
    7. Green → status "review". Blocked → status "in_progress" with exact
       errors and what you need

    RULES:
    - Do NOT expand into the story's OUT list
    - Do NOT skip tests, disable linters or weaken assertions to get green
    - Do NOT mark "review" on a red build

    Work directory: SERVICE_PATH/
```

### Review Agent

```
Task tool call:
- subagent_type: "code-documentation:code-reviewer"
- prompt: |
    Review the diff for story NNN-feature-name. You did NOT write this code.

    CHECK:
    1. Every acceptance criterion genuinely satisfied, not approximated
    2. Code matches the contracts; divergence recorded as Contract Correction
    3. Tests meaningful - no skipped cases, no weakened assertions
    4. Error handling, logging and style follow the project references
    5. Coverage thresholds met (domain 95%, application 80%)
    6. Nothing from the story's OUT list implemented

    OUTPUT:
    - APPROVED → set status to "done"
    - NEEDS_CHANGES → list specific issues
```

### Reconcile Agent

```
Task tool call:
- subagent_type: "systems-programming:golang-pro"
- model: "opus"
- prompt: |
    You are a RECONCILE AGENT for story NNN-feature-name.

    The story and the repository disagree. Your job:
    1. Establish what the code actually does - read it, run the tests
    2. Classify every contract: matches / story stale / code drifted / both wrong
    3. Decide which side is right, and why
    4. Update the story to describe reality, or specify the code change
    5. Record decisions under "Contract Corrections"
    6. Check dependent and same-phase stories; update them too

    RULES:
    - Do NOT edit code - specify the change, implementation verifies it
    - Do NOT assume the story is right because it is written down
    - Do NOT assume the code is right because it compiles

    Work directory: SERVICE_PATH/
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
| `ready` | Contracts and criteria written, reviewed | Planning / Reconcile Agent |
| `in_progress` | Implementation running, or blocked | Implementation Agent |
| `review` | Verification green, awaiting review | Implementation Agent |
| `done` | Review approved | Review Agent |

## Quick Reference

| When this happens... | Do this... |
|----------------------|------------|
| User creates story (draft) | Launch Planning Agent |
| Planning complete (ready) | User reviews contracts, then launch Implementation |
| Build/lint/test fails | Implementation fixes it - that is the loop, not an escalation |
| Contract proves wrong | Implementation fixes the story first, records it, continues |
| Divergence spans several stories | Launch Reconcile Agent |
| Implementation green (review) | Launch Review Agent with fresh context |
| Review fails | Back to Implementation with the specific issues |
| Review passes (done) | Story complete |

## Anti-Patterns to Avoid

### DON'T: Write the implementation into the story

```
❌ Story contains every file fully written; implementation copies it.
✓ Story contains interfaces and invariants; implementation writes and
   verifies the bodies.
```

### DON'T: Treat a failing build as an escalation

```
❌ "Build failed - handing back to planning."
✓ Build errors are the implementer's normal working material. Read the error,
   fix the code. Only a wrong CONTRACT goes back.
```

### DON'T: Let the story rot

```
❌ "The code changed but the story still describes the old interface -
   it's only documentation."
✓ A stale story misleads every later reader and every later agent. Reconcile it.
```

### DON'T: Plan what you cannot verify

```
❌ "Improve performance of the sync loop."
✓ "p99 of SyncZone under 2s for a 200-record zone; benchmark added."
```

## Related Documentation

- [Story Template](story-template.md) - Story file format
- [Planning Guide](planning-guide.md) - Planning Agent instructions
- [Implementation Guide](implementation-guide.md) - Implementation loop
- [Reconcile Guide](reconcile-guide.md) - Resolving story/code divergence

## Summary

| Role | Tier | Does |
|------|------|------|
| Planner | Opus | Scope, contracts, acceptance criteria |
| Implementer | Sonnet | Code in the repo, verified against tooling |
| Mechanical edits | Haiku | Single-file edits, codemods |
| Reviewer | fresh context | Diff vs acceptance criteria |
| Reconciler | Opus | Story/code divergence |

**Planning decides. Implementation builds and verifies. The story stays true.**
