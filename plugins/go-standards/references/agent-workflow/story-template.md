# Story File Template

## Format

Stories use Markdown with YAML frontmatter for metadata.

## Key Principle: Contract-First Planning

**A story fixes intent and contracts. Implementation writes the code in the repo.**

- Planning decides architecture, boundaries, signatures and acceptance criteria
- Implementation writes bodies and tests against compiler, linter and test feedback
- The story is the review surface; the repository is the source of truth for code

Function bodies, full test suites and migrations do **not** belong in a story.
Code that never compiled has been checked by nothing, and duplicating it in
Markdown guarantees drift the moment implementation adjusts anything.

## Template

```markdown
---
id: 42
title: "Feature: Short descriptive title"
status: draft
phase: a-3-catalog-sync
depends_on: []
files_touched:
  - internal/catalog/repository.go
  - internal/catalog/repository_test.go
acceptance:
  - "CatalogRepository.Upsert is idempotent for identical rows."
  - "go test -race ./internal/catalog/... passes."
verify: "make lint test"
operator_attention: false
created: 2026-09-17
updated: 2026-09-17
---

## Context

Background and motivation:
- Why this is needed
- Current state and pain points
- Relevant business context

## Scope

**IN:**
- What this story delivers, concretely.

**OUT:**
- What is deliberately deferred, and to which story if known.
- Adjacent work that a reviewer might otherwise expect.

## Contracts

Everything a second engineer needs in order to implement this without guessing.

### Types and signatures

```go
type CatalogRepository interface {
    Upsert(ctx context.Context, item Item) error
    ListByOwner(ctx context.Context, owner OwnerID) ([]Item, error)
}
```

### Data schema

Table `catalog_items`:

| Column | Type | Notes |
|---|---|---|
| `id` | uuid | primary key |
| `owner_id` | uuid | fk → owners(id), indexed |
| `name` | text | unique per owner |
| `updated_at` | timestamptz | set on every write |

### Errors

- `ErrItemNotFound` - sentinel, returned by the repository
- Use cases wrap with context: `fmt.Errorf("list catalog for %s: %w", owner, err)`
- HTTP maps `ErrItemNotFound` → 404, validation failures → 400

### Wire format

```
POST /v1/items  { "name": string, "owner_id": uuid }
201 → { "id": uuid }
409 → { "error": "item exists" }
```

### Invariants

State rules in prose - they survive refactoring, copied bodies do not.

- An item is identified by `(owner_id, name)`; the provider exposes no stable id.
- Writes are idempotent: re-sending an identical row is a no-op, not an error.

## Steps

Ordered, each ending in a check:

1. Repository interface + in-memory fake → `go build ./...`
2. Postgres implementation + integration test → `make test-integration`
3. Wire into the use case → `make lint test`

## Verification

```bash
make fmt lint test
go test -race -count=1 ./internal/catalog/...
```

## Open Questions

- [NEEDS CLARIFICATION: specific question that must be answered before `ready`]

Unanswered questions block `ready`. Never resolve them by plausible invention.

## Notes

Commit: `feat(catalog): add repository with idempotent upsert`

Anything else the implementer or reviewer needs to know.

---

## Implementation Notes

> Filled in during implementation

### Progress

- [ ] Step 1
- [ ] Step 2

### Verification Results

```
golangci-lint: [PASS/FAIL]
go test: [PASS/FAIL]
coverage: [X%]
```

### Contract Corrections

> If implementation proved a contract wrong, the story was fixed FIRST.
> Record what changed and why.

## Files Changed

[Filled in after implementation]

## Review Notes

[Reviewer comments]
```

## Field Descriptions

### Frontmatter Fields

| Field | Type | Description |
|-------|------|-------------|
| `id` | number | Sequential story number; other stories reference it |
| `title` | string | Short descriptive title with type prefix |
| `status` | enum | `draft`, `ready`, `in_progress`, `review`, `done` |
| `phase` | string | Groups stories belonging to one larger change |
| `depends_on` | array | Story ids that must be `done` first |
| `files_touched` | array | Expected blast radius, known before work starts |
| `acceptance` | array | Checkable statements that define done |
| `verify` | string | Exact command proving the story is complete |
| `operator_attention` | bool | `true` when a human must decide or act |
| `created` / `updated` | date | YYYY-MM-DD |

### Status Field Values

| Status | Meaning | Who Sets |
|--------|---------|----------|
| `draft` | Story created, needs planning | User |
| `ready` | Contracts and criteria written, reviewed, implementable | Planning Agent |
| `in_progress` | Implementation running | Implementation Agent |
| `review` | Verification green, awaiting review | Implementation Agent |
| `done` | Review approved | Review Agent |

Note: there is no separate "fix" status. Implementation fixes its own errors
against compiler and test output; only a **wrong contract** sends the story
back, and that is recorded in Contract Corrections.

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

## Writing Acceptance Criteria

Criteria must be checkable by reading output, not by opinion.

### Bad

```yaml
acceptance:
  - "Repository is well designed"
  - "Tests are added"
```

### Good

```yaml
acceptance:
  - "Upsert with an identical row twice leaves exactly one row and returns nil."
  - "ListByOwner returns items ordered by name, ascending."
  - "go test -race ./internal/catalog/... passes."
```

## Contracts: What Belongs, What Does Not

### Belongs

```markdown
```go
type Email struct{ value string }

func NewEmail(value string) (Email, error)
func (e Email) Value() string
func (e Email) IsZero() bool
```

`NewEmail` rejects empty input and anything without a single `@` with a
non-empty local part and domain, returning `ErrInvalidEmail`.
```

### Does Not Belong

```markdown
```go
func NewEmail(value string) (Email, error) {
    if value == "" {
        return Email{}, ErrInvalidEmail
    }
    if !emailRegex.MatchString(value) {
        return Email{}, ErrInvalidEmail
    }
    return Email{value: value}, nil
}
```
```

The signature and the rule are the contract. The body is implementation, and
it belongs in the repository where `go test` can check it.

## Sizing

Keep a story to roughly a day of work and a reviewable diff. If
`files_touched` sprawls across unrelated packages, or the steps cannot each end
in a check, split it and use `depends_on` to order the pieces.

## Naming Convention

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
│  Creates story: Context, Scope, Constraints                 │
│  Status: draft                                              │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    PLANNING (Opus)                          │
│  - Explores codebase                                        │
│  - Writes Scope, Contracts, Steps, Verification             │
│  - Marks open questions; they block `ready`                 │
│  Status: draft → ready                                      │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    USER REVIEW                              │
│  Reviews contracts and criteria - the high-leverage surface │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                 IMPLEMENTATION (Sonnet)                     │
│  - Writes code in the repo                                  │
│  - Compiles, lints, tests, fixes its own errors             │
│  - Contract wrong? Fix story first, then code               │
│  Status: ready → in_progress → review                       │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                 REVIEW (fresh context)                      │
│  - Reviews the diff against acceptance criteria             │
│  - Never the agent that wrote the code                      │
│  Status: review → done                                      │
└─────────────────────────────────────────────────────────────┘
```
