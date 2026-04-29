# Linting & Code Quality

## Overview

All Go services must pass linting and quality checks before merge. The CI pipeline runs these checks automatically, but agents and developers should verify code locally first.

## golangci-lint

Primary linter for Go code. The CI uses `golangci-lint run ./... --timeout=5m`.

### Project Configuration

Create `.golangci.yml` in project root:

```yaml
run:
  timeout: 5m
  modules-download-mode: readonly

linters:
  enable:
    - errcheck
    - gosimple
    - govet
    - ineffassign
    - staticcheck
    - unused
    - gofmt
    - goimports
    - misspell
    - unconvert
    - unparam
    - gosec
    - bodyclose
    - noctx
    - prealloc
    - revive
    - gocritic
    - errorlint
    - exhaustive
    - nilerr
    - nilnil

linters-settings:
  goimports:
    local-prefixes: gitlab.morbo.dev

  revive:
    rules:
      - name: exported
        disabled: true
      - name: package-comments
        disabled: true

  gocritic:
    enabled-checks:
      - appendAssign
      - argOrder
      - badCall
      - badCond
      - badLock
      - badRegexp
      - builtinShadow
      - caseOrder
      - codegenComment
      - commentFormatting
      - defaultCaseOrder
      - deprecatedComment
      - dupArg
      - dupBranchBody
      - dupCase
      - dupSubExpr
      - emptyFallthrough
      - equalFold
      - evalOrder
      - exitAfterDefer
      - flagDeref
      - flagName
      - nilValReturn
      - offBy1
      - regexpMust
      - sloppyLen
      - sloppyTypeAssert
      - sortSlice
      - sprintfQuotedString
      - sqlQuery
      - syncMapLoadAndDelete
      - truncateCmp
      - unnecessaryDefer
      - weakCond

issues:
  max-issues-per-linter: 0
  max-same-issues: 0
  exclude-rules:
    - path: _test\.go
      linters:
        - errcheck
        - gosec
    - path: mock_
      linters:
        - unused
```

### Running Locally

```bash
# Run all linters
golangci-lint run ./...

# Run with auto-fix where possible
golangci-lint run ./... --fix

# Check specific package
golangci-lint run ./domain/...

# Verbose output
golangci-lint run ./... -v
```

## Additional Quality Tools

CI also runs these tools (in `quality_check` stage):

| Tool | Purpose | Command |
|------|---------|---------|
| `gocyclo` | Cyclomatic complexity (max 15) | `gocyclo -over 15 .` |
| `ineffassign` | Ineffectual assignments | `ineffassign ./...` |
| `misspell` | Spelling mistakes | `misspell ./...` |
| `staticcheck` | Static analysis | `staticcheck ./...` |
| `go vet` | Go's built-in checks | `go vet ./...` |

### Cyclomatic Complexity

Functions should have complexity ≤15. Refactor large functions:

```go
// BAD - High complexity
func processOrder(order *Order) error {
    if order == nil { return err }
    if order.Status == "" { return err }
    if order.Items == nil { return err }
    // ... 20 more conditions
}

// GOOD - Split into focused functions
func processOrder(order *Order) error {
    if err := validateOrder(order); err != nil {
        return err
    }
    return executeOrder(order)
}

func validateOrder(order *Order) error {
    // validation logic
}
```

## Security Scanning

CI runs security tools (in `security` stage):

| Tool | Purpose |
|------|---------|
| `gosec` | Security vulnerability scanner |
| `govulncheck` | Official Go vulnerability checker |
| `nancy` | Dependency vulnerability scanner |

Security issues generate reports but don't block pipeline.

```bash
# Run locally
gosec ./...
govulncheck ./...
go list -json -m all | nancy sleuth
```

## Agent Verification Requirements

**Before completing implementation**, agents MUST verify:

### 1. Lint Check

```bash
golangci-lint run ./...
```

All linter errors must be resolved. No `//nolint` without justification.

### 2. Test Execution

```bash
go test ./... -v
```

All tests must pass.

### 3. Coverage Check

```bash
go test ./... -coverprofile=coverage.out
go tool cover -func=coverage.out | tail -1
```

Coverage must meet or exceed project's `COVERAGE_THRESHOLD` (default: 80%).

For CA services with layer-specific thresholds:

| Layer | Minimum | Command |
|-------|---------|---------|
| Domain | 95% | `go test ./domain/... -cover` |
| Application | 80% | `go test ./application/... -cover` |
| Infrastructure | 70% | `go test ./infrastructure/... -cover` |
| Interface | 70% | `go test ./interface/... -cover` |

### 4. Build Verification

```bash
go build ./...
```

Code must compile without errors.

## Common Lint Issues

### errcheck - Unhandled errors

```go
// BAD
file.Close()

// GOOD
if err := file.Close(); err != nil {
    return fmt.Errorf("close file: %w", err)
}

// Or use defer with named return
defer func() {
    if cerr := file.Close(); cerr != nil && err == nil {
        err = cerr
    }
}()
```

### bodyclose - Unclosed HTTP response body

```go
// BAD
resp, _ := http.Get(url)
// body never closed

// GOOD
resp, err := http.Get(url)
if err != nil {
    return err
}
defer resp.Body.Close()
```

### noctx - HTTP request without context

```go
// BAD
req, _ := http.NewRequest("GET", url, nil)

// GOOD
req, err := http.NewRequestWithContext(ctx, "GET", url, nil)
```

### gosec - Security issues

```go
// BAD - SQL injection
query := "SELECT * FROM users WHERE id = " + userID
db.Raw(query)

// GOOD - Parameterized query
db.Raw("SELECT * FROM users WHERE id = ?", userID)
```

### errorlint - Error wrapping

```go
// BAD
if err == ErrNotFound {

// GOOD
if errors.Is(err, ErrNotFound) {
```

## Makefile Integration

Standard Makefile targets:

```makefile
.PHONY: lint test test-coverage build verify

lint:
	golangci-lint run ./...

test:
	go test ./... -v

test-coverage:
	go test ./... -coverprofile=coverage.out -covermode=atomic
	go tool cover -func=coverage.out | tail -1

build:
	go build -o bin/service ./cmd/server

# Run all checks before commit
verify: lint test build
	@echo "All checks passed"
```

## Pre-commit Hooks

Configure `.pre-commit-config.yaml`:

```yaml
repos:
  - repo: https://github.com/dnephin/pre-commit-golang
    rev: v0.5.1
    hooks:
      - id: go-fmt
      - id: go-vet
      - id: go-imports
      - id: golangci-lint
        args: [--timeout=5m]
```

Install:

```bash
pip install pre-commit
pre-commit install
```

## CI Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `COVERAGE_THRESHOLD` | `80` | Minimum overall coverage (%) |
| `DOMAIN_COVERAGE_THRESHOLD` | `95` | Domain layer coverage (CA services) |
| `LINT_TIMEOUT` | `5m` | golangci-lint timeout |
