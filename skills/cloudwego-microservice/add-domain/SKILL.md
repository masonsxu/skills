---
name: add-domain
description: |
  Create a new business domain module skeleton in the CloudWeGo RPC service (Model→DAL→Converter→Logic→Wire).
  Use this skill whenever the user wants to add a new domain, create a new business module, scaffold a new feature area,
  or mentions creating a new entity/service/module in the RPC service. Also trigger when the user mentions
  domain-driven design patterns, repository patterns, clean architecture scaffolding, or adding a new database entity
  in the context of CloudWeGo/Kitex/Hertz microservices. Triggers on keywords: new domain, new module, new entity,
  scaffold, skeleton, create domain, add domain, new feature module, new business area.
argument-hint: <domain-name>
---

# Add New Business Domain Module

Scaffold a complete business domain directory structure and boilerplate code in the RPC service, integrating it into all three aggregate interfaces.

**User input**: `$ARGUMENTS` — the domain name (e.g., `notification`, `audit`, `tenant`)

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

## Step-by-Step Instructions

### Step 1: Confirm Requirements

Clarify with the user based on `$ARGUMENTS`:
- **Domain name** (lowercase, for package and directory names)
- **Entity name** (PascalCase, for interface/struct naming)
- **Primary entity fields** (to shape the Model and Repository)
- **Relationships** with existing domains (e.g., belongs to `user`, `organization`)

### Step 2: Create the Data Model

Create `rpc/identity_srv/models/<domain>.go`. Match the existing Model files' GORM tag style:

```go
type Notification struct {
    ID        int64          `gorm:"primaryKey;autoIncrement"`
    UserID    int64          `gorm:"index;not null"`
    Content   string         `gorm:"type:text;not null"`
    ReadAt    gorm.DeletedAt `gorm:"index"`
    CreatedAt time.Time      `gorm:"autoCreateTime"`
    UpdatedAt time.Time      `gorm:"autoUpdateTime"`
}
```

Include standard fields (ID, CreatedAt, UpdatedAt). Use `gorm:"primaryKey"` and `gorm:"index"` tags as appropriate.

### Step 3: Create the DAL Layer

Create `rpc/identity_srv/biz/dal/<domain>/` with two files:

1. **Interface file** `<entity>_interface.go` — Define `<Entity>Repository` with CRUD + domain-specific query methods. This interface is what Logic depends on, enabling testability via mocks.

2. **Implementation file** `<entity>_repository.go` — Create `<Entity>RepositoryImpl` struct with `*gorm.DB` as its dependency. Wrap all GORM errors with `errno.WrapDatabaseError()` so the error chain is preserved and can be translated to business error codes at upper layers:

```go
func (r *NotificationRepositoryImpl) Create(ctx context.Context, n *models.Notification) error {
    if err := r.db.WithContext(ctx).Create(n).Error; err != nil {
        return errno.WrapDatabaseError(err)
    }
    return nil
}
```

3. **Register in aggregate** — Add `<Entity>() <Entity>Repository` method to `biz/dal/dal.go` interface, and its implementation in `biz/dal/dal_impl.go`. This keeps all DAL access behind a single aggregate interface for Wire injection.

### Step 4: Create the Converter Layer

Create `rpc/identity_srv/biz/converter/<domain>/` with two files:

1. **Interface file** `<entity>.go` — Define `<Entity>Converter` for Model ↔ DTO bidirectional conversion.

2. **Implementation file** `<entity>_impl.go` — Implement `<Entity>ConverterImpl` as **pure functions with no side effects**. Converter isolates Logic from `kitex_gen` types so business logic stays framework-agnostic:

```go
func (c *NotificationConverterImpl) ToDTO(n *models.Notification) *identity_srv.Notification {
    return &identity_srv.Notification{Id: n.ID, Content: n.Content, CreatedAt: n.CreatedAt.Unix()}
}
```

3. **Register in aggregate** — Update `biz/converter/converter.go` to include the new converter.

### Step 5: Create the Logic Layer

Create `rpc/identity_srv/biz/logic/<domain>/` with two files:

1. **Interface file** `<entity>_logic.go` — Define `<Entity>Logic` with business operation methods.

2. **Implementation file** `<entity>_logic_impl.go` — Create `<Entity>LogicImpl` that receives DAL and Converter via dependency injection. This layer orchestrates DAL calls and Converter transformations. Return `errno.ErrNo` errors so Handler can translate them:

```go
type NotificationLogicImpl struct {
    notificationDAL       dal.NotificationRepository
    notificationConverter converter.NotificationConverter
}

func (l *NotificationLogicImpl) CreateNotification(ctx context.Context, n *models.Notification) (*models.Notification, error) {
    if err := l.notificationDAL.Create(ctx, n); err != nil {
        return nil, err
    }
    return n, nil
}
```

3. **Register in aggregate** — Update `biz/logic/logic.go` to embed the new logic interface.

### Step 6: Wire Dependency Injection

Add a new Provider function in `rpc/identity_srv/wire/provider.go`. Each Provider must create exactly one dependency and return the **interface type** (not the concrete type) so Wire can resolve the dependency graph:

```go
func ProvideNotificationRepository(db *gorm.DB) dal.NotificationRepository {
    return dal.NewNotificationRepositoryImpl(db)
}
```

Run code generation:

```bash
cd rpc/identity_srv/wire && wire
```

### Step 7: Verify

```bash
cd rpc/identity_srv && go build ./...
cd rpc/identity_srv && go vet ./...
```

---

## Output Checklist

After completion, list all created and modified files:
- **New files** (6–8): model, DAL interface + impl, converter interface + impl, logic interface + impl
- **Modified files** (3–4): `biz/dal/dal.go`, `biz/dal/dal_impl.go`, `biz/converter/converter.go`, `biz/logic/logic.go`, optionally `wire/provider.go`
