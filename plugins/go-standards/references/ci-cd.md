# Go Service CI/CD Configuration

## GitLab CI Setup

All Go services use the shared CI configuration from `homelab-ci` repository.

### Basic Configuration

Create `.gitlab-ci.yml` in your project root:

```yaml
include:
  - project: homelab/homelab-ci
    file: golang.yml

variables:
  GO_VERSION: "1.24"
  BINARY_NAME: "service-name"
  COVERAGE_THRESHOLD: "80"
```

### Available Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `GO_VERSION` | `1.24` | Go compiler version |
| `BINARY_NAME` | `service` | Output binary name |
| `COVERAGE_THRESHOLD` | `80` | Minimum test coverage (%) |
| `CGO_ENABLED` | `0` | CGO compilation |
| `GOOS` | `linux` | Target OS |
| `GOARCH` | `amd64` | Target architecture |
| `LINT_TIMEOUT` | `5m` | golangci-lint timeout |
| `BUILD_FLAGS` | `-ldflags='-w -s' -trimpath` | Go build flags |

### Pipeline Stages

```
validate → lint → test → build → security → package
```

1. **validate** - Check project structure and dependencies
2. **lint** - Run golangci-lint
3. **test** - Execute tests with coverage
4. **build** - Compile binary
5. **security** - Security scanning (gosec, nancy, govulncheck)
6. **package** - Build and push Docker image

### For Clean Architecture Services

Use the CA-specific configuration for stricter validation:

```yaml
include:
  - project: homelab/homelab-ci
    file: go-service.yml

variables:
  GO_VERSION: "1.24"
  BINARY_NAME: "service-name"
  COVERAGE_THRESHOLD: "80"
  DOMAIN_COVERAGE_THRESHOLD: "95"
```

This adds:
- CA structure validation
- Layer-specific linting
- Separate tests per layer (domain, application, infrastructure, interface)
- Higher coverage requirements for domain layer

### Docker Image Publishing

Images are automatically published to GitLab Container Registry:

- **Branch builds**: `registry/project:commit-sha`
- **Tag builds**: `registry/project:tag-name`
- **Main branch**: `registry/project:latest`

## Local Development

### Makefile Commands

```bash
# Run tests
make test

# Run tests with coverage
make test-coverage

# Run linter
make lint

# Build binary
make build

# Run locally
make run

# Build Docker image
make docker-build
```

### Pre-commit Hooks

Install pre-commit hooks for local validation:

```bash
# Install pre-commit
pip install pre-commit

# Install hooks
pre-commit install
```

Create `.pre-commit-config.yaml`:

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

  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.5.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-yaml
      - id: check-added-large-files
```

## Coverage Requirements

| Layer | Minimum Coverage |
|-------|-----------------|
| Domain | 95% |
| Application | 80% |
| Infrastructure | 70% |
| Interface | 70% |
| **Overall** | **80%** |

## Security Scanning

The pipeline runs these security tools:

1. **gosec** - Go security checker
2. **nancy** - Dependency vulnerability scanner
3. **govulncheck** - Official Go vulnerability database

Security issues don't block the pipeline but generate reports for review.

## Branch Protection

Recommended branch protection settings:

- Require merge request before merging
- Require pipeline to succeed
- Require code owner approval
- Prevent force push to main
