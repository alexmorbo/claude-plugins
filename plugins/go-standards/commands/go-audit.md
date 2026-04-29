Pre-release safety audit: vulnerabilities, security findings, license compliance, dependency hygiene.

## Usage

```
/go-audit              # full audit
/go-audit deps         # only dependency-related (vuln, licenses, outdated)
/go-audit security     # only gosec + govulncheck
```

## What it checks

| Category | Tool | Purpose |
|----------|------|---------|
| Known CVEs | `govulncheck ./...` | Vulnerabilities in deps that the code actually uses |
| Security patterns | `gosec ./...` | Hardcoded secrets, weak crypto, SQL/command injection, path traversal, unsafe filesystem ops |
| Licenses | `go-licenses report ./...` | Catch GPL/AGPL/Unknown licenses in dependencies |
| Outdated deps | `go list -u -m all` | Major versions with available updates |
| Module hygiene | `go mod tidy -diff` | Detect un-tidied `go.mod` / unused deps |
| Indirect deps growth | `go mod graph \| wc -l` vs. baseline | Catch dependency creep |

## Difference from `/go-review`

| Concern | `/go-review` | `/go-audit` |
|---------|-------------|-------------|
| Scope | Diff vs. base branch | Whole module |
| Frequency | Every PR | Pre-release / weekly |
| Includes lint? | Yes | No (lint is `/go-review`'s job) |
| Includes tests? | Yes (-short) | No |
| Includes licenses? | No | Yes |
| Includes outdated deps? | No | Yes |

`/go-review` is fast and per-change. `/go-audit` is slower and module-wide.

## Execution flow

### 1. Verify tools

```bash
for tool in govulncheck gosec go-licenses; do
  if ! command -v $tool >/dev/null; then
    echo "MISSING: $tool"
  fi
done
```

Install hints:
```
govulncheck:  go install golang.org/x/vuln/cmd/govulncheck@latest
gosec:        go install github.com/securego/gosec/v2/cmd/gosec@latest
go-licenses:  go install github.com/google/go-licenses@latest
```

If a tool is missing, **note it in the report and continue** — do not block on missing optional tools.

### 2. Run scans

Run each in parallel; capture stdout, stderr, and exit code:

```bash
# Vulnerabilities — only flags vulns in CALLED paths (low false-positive rate)
govulncheck ./...

# Security findings
gosec -quiet -fmt=text ./...

# License audit — fail on disallowed (configure ALLOWED_LICENSES per project)
go-licenses report ./... --template /dev/stdin <<EOF
{{range .}}{{.Name}} {{.LicenseName}}
{{end}}
EOF

# Outdated direct deps (ignore indirect)
go list -u -m -mod=mod -json all | \
  jq -r 'select(.Update != null and .Indirect != true) | "\(.Path) \(.Version) -> \(.Update.Version)"'

# Tidy check (no diff = clean)
go mod tidy -diff || echo "go.mod needs tidying"
```

### 3. Classify license findings

| Category | Examples | Action |
|----------|----------|--------|
| **Permissive (OK)** | MIT, Apache-2.0, BSD-2/3, ISC, Zlib, MPL-2.0 | accept |
| **Weak copyleft (review)** | LGPL-2.1, LGPL-3.0, EPL-2.0 | flag for review |
| **Strong copyleft (block)** | GPL-2.0, GPL-3.0, AGPL-3.0 | block, replace dep |
| **Unknown/missing** | "Unknown", empty | flag for manual lookup |

### 4. Format the report

```markdown
# Audit Report

**Module:** <module path from go.mod>
**Status:** <PASS | WARN | FAIL>
**Tools run:** govulncheck ✓ · gosec ✓ · go-licenses ✓ · outdated ✓ · tidy ✓

## Vulnerabilities (govulncheck)

CRITICAL: <N>
- `GO-2024-XXXX` (CVE-2024-YYYY) in `pkg.Path` — <summary>
  - Called via: <path through your code>
  - Fix: bump to `vX.Y.Z`

## Security findings (gosec)

HIGH: <N>
- `<file>:<line>` — G204 (subprocess with variable) — <message>

## License findings

BLOCKED: <N>
- `github.com/some/pkg` — GPL-3.0 — replace or remove

REVIEW: <N>
- `github.com/other/pkg` — LGPL-2.1 — verify dynamic-link only

UNKNOWN: <N>
- `github.com/third/pkg` — license not detected — confirm manually

## Outdated deps

MAJOR: <N>  (potentially breaking)
- `github.com/x/y v1.2.3 -> v2.0.0`

MINOR: <N>
- `github.com/a/b v1.4.0 -> v1.6.2`

## Module hygiene

- ✓ `go.mod` is tidy
- ✗ `go.mod` not tidy — run `go mod tidy`

## Skipped

- `go-licenses` not installed — install with `go install github.com/google/go-licenses@latest`
```

**PASS / WARN / FAIL rule:**
- **FAIL** if any CRITICAL vuln, HIGH gosec, or BLOCKED license
- **WARN** if any other finding
- **PASS** if no findings

### 5. Don't auto-fix

Like `/go-review`, this is read-only. The user picks remediation:
- For vuln → `go get pkg@vX.Y.Z` then re-run audit
- For gosec → fix the code, optionally suppress with `// #nosec G204 -- justification` if false positive
- For license → swap dep or document exception

## Configuration

If the project has `.go-audit.yml` (or similar) at the module root, prefer those settings:

```yaml
allowed_licenses: [MIT, Apache-2.0, BSD-3-Clause, BSD-2-Clause, ISC, MPL-2.0]
ignore_outdated: [github.com/some/pinned/dep]   # don't suggest update
gosec_exclude: [G104]                            # exclude specific rules
```

If no config, use the defaults above.

## Related

- `/go-review` — per-PR review (lint + tests + vet)
- `${CLAUDE_PLUGIN_ROOT}/references/security.md` — broader security guidance
- `${CLAUDE_PLUGIN_ROOT}/references/ci-cd.md` — wire `/go-audit` checks into CI
