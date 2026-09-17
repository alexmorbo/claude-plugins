---
name: go-code-planning
description: "Contract-first planning for Go microservices. Use when creating story files, planning features, or writing technical specifications. Stories capture scope, contracts, acceptance criteria and verification commands - not function bodies. Implementation writes code in the repo against compiler and test feedback."
---

# Go Contract-First Planning Skill

A story is a **specification of intent**, not a copy of the implementation.
It fixes *what* must exist and *how success is verified*. The code itself is
written during implementation, in the repository, where the compiler, the
linter and the tests can answer back.

## Core Principle

```
┌────────────────────────────────────────────────────────────┐
│  PLANNING                                                  │
│                                                            │
│  ✓ Explore the codebase                                    │
│  ✓ Decide architecture and boundaries                      │
│  ✓ Write contracts: signatures, schemas, error shapes      │
│  ✓ Write acceptance criteria that can be checked           │
│  ✓ Write the verification command for every step           │
│                                                            │
│  ✗ NO function bodies                                      │
│  ✗ NO full test suites                                     │
│  ✗ NO migrations                                           │
│                                                            │
│  Code that never compiled is not a plan - it is a guess.   │
└────────────────────────────────────────────────────────────┘
```

Three reasons this boundary exists:

1. **No feedback.** Code written into Markdown never sees `go build`. Planning
   only improves outcomes when it encodes feedback; a plan full of untested
   code encodes none.
2. **Drift.** Implementation patches the repo to make tests pass. The story is
   not patched. What remains is a confident document describing a codebase that
   does not exist - and it misleads every later reader, human or agent.
3. **Reviewability.** A bad line in a plan becomes hundreds of bad lines of
   code, so the plan is the highest-leverage thing to review. That leverage
   disappears when the plan is as long as the implementation but has no tooling
   behind it.

## When to Use This Skill

- Creating a new story file
- Planning a feature or a refactor
- Writing a technical specification
- Breaking a phase into ordered, dependent units of work

## Story Frontmatter

```yaml
---
id: 337
title: Extract regrab + watchdog handlers wiring
status: draft          # draft → ready → in_progress → review → done
phase: b-11-cmd-server-refactor
depends_on: [332, 333, 334]
files_touched:
  - cmd/server/wiring/regrab.go
  - cmd/server/server.go
acceptance:
  - "wiring.BuildRegrab returns RegrabBundle with QbitSettingsUC, RegrabUC, RegrabLoop."
  - "Reload bus subscriber still receives qbit settings updates."
  - "go build ./... and go test -race ./... pass."
verify: "make lint test"
operator_attention: false
---
```

| Field | Purpose |
|---|---|
| `id` | Sequential number; stories reference each other by it |
| `phase` | Groups stories belonging to one larger change |
| `depends_on` | Stories that must reach `done` first |
| `files_touched` | Expected blast radius, known before work starts |
| `acceptance` | Checkable statements, not checkboxes of intent |
| `verify` | The exact command that proves the story is done |
| `operator_attention` | `true` when a human must decide or act |

## Story Sections

### Scope

Explicit **IN** and **OUT**. The OUT list prevents the implementation from
quietly expanding, and prevents a reviewer from asking for things the story
deliberately deferred.

```markdown
## Scope

**IN:**
- `wiring/regrab.go` exporting `BuildRegrab(...) (*RegrabBundle, error)`.
- Bundle covers qbit settings UC + HTTP handler, blacklist repo, regrab UC.

**OUT:**
- `loops.NewRegrabLoop(...).Start(rootCtx)` stays in server.go - it needs rootCtx.
- `startSubscribers(...)` keeps its current signature.
```

### Contracts

The heart of the story. Everything a second engineer would need to implement
this without guessing - and nothing more.

Include:
- **Type and function signatures** - names, parameters, return types
- **Interfaces** - declared on the consumer side
- **Data schemas** - table columns, indexes, constraints
- **Error shapes** - sentinel errors, which layer wraps what
- **Wire formats** - request/response bodies, status codes, headers

Exclude: bodies, loops, branches, error-handling detail. If the logic is
genuinely subtle, describe the rule in prose - an invariant survives
refactoring, a copied function body does not.

```markdown
## Contracts

```go
type ZoneRepository interface {
    Upsert(ctx context.Context, z Zone) error
    ListAll(ctx context.Context) ([]Zone, error)
}
```

Table `zones`: `id` (uuid, pk), `name` (text, unique), `provider` (text),
`expires_at` (timestamptz, null), `mode` (text: managed|readonly),
`synced_at` (timestamptz).

Errors: `ErrZoneNotFound` returned by the repository; the use case wraps it
as `fmt.Errorf("sync zone %s: %w", name, err)`.

Invariant: a record is identified by `(zone, name, type, value)` - the
provider exposes no stable record id.
```

### Steps

Ordered work, each step ending in a check. A step that cannot be verified is
too big or too vague.

```markdown
## Steps

1. Add `ZoneRepository` interface + in-memory fake → `go build ./...`
2. Postgres implementation + integration test → `make test-integration`
3. Wire into sync use case → `make lint test`
```

### Verification

The exact commands. Not "run the tests" - the command line, so there is no
interpretation left.

### Open Questions

Anything unresolved must be marked, never guessed:

```markdown
## Open Questions

- [NEEDS CLARIFICATION: does the provider rate-limit per account or per method?]
```

An unanswered question blocks `ready`. Plausible invention is the most
expensive failure mode in planning.

### Notes

The commit message the implementation should use, plus anything the next
reader needs.

## Model Tiers

Roles, not phases. Every tier works with tools and a closed verification loop.

| Tier | Responsibility |
|---|---|
| **Opus** | Architecture, contracts, decisions that cascade to other stories |
| **Sonnet** | Implementation against the contracts, cross-file changes |
| **Haiku** | Deterministic single-file edits, codemods, mechanical refactors |
| Review | A separate agent with fresh context - never the author of the code |

The implementer is a full agent: it edits, compiles, runs tests and fixes what
breaks. It is not a transcriber.

## Drift Rule

If implementation reveals that a contract in the story is wrong:

1. Stop.
2. Fix the story first.
3. Then continue the code.

Story and repository must never accumulate divergence. When they disagree, the
disagreement is the bug.

## Verification Loop

Planning is only worth the tokens if the implementation phase is wired to
feedback. Configure hooks in `.claude/settings.json`:

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

`PostToolUse` runs `gofmt` + `golangci-lint` after every edit; `Stop` runs
`go build` + `go test` before the turn is allowed to end. A hook exiting with
code **2** sends its stderr back to the model as feedback to act on.

## Checklist Before Setting Status to `ready`

- [ ] Scope has explicit IN and OUT
- [ ] Every contract a second engineer would need is written down
- [ ] No function bodies, no full test suites, no migrations in the story
- [ ] Acceptance criteria are checkable statements
- [ ] Every step ends in a verification command
- [ ] `verify` command is exact and runnable
- [ ] Open questions are either answered or marked and blocking
- [ ] `files_touched` and `depends_on` filled in
- [ ] Commit message drafted in Notes

## Reference Documentation

- `${CLAUDE_PLUGIN_ROOT}/references/agent-workflow/planning-guide.md` - Full guide
- `${CLAUDE_PLUGIN_ROOT}/references/agent-workflow/story-template.md` - Story format
- `${CLAUDE_PLUGIN_ROOT}/references/agent-workflow/implementation-guide.md` - Implementation loop
- `${CLAUDE_PLUGIN_ROOT}/references/clean-architecture.md` - Layer rules and when they apply
- `${CLAUDE_PLUGIN_ROOT}/references/code-style.md` - Code conventions
