Generate a table-driven test for a Go function or method, following project standards (testify, subtests, edge cases, modern stdlib).

## Usage

```
/go-gen-test <path>                       # generate tests for all exported functions in a file
/go-gen-test <path> <FuncName>            # generate test for a specific function/method
/go-gen-test <path> <Type>.<Method>       # generate test for a method on a type
```

Examples:
```
/go-gen-test domain/valueobject/email.go
/go-gen-test domain/valueobject/email.go NewEmail
/go-gen-test application/usecase/create_user.go CreateUserUseCase.Execute
```

## Execution flow

### 1. Read the target

- Read the source file at `<path>`.
- Identify the target function(s)/method(s).
- Note the signature: parameter types, return types, errors.
- Read existing `_test.go` in the same package (if any) to match style and avoid duplicating tests.
- Check `go.mod` for the Go version — affects whether to use `t.Context()`, `for b.Loop()`, etc. (see `${CLAUDE_PLUGIN_ROOT}/skills/go-modernize/SKILL.md`).

### 2. Identify the test layer

Match the layer to the patterns in `${CLAUDE_PLUGIN_ROOT}/references/testing.md`:

| File location | Test pattern |
|---------------|--------------|
| `domain/...` | Pure unit, no mocks, target ≥95% coverage |
| `application/usecase/...` | Mock at boundaries (repos, ports), target ≥80% |
| `infrastructure/persistence/...` | Mock DB OR use testcontainers integration test |
| `interface/http/handler/...` | `httptest` + `gin.TestMode`, target ≥70% |

### 3. Identify edge cases

For each parameter, derive cases:

| Type | Cases to consider |
|------|-------------------|
| `string` | empty, whitespace, valid, max-length, unicode, with control chars |
| Numeric | zero, negative, max, min, overflow, precision boundaries |
| Slice/map | nil, empty, one element, many, with duplicates |
| Pointer | nil, non-nil |
| `time.Time` | zero, past, future, UTC vs local |
| Error param | nil, sentinel error, wrapped error |
| `context.Context` | background, cancelled, with timeout already expired |

For domain validators specifically, generate at least one **invalid** case per validation rule.

### 4. Generate the test file

If `<path>` is `domain/valueobject/email.go`, write to `domain/valueobject/email_test.go`. **Don't overwrite an existing test** — add new test functions or extend existing tables. Confirm with the user before merging into an existing test file.

Template (Go 1.24+):

```go
package <pkg>_test

import (
	"testing"

	"github.com/stretchr/testify/assert"
	"github.com/stretchr/testify/require"

	"<module>/path/to/pkg"
)

func Test<Func>(t *testing.T) {
	t.Parallel()

	tests := []struct {
		name    string
		input   <T>
		want    <T>
		wantErr error  // or bool wantErr if just checking presence
	}{
		{
			name:  "valid",
			input: <valid value>,
			want:  <expected>,
		},
		{
			name:    "empty",
			input:   <zero value>,
			wantErr: pkg.ErrInvalidInput,
		},
		// ... more cases
	}

	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			t.Parallel()

			got, err := pkg.<Func>(tt.input)

			if tt.wantErr != nil {
				require.Error(t, err)
				assert.ErrorIs(t, err, tt.wantErr)
				return
			}

			require.NoError(t, err)
			assert.Equal(t, tt.want, got)
		})
	}
}
```

Adjust per:
- **Method on type**: receiver setup before each subtest (e.g., `u := mustCreateUser(t, ...)`).
- **Use case with mocks**: `setupMock func(*MockRepo)` field in struct, pre-condition mocks per case.
- **HTTP handler**: `httptest.NewRecorder()` + `httptest.NewRequest()` + assert status/body.
- **Context-using function**: pass `t.Context()` (Go 1.24+) or `context.Background()`.

### 5. Conventions to enforce

- **`t.Parallel()`** in both the outer test and each subtest unless there's shared mutable state.
- **`require`** for fatal preconditions (test setup); **`assert`** for expectations.
- **Assertions on chained returns**: assert single thing per `assert.*` call.
- **Don't use `assert.True(t, err == nil)`** — use `require.NoError`.
- **Don't use `==` for errors** — use `assert.ErrorIs` / `assert.ErrorAs`.
- **Helpers**: extract `mustCreate*` factories and call `t.Helper()` in them.
- **Mock setup inline** in the case struct (`setupMock` func), not in nested loops.

### 6. After generation

```bash
# Show what was created
git diff <path that was created>

# Verify it compiles and passes
go test ./<package> -run "Test<Func>" -v -race
```

If a case fails, the case is wrong, **not the implementation** — adjust the case (or report a real bug to the user).

## Anti-recommendations

- **Don't mock things in the same package** as the function under test. Domain functions test against real domain objects.
- **Don't generate trivial tests** (`func TestNew(t *testing.T) { _ = New() }`) — they pad coverage but verify nothing.
- **Don't generate one giant test** with 30 sub-cases for unrelated behaviors. Split by behavior.
- **Don't add a `setupMock` parameter** if the function takes no dependencies. Keep the table struct minimal.

## Related

- `${CLAUDE_PLUGIN_ROOT}/references/testing.md` — full testing patterns
- `${CLAUDE_PLUGIN_ROOT}/skills/go-testing/SKILL.md` — testing skill (auto-active)
- `${CLAUDE_PLUGIN_ROOT}/skills/go-modernize/SKILL.md` — version-appropriate idioms (e.g., `t.Context()`)
- `/go-review` — verify generated tests pass linting
