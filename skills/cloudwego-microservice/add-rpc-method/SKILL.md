---
name: add-rpc-method
description: |
  End-to-end guide for adding a new RPC interface method to a CloudWeGo/Kitex microservice — from IDL definition through all-layer implementation (IDL → codegen → DAL → Converter → Logic → Handler → Wire). Trigger this skill whenever the user wants to add a new RPC method, create a new service endpoint, implement a new Thrift IDL method, or extend an existing RPC service. Also trigger on mentions of adding API methods, creating new service operations, implementing new backend endpoints, or any task involving Kitex RPC interface changes in a CloudWeGo project.
argument-hint: "<MethodName>"
---

# Add New RPC Method

End-to-end guide for adding a new RPC interface, covering IDL definition through full-layer implementation.

**Input**: `$ARGUMENTS` — method name or feature description (e.g. `CreateDepartment`)

---

## Architecture Overview

```
RPC Service Layering:
  handler.go → biz/logic/<domain>/ → biz/dal/<domain>/ → models/<domain>.go
                     ↕ biz/converter/<domain>/

Key constraints:
  - Handler must NOT call DAL directly (go through Logic)
  - Logic must NOT import kitex_gen (use Converter)
  - DAL must NOT import Logic or Handler
  - Interface: no I prefix (UserRepository) | Implementation: Impl suffix (UserRepositoryImpl)
  - Error chain: DAL (errno.WrapDatabaseError) → Logic (errno.ErrNo) → Handler (errno.ToKitexError)
  - Error codes: 6-digit, domain-segmented (100xxx=user, 200xxx=org, 300xxx=dept)
  - Import order: stdlib → third-party → internal | Max line: 120 chars
  - DI via Wire; each Provider returns an interface type
```

---

## IDL-First Development Flow

The project strictly follows IDL-First development. The IDL is the contract — all code flows from it:

```
1. Define interface (edit .thrift files under idl/)
2. Generate code  (Kitex/Hertz tools auto-generate stubs)
3. Implement logic (layered implementation under biz/)
4. Test & verify   (unit tests + integration tests)
```

### Layer Code Examples

**Handler** — parameter validation, error translation, response building. The handler is a thin adapter: validate input, delegate to logic, translate errors:

```go
func (s *IdentityServiceImpl) CreateUser(ctx context.Context, req *identity_srv.CreateUserReq) (*identity_srv.CreateUserResp, error) {
    if req.Username == "" {
        return nil, errno.ToKitexError(errno.ErrInvalidParam.WithMessage("username required"))
    }
    userModel := converter.ToUserModel(req)
    createdUser, err := s.userLogic.CreateUser(ctx, userModel)
    if err != nil {
        return nil, errno.ToKitexError(err)
    }
    return &identity_srv.CreateUserResp{User: converter.ToUserDTO(createdUser)}, nil
}
```

**Logic** — core business logic, orchestrates DAL operations. Returns `errno.ErrNo` so Handler can translate:

```go
func (l *UserLogic) CreateUser(ctx context.Context, user *models.User) (*models.User, error) {
    if err := l.validateUser(user); err != nil {
        return nil, err
    }
    hashedPassword, err := bcrypt.GenerateFromPassword([]byte(user.Password), bcrypt.DefaultCost)
    if err != nil {
        return nil, errno.ErrInternalServer
    }
    user.PasswordHash = string(hashedPassword)
    if err := l.userDAL.Create(ctx, user); err != nil {
        return nil, err
    }
    return user, nil
}
```

**DAL** — data persistence, GORM wrapper. Wraps all database errors so upper layers never see raw GORM errors:

```go
func (d *UserDAL) Create(ctx context.Context, user *models.User) error {
    if err := d.db.WithContext(ctx).Create(user).Error; err != nil {
        if errors.Is(err, gorm.ErrDuplicatedKey) {
            return errno.ErrUserAlreadyExists
        }
        return errno.ErrDatabaseOperation.WithCause(err)
    }
    return nil
}
```

**Converter** — pure Model ↔ DTO functions. No side effects, no framework dependencies:

```go
func ToUserModel(req *identity_srv.CreateUserReq) *models.User {
    return &models.User{Username: req.Username, Email: req.Email, PhoneNumber: req.PhoneNumber}
}
func ToUserDTO(user *models.User) *identity_srv.User {
    return &identity_srv.User{Id: user.ID, Username: user.Username, Email: user.Email, CreatedAt: user.CreatedAt.Unix()}
}
```

---

## Execution Steps

### Step 1: Confirm Requirements

Clarify based on `$ARGUMENTS`: method name (PascalCase), business domain, request and response field design.

### Step 2: Define the IDL

Open the relevant Thrift IDL file under `idl/rpc/identity_srv/`. Add request/response structs and the Service method definition. Follow the naming and structure style of existing IDL definitions.

### Step 3: Generate Code

```bash
cd rpc/identity_srv && ./script/gen_kitex_code.sh
```

Verify the new method appears under `kitex_gen/`. **Never manually edit `kitex_gen/`** — it is auto-generated and will be overwritten on the next generation run.

### Step 4: Implement Layer by Layer

#### 4.1 DAL Layer
- Extend the interface and implementation in `biz/dal/<domain>/`
- Wrap database errors with `errno.WrapDatabaseError()`
- Check for record-not-found with `errno.IsRecordNotFound(err)`
- Ensure new interface methods appear in the `biz/dal/dal.go` aggregate interface

#### 4.2 Converter Layer
- Add pure Model ↔ Thrift DTO conversion functions
- Update `biz/converter/converter.go`

#### 4.3 Logic Layer
- Orchestrate DAL calls and Converter transformations
- Return `errno.ErrNo` type errors for Handler translation
- Update `biz/logic/logic.go`

#### 4.4 Handler Layer
- Use the standard pattern: `s.logic.Xxx(ctx, req)` + `errno.ToKitexError(err)`
- Keep handlers thin — validate, delegate, translate errors

### Step 5: Wire Dependency Injection

Update `rpc/identity_srv/wire/provider.go` if new providers are needed. Run:

```bash
cd rpc/identity_srv/wire && wire
```

### Step 6: Verify

```bash
cd rpc/identity_srv && go build ./...
cd rpc/identity_srv && go vet ./...
cd rpc/identity_srv && golangci-lint run
```

### Step 7: Write Tests

- **Converter**: pure function tests (no mocks needed)
- **Logic**: mock DAL with gomock, verify business logic
- Test naming: `Test<InterfaceName>_<MethodName>`
