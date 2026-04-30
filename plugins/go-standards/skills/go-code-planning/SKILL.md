---
name: go-code-planning
description: "Code-first planning for Go microservices. Use when creating story files, planning features, or writing technical specifications. Opus writes COMPLETE, PRODUCTION-READY code in story files. This skill ensures all code is written during planning, not implementation."
---

# Go Code-First Planning Skill

This skill ensures Opus writes complete, production-ready code in story files.
Haiku will only copy this code to files - no thinking, no modifications.

## Core Principle

```
┌────────────────────────────────────────────────────────────┐
│  OPUS (You)                                            │
│                                                            │
│  ✓ Explore codebase                                        │
│  ✓ Make ALL decisions                                      │
│  ✓ Write COMPLETE code                                     │
│  ✓ Write ALL tests                                         │
│                                                            │
│  Your code must be PERFECT. There is no safety net.        │
└────────────────────────────────────────────────────────────┘
```

## When to Use This Skill

- Creating a new story file
- Planning a new feature
- Writing technical specification
- Any task that involves "planning" Go code

## Story File Structure

Each story MUST contain complete code in this format:

```markdown
## Technical Specification

### 1. Domain Layer

#### File: `domain/valueobject/email.go`

```go
package valueobject

// COMPLETE CODE HERE
// All imports
// All functions
// All error handling
```

#### File: `domain/valueobject/email_test.go`

```go
package valueobject_test

// COMPLETE TESTS HERE
// Table-driven tests
// All edge cases
```
```

## Code Block Requirements

Every code block MUST:

1. **Have exact path**: `#### File: \`exact/path/to/file.go\``
2. **Be complete**: All imports, all functions, all logic
3. **Be production-ready**: No TODOs, no placeholders
4. **Include tests**: Every `.go` has `_test.go`

## What You MUST Include

### In Every Code File:
- Package declaration
- All imports (including third-party)
- Complete implementation
- Error handling with context wrapping
- Godoc for public APIs

### In Every Test File:
- Table-driven tests
- Edge cases (empty, nil, invalid)
- Error path testing
- testify/assert and testify/require

## What You MUST NOT Do

### DO NOT write incomplete code:
```go
// BAD
func CreateUser() (*User, error) {
    // TODO: implement
}
```

### DO NOT leave decisions for implementation:
```go
// BAD
// Choose appropriate error handling approach
func Validate() error {
    // implementation decides
}
```

### DO NOT write abstract plans:
```markdown
<!-- BAD -->
1. Create email value object
2. Add validation
3. Write tests
```

## Clean Architecture Order

Write code in dependency order:

1. **Domain** (innermost - no dependencies)
   - `domain/valueobject/` - Value Objects
   - `domain/entity/` - Entities
   - `domain/repository/` - Repository Interfaces
   - `domain/service/` - Domain Services

2. **Application** (depends on Domain)
   - `application/dto/` - DTOs
   - `application/usecase/` - Use Cases
   - `application/port/` - Ports

3. **Infrastructure** (depends on Domain, Application)
   - `infrastructure/persistence/` - Repository Implementations
   - `infrastructure/external/` - External Clients

4. **Interface** (outermost)
   - `interface/http/handler/` - HTTP Handlers
   - `interface/http/middleware/` - Middleware

## Session Rules

### MUST End Session After Planning

After setting story status to `ready`:

1. Inform user that code is ready
2. List all files to be created
3. Explain next step (new session for implementation)
4. **STOP** - Do NOT start implementation

### Example Final Message:

```
Planning complete. Story file contains production-ready code.

Files to be created:
- domain/valueobject/email.go
- domain/valueobject/email_test.go
- domain/entity/user.go
- domain/entity/user_test.go
[...]

Status: ready

Next steps:
1. Review the code in story file
2. Start NEW session for implementation (Haiku copies code)

This planning session is complete.
```

## Verification Before `ready`

- [ ] All files have `#### File: \`path\`` format
- [ ] All code is complete (no TODOs)
- [ ] All imports included
- [ ] All error handling implemented
- [ ] All tests written (table-driven)
- [ ] Domain layer pure (no infrastructure imports)
- [ ] Follows project code style
- [ ] Agent Execution section has Task tool prompts
- [ ] Status set to `ready`

## Reference Documentation

- `${CLAUDE_PLUGIN_ROOT}/references/agent-workflow/planning-guide.md` - Full guide
- `${CLAUDE_PLUGIN_ROOT}/references/agent-workflow/story-template.md` - Story format
- `${CLAUDE_PLUGIN_ROOT}/references/clean-architecture.md` - Layer rules
- `${CLAUDE_PLUGIN_ROOT}/references/code-style.md` - Code conventions
