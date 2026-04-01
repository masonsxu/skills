---
name: codegen
description: |
  Run code generation for CloudWeGo microservice projects — Kitex RPC stubs, Hertz HTTP handlers, and Wire dependency injection wiring. Trigger this skill when the user mentions "codegen", "code generation", "generate code", "kitex", "hertz", "wire", "IDL", "thrift", "run generation", "regenerate", "stub generation", or any task involving generating Go code from IDL definitions or wiring DI containers. Also trigger when adding new RPC methods, HTTP endpoints, or modifying .thrift files.
argument-hint: "[kitex|hertz|wire|all]"
---

# Code Generation

Generate Kitex RPC stubs, Hertz HTTP handlers/routes, and Wire dependency injection code. Supports automatic detection of what needs regeneration based on git changes.

**Arguments**: `$ARGUMENTS` — optional scope: `kitex`, `hertz`, `wire`, or `all`. Leave empty for auto-detection.

---

## Architecture Context

The project uses a two-service architecture with IDL-driven code generation:

```
IDL files (.thrift)
  ├── idl/rpc/   → Kitex generates kitex_gen/ (RPC service stubs)
  └── idl/http/  → Hertz generates biz/handler/, biz/router/, biz/model/ (HTTP layer)

Wire DI assembly order:
  Gateway: InfrastructureSet → ApplicationSet → DomainServiceSet → MiddlewareSet → ServerSet
  RPC:     InfrastructureSet → ConverterSet → DALSet → CasbinSet → LogicSet → ServerSet
```

**Why this matters**: Generated code (`kitex_gen/`) must never be hand-edited — it is the single source of truth from IDL definitions. Wire providers must return interface types (not concrete structs) so DI can swap implementations for testing.

---

## Generation Commands

### Kitex RPC Code

```bash
cd rpc/identity_srv && ./script/gen_kitex_code.sh
```

Outputs to `kitex_gen/`. Never manually edit files in `kitex_gen/`.

### Hertz HTTP Code

```bash
cd gateway && ./script/gen_hertz_code.sh               # Generate all services
cd gateway && ./script/gen_hertz_code.sh identity      # Generate identity service only
```

Outputs to `biz/handler/` and `biz/router/`.

### Wire Dependency Injection

```bash
cd rpc/identity_srv/wire && wire    # RPC service
cd gateway/internal/wire && wire    # Gateway service
```

---

## Execution Steps

### 1. Determine generation scope

Check `$ARGUMENTS`:
- **`kitex`** — run only Kitex generation
- **`hertz`** — run only Hertz generation
- **`wire`** — run only Wire generation
- **`all`** — run Kitex → Hertz → Wire in sequence
- **empty** — auto-detect using `git diff --name-only`:
  - Changes under `idl/rpc/` → run Kitex
  - Changes under `idl/http/` → run Hertz
  - Changes in `wire/` related files → run Wire

### 2. Run each required generation command

Execute the appropriate commands from the Generation Commands section above in order: Kitex first, then Hertz, then Wire. This order matters because Hertz handlers may reference types that Kitex generates.

### 3. Verify generated code compiles

```bash
cd rpc/identity_srv && go mod tidy && go build ./...
cd gateway && go mod tidy && go build ./...
```

These commands catch mismatched interfaces, missing imports, or stale generated files.

### 4. Report results

Summarize: which generators ran, whether each succeeded, and the compilation verification outcome.

---

## Troubleshooting

| Error | Cause | Fix |
|-------|-------|-----|
| `command not found: kitex` | Tool not installed | `go install github.com/cloudwego/kitex/tool/cmd/kitex@latest` |
| `command not found: hz` | Tool not installed | `go install github.com/cloudwego/hertz/cmd/hz@latest` |
| IDL file not found | Wrong working directory | Confirm you are in `rpc/identity_srv/` or `gateway/` |
| `wire: no provider found` | Missing Provider | Check `wire/provider.go` — ensure return type is an interface |
| `wire: cycle detected` | Circular dependency | Review Provider dependency graph for cycles |
