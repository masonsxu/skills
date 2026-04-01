---
name: write-tests
description: |
  Write Go unit tests following project conventions for CloudWeGo microservices. Trigger this skill whenever you need to write tests, add test cases, improve test coverage, create mocks, or debug failing tests in a Go project using testify, gomock, or table-driven patterns. Also trigger when you see test file naming patterns like *_test.go, when the user mentions "test", "coverage", "mock", or "unit test" in the context of Go code. Covers all layers: handler, logic, DAL, converter, and middleware.
argument-hint: "<target-path>"
---

# Write Tests

Write unit tests for `$ARGUMENTS` — a target Go package path (e.g. `rpc/identity_srv/biz/logic/department/`). Follow the project's layered architecture conventions to determine the correct testing strategy.

**Input**: `$ARGUMENTS` — path to the code under test.

---

## Architecture

```
RPC Service Layering:
  handler.go → biz/logic/ → biz/dal/ → models/
                     ↕ biz/converter/
```

| Layer | Strategy | Mock Dependencies |
|-------|----------|-------------------|
| Converter | Pure function tests | None — pure input/output |
| Logic | Mock DAL + Converter | `TestMocks` aggregate struct |
| Handler | Mock Logic | Kitex test utilities |
| DAL | Integration tests | Test database or mock GORM |
| Middleware | HTTP context tests | Construct `*app.RequestContext` |

---

## Conventions

### Naming

- **Files**: `<source>_test.go` in the same directory as the source file.
- **Functions**: `Test<InterfaceName>_<MethodName>` or `Test<InterfaceName>_<MethodName>_<Scenario>`.
- **Structure**: Always use table-driven tests with `t.Run(tt.name, ...)`.

These naming conventions make test failures immediately identifiable — you know exactly which interface method and scenario failed from the test name alone.

### Assertions

Use `testify/assert` for non-fatal checks and `testify/require` for hard stops (e.g. verifying the error type before inspecting its fields). Using `require` early prevents cascading panics from nil dereferences.

### Mocks

The project uses `go.uber.org/mock/gomock` v0.6.0. All mocks live in `biz/mock/`.

The `TestMocks` aggregate struct provides a unified entry point for all mock dependencies. This avoids manually creating each mock and wiring them together — call `mock.NewTestMocks(ctrl)` once to get everything:

```go
func setupTest(t *testing.T) (*LogicImpl, *mock.TestMocks) {
    t.Helper()
    ctrl := gomock.NewController(t)
    mocks := mock.NewTestMocks(ctrl)
    logic := &LogicImpl{
        dal:       mocks.DAL,
        converter: mocks.Converter,
    }
    return logic, mocks
}

func assertErrCode(t *testing.T, expected errno.ErrNo, actual error) {
    t.Helper()
    errNo, ok := actual.(errno.ErrNo)
    require.True(t, ok, "expected errno.ErrNo, got %T: %v", actual, actual)
    assert.Equal(t, expected.ErrCode, errNo.ErrCode)
}
```

Key behaviors of `TestMocks`:
- `mock.NewTestMocks(ctrl)` creates MockDAL and all sub-repository mocks, configuring `AnyTimes()` accessors.
- `WithTransaction` is stubbed to execute the callback directly — no real transaction overhead.
- `converter` uses the real `converter.NewConverter()` since it is pure functions with no side effects.

---

## Steps

### 1. Analyze the target code

Read `$ARGUMENTS` and identify:
- Which layer the code belongs to (converter, logic, handler, DAL, middleware).
- What interfaces it depends on.
- What error paths and edge cases exist.

### 2. Select the testing strategy

#### Converter layer — pure function tests

No mocks needed. Test input→output directly. Cover: normal conversion, nil/zero inputs, boundary values.

```go
func TestToUserModel(t *testing.T) {
    t.Run("normal conversion", func(t *testing.T) {
        req := &identity_srv.CreateUserReq{Username: "test", Email: "test@example.com"}
        model := ToUserModel(req)
        assert.Equal(t, "test", model.Username)
    })
}
```

#### Logic layer — mock DAL

Mock all DAL calls, verify business logic in isolation. Cover: happy path, validation failures, DAL error propagation, concurrent safety if applicable.

```go
func TestLogicImpl_CreateDepartment(t *testing.T) {
    t.Run("success", func(t *testing.T) {
        logic, mocks := setupTest(t)
        ctx := context.Background()

        mocks.OrgRepo.EXPECT().ExistsByID(ctx, orgID).Return(true, nil)
        mocks.DeptRepo.EXPECT().Create(gomock.Any(), gomock.Any()).Return(nil)

        result, err := logic.CreateDepartment(ctx, req)
        require.NoError(t, err)
        assert.NotNil(t, result)
    })
}
```

For table-driven logic tests, use a struct slice with descriptive names:

```go
func TestLogicImpl_CreateDepartment(t *testing.T) {
    tests := []struct {
        name    string
        setup   func(mocks *mock.TestMocks)
        req     *CreateDepartmentReq
        wantErr bool
        errCode int
    }{
        {
            name:    "org not found",
            setup:   func(m *mock.TestMocks) { m.OrgRepo.EXPECT().ExistsByID(gomock.Any(), gomock.Any()).Return(false, nil) },
            wantErr: true,
            errCode: 200001,
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            logic, mocks := setupTest(t)
            tt.setup(mocks)
            _, err := logic.CreateDepartment(context.Background(), tt.req)
            if tt.wantErr {
                assertErrCode(t, errno.ErrNo{ErrCode: tt.errCode}, err)
            } else {
                require.NoError(t, err)
            }
        })
    }
}
```

#### Middleware layer — HTTP context tests

Construct `*app.RequestContext` manually and verify middleware behavior (auth, logging, CORS, etc.).

#### DAL layer — integration tests

Use a test database or mock GORM. These are optional and depend on project setup.

### 3. Run tests and verify coverage

```bash
go test <target_package> -v -count=1
go test <target_package> -coverprofile=coverage.out
go tool cover -func=coverage.out
```

### 4. Run linter

```bash
golangci-lint run <target_path>
```

---

## Coverage Targets

| Component | Minimum | Recommended |
|-----------|---------|-------------|
| Core business logic | 80% | 90%+ |
| Utility functions | 85% | 95%+ |
| Data access layer | 70% | 80%+ |
| HTTP/RPC Handler | 60% | 75%+ |
| Middleware | 75% | 85%+ |

**Why these differ**: Business logic contains critical branching that directly affects correctness — it needs the highest coverage. Handlers are thin adapters partially covered by integration tests, so a lower threshold is acceptable.
