---
name: check-quality
description: |
  Run comprehensive code quality checks for CloudWeGo microservice projects — compilation, linting, test coverage analysis, and architecture compliance verification. Trigger this skill when the user mentions "quality check", "lint", "test coverage", "code review", "check code", "quality gate", "ci check", "architecture compliance", "coverage report", "golangci-lint", "go vet", "run tests", "check build", or any task involving validating code quality, enforcing coding standards, or verifying test coverage targets. Also trigger before merging code or submitting PRs.
argument-hint: "[scope]"
---

# Code Quality Check

Run the full quality gate: compilation, lint, test coverage, and architecture compliance checks. Outputs a structured report with actionable findings.

**Arguments**: `$ARGUMENTS` — optional scope path (e.g. `rpc/identity_srv` or `gateway`). Leave empty to check all modules.

---

## Quality Standards

### Coverage Targets

| Component Type | Minimum | Recommended |
|---------------|---------|-------------|
| Core business logic (`biz/logic/`) | 80% | 90%+ |
| Utility functions | 85% | 95%+ |
| Data access layer (`biz/dal/`) | 70% | 80%+ |
| HTTP/RPC handlers | 60% | 75%+ |
| Middleware | 75% | 85%+ |

**Why these targets differ**: Business logic contains critical branching that directly affects correctness — it needs the highest coverage. Handlers act as thin adapters and are partially covered by integration tests, so a lower threshold is acceptable.

### Architecture Compliance Rules

1. **Layering constraints** — violations indicate coupling that makes testing and refactoring harder:
   - Handler must NOT import DAL directly (go through Logic layer)
   - Logic must NOT import `kitex_gen` directly (use Converter)
   - DAL must NOT import Logic or Handler packages

2. **No manual edits** in `kitex_gen/` — these files are auto-generated and will be overwritten.

3. **Naming conventions**:
   - Interfaces: no `I` prefix (e.g. `UserProfileRepository`)
   - Implementations: `Impl` suffix (e.g. `UserProfileRepositoryImpl`)

4. **Error handling** — consistent error chain enables proper error classification at the Gateway:
   ```
   DAL: errno.WrapDatabaseError() → Logic: errno.ErrNo → Handler: errno.ToKitexError() → Gateway: kerrors.FromBizStatusError()
   ```
   - DAL layer wraps database errors with `errno.WrapDatabaseError()`
   - Never compare against `gorm.ErrRecordNotFound` directly — use the wrapped error

5. **Configuration** — no YAML config files; manage all config through environment variables for container compatibility.

6. **Error codes** — 6-digit business codes in `A-BB-CCC` format, segmented by domain (e.g. `100xxx` = user, `200xxx` = org).

---

## Execution Steps

### 1. Determine scope

- If `$ARGUMENTS` specifies a path, check only that module
- If empty, check both: `rpc/identity_srv` and `gateway`

### 2. Run compilation check

```bash
cd rpc/identity_srv && go build ./...
cd gateway && go build ./...
```

Compilation failures block all further checks. Fix these first.

### 3. Run lint

```bash
cd rpc/identity_srv && golangci-lint run
cd gateway && golangci-lint run
```

Focus on: import ordering, line length (120 chars), unused variables/imports, error handling patterns.

### 4. Run tests and measure coverage

```bash
cd rpc/identity_srv && go test ./... -coverprofile=coverage.out && go tool cover -func=coverage.out | grep total
cd gateway && go test ./... -coverprofile=coverage.out && go tool cover -func=coverage.out | grep total
```

Compare results against the coverage targets above. Per-package breakdown: `go tool cover -func=coverage.out`.

### 5. Check architecture compliance

Search the codebase for violations of the compliance rules:
- Handler importing DAL: `grep -r "dal" handler/` patterns
- Logic importing `kitex_gen`: `grep -r "kitex_gen" logic/` patterns
- Manual edits in `kitex_gen/`: `git diff kitex_gen/`
- Direct `gorm.ErrRecordNotFound` usage: `grep -r "ErrRecordNotFound" biz/`

### 6. Output report

```
## Code Quality Report

### Compilation
- rpc/identity_srv: PASS / FAIL
- gateway: PASS / FAIL

### Lint
- rpc/identity_srv: N warnings
- gateway: N warnings

### Test Coverage
- rpc/identity_srv: XX.X% (target: 80%)
- gateway: XX.X% (target: 80%)

### Architecture Compliance
- [List each violation with file path and rule violated]

### Recommendations
1. [Specific, actionable fix with file path]
```
