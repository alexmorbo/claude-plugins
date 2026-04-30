# Story File Template

## Format

Stories use Markdown with YAML frontmatter for metadata.

## Key Principle: Code-First Planning

**Opus writes COMPLETE, PRODUCTION-READY code in the story file.**
**Haiku only copies this code to actual files.**

This ensures:
- Opus (smarter model) makes all architectural decisions
- Haiku (faster model) only performs mechanical copy-paste
- No implementation decisions left to the faster model

## Template

```markdown
---
title: "Feature: Short descriptive title"
status: draft
priority: medium
complexity: 5
planning_model: opus
implementation_model: haiku
created: 2025-01-16
updated: 2025-01-16
risk_areas: []
---

## Context

Describe the background and motivation for this task. Include:
- Why this feature/fix is needed
- Current state and pain points
- Any relevant business context

## User Story

**As a** [role],
**I want to** [capability],
**So that** [benefit].

## Acceptance Criteria

- [ ] Criterion 1: Specific, measurable outcome
- [ ] Criterion 2: Another specific outcome
- [ ] Criterion 3: Test requirements
- [ ] Tests pass and coverage maintained

## Constraints

List any technical or business constraints:
- Must be backwards compatible
- Must not affect existing API
- Performance requirements

---

## Technical Specification

> **CRITICAL**: This section contains COMPLETE, READY-TO-USE code.
> The Implementation Agent (Haiku) will COPY this code exactly to files.
> All architectural decisions, error handling, and tests are defined here.

### Analysis

[Summary of codebase exploration findings by Planning Agent]

### Implementation Order

1. Domain Layer (entities, value objects, repository interfaces)
2. Application Layer (use cases, DTOs, ports)
3. Infrastructure Layer (repository implementations, external adapters)
4. Interface Layer (HTTP handlers, middleware)

---

### 1. Domain Layer

#### File: `domain/valueobject/example.go`

```go
package valueobject

// Complete, production-ready code here
// All imports included
// All error handling included
```

#### File: `domain/valueobject/example_test.go`

```go
package valueobject_test

// Complete test file
// Table-driven tests
// All edge cases covered
```

#### File: `domain/entity/example.go`

```go
package entity

// Entity code with all business rules
```

#### File: `domain/repository/example_repository.go`

```go
package repository

// Repository INTERFACE only (not implementation)
```

---

### 2. Application Layer

#### File: `application/usecase/example_usecase.go`

```go
package usecase

// Use case with all orchestration logic
```

#### File: `application/usecase/example_usecase_test.go`

```go
package usecase_test

// Use case tests with mocks
```

#### File: `application/dto/example_dto.go`

```go
package dto

// Input/Output DTOs
```

---

### 3. Infrastructure Layer

#### File: `infrastructure/persistence/example_repository_postgres.go`

```go
package persistence

// Repository implementation with GORM
```

#### File: `infrastructure/persistence/example_repository_postgres_test.go`

```go
package persistence_test

// Repository tests (unit with mocked DB)
```

---

### 4. Interface Layer

#### File: `interface/http/handler/example_handler.go`

```go
package handler

// HTTP handler - only HTTP concerns
// Request binding, response formatting
// Delegates to use case
```

#### File: `interface/http/handler/example_handler_test.go`

```go
package handler_test

// Handler tests with httptest
```

---

### Dependencies

[External libraries or internal modules required - if any new ones]

### Risks

[Potential issues identified during planning]

---

## Agent Execution

> **MANDATORY**: All operations MUST be executed via Task tool agents.
> Main context ONLY orchestrates - launches agents and reviews results.

### Implementation Agent Instructions

```
Task tool call:
- subagent_type: "systems-programming:golang-pro"
- model: "haiku"
- prompt: |
    You are an IMPLEMENTATION AGENT for story NNN-feature-name.

    YOUR ONLY JOB: Copy code from the story file to actual files.

    RULES:
    - DO NOT modify the code
    - DO NOT add anything
    - DO NOT "improve" anything
    - DO NOT fix errors
    - ONLY copy code blocks to their specified file paths

    PROCESS:
    1. Read story file: SERVICE_PATH/documentation/stories/NNN-feature-name.md
    2. Find each "#### File: `path`" section
    3. Use Write tool to create file at that path with the code block content
    4. Repeat for ALL files in the story
    5. Run verification:
       - golangci-lint run ./...
       - go test ./...
    6. If verification PASSES: set status to "review"
    7. If verification FAILS:
       - Write errors to "Issues Found" section
       - Keep status as "in_progress"
       - STOP (do NOT fix them)

    Work directory: SERVICE_PATH/
```

### Fix Agent Instructions

> **When to use**: When Implementation Agent fails with errors.

```
Task tool call:
- subagent_type: "systems-programming:golang-pro"
- model: "sonnet"
- prompt: |
    You are a FIX AGENT for story NNN-feature-name.

    Implementation failed with errors. Your job:
    1. Read story: SERVICE_PATH/documentation/stories/NNN-feature-name.md
    2. Find "Issues Found" section with error details
    3. Fix ALL errors in the code blocks IN THE STORY FILE
    4. Document fixes in "Fixes Applied" section
    5. Clear "Issues Found" section
    6. Set status to "ready"

    RULES:
    - ONLY edit the story file, NOT actual code files
    - Fix ALL errors, not just some
    - Do NOT add new features
    - Do NOT run tests
    - Do NOT create actual files

    Work directory: SERVICE_PATH/
```

### Code Review Agent Instructions

```
Task tool call:
- subagent_type: "code-documentation:code-reviewer"
- model: "haiku"
- prompt: |
    Review implementation for story NNN-feature-name.

    CHECKLIST:
    1. All files from story were created correctly
    2. Code matches story exactly (no modifications)
    3. Tests pass
    4. Lint passes
    5. Coverage meets thresholds (domain 95%, application 80%, overall 80%)

    OUTPUT:
    - APPROVED: Set status to "done"
    - NEEDS_CHANGES: Report issues (Fix Agent must fix the story)
```

---

## Implementation Notes

> Updated by Implementation Agent during execution

### Progress

- [ ] Domain layer files created
- [ ] Application layer files created
- [ ] Infrastructure layer files created
- [ ] Interface layer files created
- [ ] Verification passed

### Verification Results

```
golangci-lint: [PASS/FAIL]
go test: [PASS/FAIL]
coverage: [X%]
```

### Issues Found

[If verification fails, Implementation Agent lists exact errors here]

### Fixes Applied

[Fix Agent documents all fixes here after correcting errors in code blocks]

---

## Files Changed

[List of all files created/modified - filled after implementation]

---

## Review Notes

> Code review feedback

[Reviewer comments]
```

## Field Descriptions

### Frontmatter Fields

| Field | Type | Description |
|-------|------|-------------|
| `title` | string | Short descriptive title with type prefix |
| `status` | enum | `draft`, `planning`, `ready`, `in_progress`, `review`, `done` |
| `priority` | enum | `high`, `medium`, `low` |
| `complexity` | number | 1-10 scale (affects review requirements) |
| `planning_model` | string | Model for planning: `opus` (recommended) |
| `implementation_model` | string | Model for implementation: `haiku` |
| `created` | date | Story creation date (YYYY-MM-DD) |
| `updated` | date | Last update date (YYYY-MM-DD) |
| `depends_on` | string | (optional) Story ID that must be completed first |
| `risk_areas` | array | (optional) Areas of risk: `database`, `api-breaking`, `security` |

### Complexity Scale

| Level | Description | Review Requirements |
|-------|-------------|---------------------|
| 1-3 | Simple, isolated change | Self-review OK |
| 4-6 | Moderate, touches multiple files | Code review required |
| 7-8 | Complex, architectural impact | Senior review required |
| 9-10 | Critical, system-wide changes | Architecture review required |

### Status Field Values

| Status | Meaning | Who Sets |
|--------|---------|----------|
| `draft` | Story created, needs planning | User |
| `planning` | Opus writing code in story | Planning Agent (Opus) |
| `ready` | Code written, ready for copy to files | Planning Agent (Opus) |
| `in_progress` | Haiku copying code to files | Implementation Agent (Haiku) |
| `review` | Files created, awaiting review | Implementation Agent (Haiku) |
| `done` | Review approved | Code Review Agent |

### Title Prefixes

| Prefix | Description |
|--------|-------------|
| `Feature:` | New functionality |
| `Fix:` | Bug fix |
| `Refactor:` | Code restructuring |
| `Perf:` | Performance improvement |
| `Docs:` | Documentation |
| `Test:` | Test additions/fixes |
| `Chore:` | Maintenance tasks |

## Code Block Requirements

Each code block in Technical Specification MUST:

1. **Have exact file path**: `#### File: \`exact/path/to/file.go\``
2. **Be complete**: All imports, all functions, all error handling
3. **Be production-ready**: No TODOs, no placeholders, no "implement later"
4. **Include tests**: Every `.go` file has corresponding `_test.go`
5. **Follow standards**: Clean Architecture, project code style

### Bad Example (DO NOT DO THIS)

```markdown
#### File: `domain/entity/user.go`

```go
// TODO: implement user entity
type User struct {
    // add fields
}

func NewUser() *User {
    // implement
}
```
```

### Good Example

```markdown
#### File: `domain/entity/user.go`

```go
package entity

import (
    "errors"
    "time"

    "service/domain/valueobject"
)

var (
    ErrEmptyUserName = errors.New("user name cannot be empty")
)

type User struct {
    id        valueobject.UserID
    email     valueobject.Email
    name      string
    createdAt time.Time
    updatedAt time.Time
}

func NewUser(email valueobject.Email, name string) (*User, error) {
    if name == "" {
        return nil, ErrEmptyUserName
    }

    return &User{
        id:        valueobject.NewUserID(),
        email:     email,
        name:      name,
        createdAt: time.Now(),
        updatedAt: time.Now(),
    }, nil
}

func (u *User) ID() valueobject.UserID {
    return u.id
}

func (u *User) Email() valueobject.Email {
    return u.email
}

func (u *User) Name() string {
    return u.name
}

func (u *User) UpdateName(name string) error {
    if name == "" {
        return ErrEmptyUserName
    }
    u.name = name
    u.updatedAt = time.Now()
    return nil
}
```
```

## Naming Convention

Story files should be named with a sequential number prefix:

```
NNN-short-kebab-case-description.md
```

Examples:
- `001-initial-project-setup.md`
- `002-add-user-authentication.md`
- `003-implement-order-processing.md`

## Workflow Summary

```
┌─────────────────────────────────────────────────────────────┐
│                    USER                                     │
│  Creates story file with Context, User Story, Constraints   │
│  Status: draft                                              │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    OPUS (Planning)                      │
│  - Explores codebase                                        │
│  - Writes COMPLETE CODE in story file                       │
│  - All decisions made here                                  │
│  Status: planning → ready                                   │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    USER REVIEW                              │
│  Reviews the code in story file                             │
│  Approves or requests changes                               │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    HAIKU (Implementation)                   │
│  - Reads story file                                         │
│  - COPIES code to files (no modifications)                  │
│  - Runs verification                                        │
│  Status: in_progress → review                               │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    HAIKU (Code Review)                      │
│  - Verifies files match story                               │
│  - Checks tests/coverage                                    │
│  Status: review → done                                      │
└─────────────────────────────────────────────────────────────┘
```
