Diff-aware code review for the current Go branch — runs all major static analysis tools and produces a severity-tagged report.

## Usage

```
/go-review                    # review all changes vs. main (or origin/main)
/go-review <base-branch>      # review vs. a specific branch
/go-review HEAD~3             # review last N commits
```

## What it checks

For every `.go` file changed in the diff (and the packages they belong to):

1. **`go vet ./...`** — official static analysis (printf checks, suspicious constructs)
2. **`staticcheck ./...`** — deeper static analysis (deprecated APIs, inefficiencies)
3. **`golangci-lint run`** — full linter suite per project's `.golangci.yml`
4. **`gosec ./...`** — security: hardcoded credentials, weak crypto, SQL injection, command injection, path traversal
5. **`govulncheck ./...`** — known CVEs in dependencies that the code actually uses
6. **`go test ./...`** — tests for the packages touched (skip with `-short` if integration tests heavy)
7. **`go build ./...`** — must compile

## Execution flow

When this command is invoked, perform the following steps:

### 1. Determine the diff scope

```bash
# Default: branch base or origin/main
BASE_BRANCH="${1:-$(git merge-base HEAD origin/main 2>/dev/null || git merge-base HEAD main 2>/dev/null || echo HEAD~1)}"

# List of changed Go files
CHANGED_FILES=$(git diff --name-only "$BASE_BRANCH"...HEAD -- '*.go')

# List of packages containing changed files (deduplicated)
CHANGED_PKGS=$(echo "$CHANGED_FILES" | xargs -I{} dirname {} | sort -u | sed 's|^|./|')
```

If `CHANGED_FILES` is empty, report "no Go changes vs. $BASE_BRANCH" and exit.

### 2. Run checks in parallel where safe

Execute these in parallel, capture both stdout and exit codes:

```bash
go vet ./... 2>&1
staticcheck ./... 2>&1
golangci-lint run --out-format=line-number 2>&1
gosec -quiet -fmt=text ./... 2>&1
govulncheck ./... 2>&1
go test -short -race -count=1 $CHANGED_PKGS 2>&1
go build ./... 2>&1
```

If a tool isn't installed, **note this in the report and continue** — don't fail the whole review. Common install hints to include:

```
staticcheck:  go install honnef.co/go/tools/cmd/staticcheck@latest
golangci-lint: brew install golangci-lint
gosec:        go install github.com/securego/gosec/v2/cmd/gosec@latest
govulncheck:  go install golang.org/x/vuln/cmd/govulncheck@latest
```

### 3. Classify findings by severity

| Severity | Includes |
|----------|----------|
| **CRITICAL** | govulncheck CVEs that match called paths; gosec rules `G101` (creds), `G201/G202` (SQL injection), `G204` (command injection) |
| **HIGH** | Failing tests; build errors; staticcheck `SA*` rules (real bugs); other gosec findings; `go vet` errors |
| **MEDIUM** | golangci-lint warnings; staticcheck `ST*` rules (style); deprecated API use |
| **LOW** | Style/naming nits not enforced by linters but matching `${CLAUDE_PLUGIN_ROOT}/references/anti-patterns.md` |

### 4. Format the report

Output structure:

```markdown
# Code Review — branch <BRANCH> vs. <BASE>

**Status:** <PASS | WARN | FAIL>
**Files changed:** N (.go); packages: M
**Tools run:** vet ✓ · staticcheck ✓ · golangci-lint ✓ · gosec ✓ · govulncheck ✓ · tests ✓ · build ✓

## CRITICAL (X)

- `<file>:<line>` — <message>
  - Why: <one-line explanation>
  - Fix: <one-line suggestion>

## HIGH (Y)
...

## MEDIUM (Z)
...

## LOW (W)
...

## Skipped tools

- `<tool>` — not installed, install with `<command>`
```

**PASS / WARN / FAIL rule:**
- **FAIL** if any CRITICAL or HIGH
- **WARN** if any MEDIUM
- **PASS** if only LOW or none

### 5. Don't auto-fix

This command is read-only. If the user wants fixes, they should ask explicitly ("apply the suggested fixes for the HIGH findings"). The review is for assessment, not action.

## Notes

- Respect the project's `.golangci.yml` — don't override its config.
- For monorepos, scope to changed packages where possible to keep runtime under a minute.
- govulncheck only flags CVEs in code paths actually reached — false positives are rare; treat its findings as real.
- Tests run with `-short` so integration tests skip; a separate `/go-audit` covers heavier checks.

## Related

- `/go-audit` — heavier pre-release security/license scan
- `/go-gen-test` — generate tests for newly added code that's missing coverage
- `${CLAUDE_PLUGIN_ROOT}/references/anti-patterns.md` — what to flag as LOW severity
- `${CLAUDE_PLUGIN_ROOT}/references/linting.md` — `.golangci.yml` config guide
