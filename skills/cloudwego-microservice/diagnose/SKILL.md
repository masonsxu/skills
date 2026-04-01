---
name: diagnose
description: |
  Diagnose and fix errors in CloudWeGo microservice projects — classify errors, locate root causes, and provide actionable fix plans. Trigger this skill when the user mentions "error", "bug", "crash", "fail", "broken", "not working", "debug", "diagnose", "troubleshoot", "fix", "compile error", "runtime error", "wire error", "connection refused", "timeout", "unauthorized", "401", "403", "lint error", "test failure", or pastes any error message, stack trace, or log output from a CloudWeGo/Kitex/Hertz project.
argument-hint: "<error message>"
---

# Error Diagnosis

Classify the error, trace it to a root cause, and deliver a concrete fix plan.

**Arguments**: `$ARGUMENTS` — error message, log excerpt, or problem description.

---

## Architecture Context

```
Error propagation chain:
DAL (errno.WrapDatabaseError) → Logic (errno.ErrNo) → Handler (errno.ToKitexError) → Gateway (kerrors.FromBizStatusError)

Infrastructure ports:
  PostgreSQL:5432  Redis:6379  etcd:2379  RustFS:9000
  Jaeger UI:16686  RPC:8891    Gateway:8080
```

---

## Error Classification

### 1. Compilation Errors

**Signals**: `cannot find package`, `undefined`, `type mismatch`

1. Verify module paths in `go.work` are correct
2. Check `go.mod` dependency versions — mismatched versions cause subtle type errors
3. Determine whether code generation needs re-running (stale `kitex_gen/` or handler stubs)
4. Run `go mod tidy` to resolve missing/transitive dependencies

### 2. Wire DI Errors

**Signals**: `no provider found`, `multiple bindings`, `cycle detected`

1. Inspect `wire/provider.go` — confirm every injected dependency has a corresponding Provider
2. Verify each Provider returns an **interface type** (not a concrete struct) — Wire resolves by type
3. Confirm the Provider Set includes all required dependencies
4. Delete `wire_gen.go` and re-run `wire` to force full regeneration
5. Follow the DI assembly order to spot missing intermediate sets:
   ```
   Gateway: InfrastructureSet → ApplicationSet → DomainServiceSet → MiddlewareSet → ServerSet
   RPC:     InfrastructureSet → ConverterSet → DALSet → CasbinSet → LogicSet → ServerSet
   ```

### 3. Connection / Runtime Errors

**Signals**: `connection refused`, `timeout`, `dial tcp`

1. Check infrastructure status: `podman compose ps` (in the docker/ directory)
2. Verify the target port is listening
3. Confirm environment variables match the deployment context:
   - **Inside containers**: use service names (`DB_HOST=postgres`, `ETCD_ADDRESS=etcd:2379`)
   - **On host machine**: use localhost (`DB_HOST=127.0.0.1`, `ETCD_ADDRESS=127.0.0.1:2379`)
4. Check service registration in etcd: `podman exec etcd etcdctl get --prefix /kitex`

**Why this matters**: Container DNS resolves service names differently than localhost. A mismatch here is the most common cause of `connection refused` in development.

### 4. Lint / Format Errors

**Signals**: `golangci-lint` warnings

1. Fix import ordering: `gci write .`
2. Fix line length: `golines` or `golangci-lint format`
3. Review `.golangci.yml` for project-specific rules

### 5. Code Generation Errors

**Signals**: Kitex/Hertz/Wire generation script failures

1. Validate IDL syntax — a malformed `.thrift` file causes cryptic errors
2. Check tool versions: `kitex --version`, `hz version`
3. Install missing tools:
   ```bash
   go install github.com/cloudwego/kitex/tool/cmd/kitex@latest
   go install github.com/cloudwego/thriftgo@latest
   ```

### 6. Authentication / Authorization Errors

**Signals**: `401 Unauthorized`, `403 Forbidden`, Casbin-related errors

1. **JWT issues** — check token expiry, verify signing key consistency across services, confirm header format is `Authorization: Bearer <token>`
2. **Casbin issues** — follow this diagnostic order:
   - First: check policy sync status (confirm `p/g/g2` tables are loaded — empty tables mean sync failed)
   - Then: check resource mapping (does the requested resource match a policy resource?)
   - Then: check action matching (GET/POST/etc. vs. policy actions)
   - Finally: check subject dimension (does the user's role match?)

**Why this order**: Casbin evaluates policies top-down. If sync failed, nothing else matters. If resources don't map, actions won't match. Starting from sync eliminates entire categories of false debugging.

---

## Environment Variable Reference

| Variable | Default | Purpose |
|----------|---------|---------|
| `DB_HOST` | `127.0.0.1` | PostgreSQL host |
| `DB_PORT` | `5432` | PostgreSQL port |
| `DB_USERNAME` | `postgres` | Database user |
| `DB_PASSWORD` | — | Database password |
| `DB_NAME` | — | Database name |
| `REDIS_HOST` | `127.0.0.1` | Redis host |
| `REDIS_PORT` | `6379` | Redis port |
| `JWT_SIGNING_KEY` | — | Token signing key |
| `JWT_TIMEOUT` | `30m` | Token validity period |
| `JWT_SKIP_PATHS` | — | Paths bypassing auth |
| `CASBIN_ENABLED` | `true` | Enable/disable Casbin |
| `CASBIN_SKIP_PATHS` | `/login,/health` | Paths bypassing authorization |

---

## Execution Steps

### 1. Classify the error

Analyze `$ARGUMENTS` and assign it to one of the 6 categories above.

### 2. Investigate root cause

Follow the diagnostic steps for the matching category. Use `grep`, file reads, and environment checks to narrow down the root cause. Always verify assumptions by reading the actual code rather than guessing.

### 3. Deliver diagnostic report

Structure the output as:

- **Error class**: which of the 6 categories
- **Root cause**: the specific line, config, or missing dependency causing the problem
- **Fix steps**: numbered, copy-paste-ready commands or code changes
- **Prevention**: one concrete action to prevent recurrence
