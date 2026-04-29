---
name: go-linting
description: "Enforces code quality and linting standards for Go services. Use after writing Go code, before completing implementation, or when fixing lint errors. Applies to all Go code modifications and must be verified before marking tasks complete."
---

# Go Linting & Code Quality Skill

This skill ensures all Go code passes linting and quality checks based on `${CLAUDE_PLUGIN_ROOT}/references/linting.md`.

## Mandatory Verification

**BEFORE completing any Go implementation**, run these checks:

```bash
# 1. Linter - ALL errors must be resolved
golangci-lint run ./...

# 2. Tests - ALL must pass
go test ./... -v

# 3. Coverage - must meet threshold (80% overall, 95% domain)
go test ./... -coverprofile=coverage.out
go tool cover -func=coverage.out | tail -1

# 4. Build - must compile
go build ./...
```

## golangci-lint Configuration

Project should have `.golangci.yml`:

```yaml
run:
  timeout: 5m
  modules-download-mode: readonly

linters:
  enable:
    - errcheck       # Unchecked errors
    - gosimple       # Simplifications
    - govet          # Go vet checks
    - ineffassign    # Ineffectual assignments
    - staticcheck    # Static analysis
    - unused         # Unused code
    - gofmt          # Formatting
    - goimports      # Import organization
    - misspell       # Spelling
    - gosec          # Security
    - bodyclose      # HTTP body close
    - noctx          # Context in HTTP
    - errorlint      # Error handling
    - revive         # Linting rules

linters-settings:
  goimports:
    local-prefixes: gitlab.morbo.dev

issues:
  exclude-rules:
    - path: _test\.go
      linters:
        - errcheck
        - gosec
```

## Common Lint Issues & Fixes

### errcheck - Unhandled Errors

```go
// BAD
file.Close()
json.Unmarshal(data, &obj)

// GOOD
if err := file.Close(); err != nil {
    return fmt.Errorf("close file: %w", err)
}

if err := json.Unmarshal(data, &obj); err != nil {
    return fmt.Errorf("unmarshal: %w", err)
}

// Or with defer
defer func() {
    if err := file.Close(); err != nil {
        slog.Error("failed to close file", "error", err)
    }
}()
```

### bodyclose - HTTP Response Body

```go
// BAD - Body never closed
resp, err := http.Get(url)
if err != nil {
    return err
}
// body leaks!

// GOOD
resp, err := http.Get(url)
if err != nil {
    return err
}
defer resp.Body.Close()
```

### noctx - HTTP Without Context

```go
// BAD
req, _ := http.NewRequest("GET", url, nil)

// GOOD
req, err := http.NewRequestWithContext(ctx, "GET", url, nil)
if err != nil {
    return fmt.Errorf("create request: %w", err)
}
```

### errorlint - Error Comparison

```go
// BAD - Direct comparison
if err == ErrNotFound {
if err == sql.ErrNoRows {

// GOOD - Use errors.Is
if errors.Is(err, ErrNotFound) {
if errors.Is(err, sql.ErrNoRows) {

// BAD - Type assertion
if ve, ok := err.(*ValidationError); ok {

// GOOD - Use errors.As
var ve *ValidationError
if errors.As(err, &ve) {
```

### gosec - Security Issues

```go
// BAD - SQL injection
query := "SELECT * FROM users WHERE id = " + userID
db.Raw(query)

// GOOD - Parameterized
db.Raw("SELECT * FROM users WHERE id = ?", userID)

// BAD - Weak crypto
rand.Seed(time.Now().UnixNano())
token := rand.Int()

// GOOD - Crypto rand
b := make([]byte, 32)
_, err := crand.Read(b)
token := base64.URLEncoding.EncodeToString(b)
```

### ineffassign - Ineffectual Assignment

```go
// BAD
err := doSomething()
err = doSomethingElse()  // First err never used

// GOOD
if err := doSomething(); err != nil {
    return err
}
if err := doSomethingElse(); err != nil {
    return err
}
```

### unused - Unused Code

```go
// BAD - Unused function
func helperFunc() {  // lint error: unused
    // ...
}

// GOOD - Remove or use it
// If needed later, don't write it now
```

## Additional Quality Checks

### Cyclomatic Complexity (max 15)

```go
// BAD - Too complex
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
```

### Go Vet

```bash
go vet ./...
```

Catches:
- Printf format errors
- Unreachable code
- Incorrect struct tags
- Suspicious constructs

### Staticcheck

```bash
staticcheck ./...
```

Catches:
- Deprecated APIs
- Inefficient code
- Common mistakes

## Running Locally

```bash
# Full lint
golangci-lint run ./...

# Auto-fix where possible
golangci-lint run ./... --fix

# Specific package
golangci-lint run ./domain/...

# Verbose
golangci-lint run ./... -v
```

## CI Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `COVERAGE_THRESHOLD` | `80` | Minimum overall coverage |
| `DOMAIN_COVERAGE_THRESHOLD` | `95` | Domain layer coverage |
| `LINT_TIMEOUT` | `5m` | Linter timeout |

## Nolint Directive

Use sparingly with justification:

```go
// ACCEPTABLE - With reason
//nolint:errcheck // Error logged in defer, not critical for caller
defer file.Close()

// UNACCEPTABLE - No reason
//nolint:errcheck
file.Close()
```

## Verification Checklist

Before completing implementation:

- [ ] `golangci-lint run ./...` passes with no errors
- [ ] `go test ./... -v` all tests pass
- [ ] Coverage >= 80% overall (`go tool cover -func=coverage.out`)
- [ ] Domain coverage >= 95%
- [ ] `go build ./...` compiles successfully
- [ ] No `//nolint` without justification
- [ ] No TODO/FIXME left unaddressed

## If Lint Fails

1. Read the error message carefully
2. Check this skill for common fixes
3. Fix the issue in code (don't suppress)
4. Re-run lint to verify
5. Only use `//nolint` as last resort with justification
