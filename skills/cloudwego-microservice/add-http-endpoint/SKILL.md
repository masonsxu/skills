---
name: add-http-endpoint
description: |
  Add a new HTTP endpoint to the CloudWeGo/Hertz API gateway (IDL → codegen → handler → domain service → assembler → Wire DI → Casbin permissions). Trigger this skill whenever the user wants to add a REST API endpoint, create a new HTTP route, expose an RPC method via the gateway, or mentions HTTP methods (GET/POST/PUT/DELETE) in the context of the CloudWeGo/Hertz gateway. Also covers Hertz Thrift IDL definition, Hertz code generation, gateway domain service implementation, DTO assembler creation, Wire dependency injection, and Casbin RBAC permission configuration.
argument-hint: "<METHOD /api/v1/path>"
---

# Add HTTP Endpoint

End-to-end guide for adding a new HTTP endpoint to the gateway. The corresponding RPC interface must already exist in `rpc/identity_srv/`. If it doesn't, prompt the user to run `add-rpc-method` first.

**Input**: `$ARGUMENTS` — HTTP method and path, e.g. `POST /api/v1/departments`

---

## Architecture Overview

```
gateway/
├── biz/handler/                # HTTP handlers (IDL-generated stubs)
├── biz/model/                  # HTTP DTOs (IDL-generated)
├── biz/router/                 # Route registration (IDL-generated)
├── internal/
│   ├── application/assembler/  # DTO conversion: RPC ↔ HTTP
│   ├── domain/service/         # Domain services (orchestrate RPC calls)
│   ├── infrastructure/client/  # RPC clients
│   └── wire/                   # Wire DI
```

**Request flow**: Handler (param binding) → Domain Service (business logic + RPC) → Assembler (DTO conversion) → HTTP response.

**Error chain**: RPC errors are caught in domain services via `kerrors.FromBizStatusError(err)`, then converted with `errors.ProcessRPCError(bizErr)`.

**Middleware order** (executed left-to-right):
```
OTel Tracing → RequestID → ResponseHeader → Trace → CORS → ErrorHandler → JWT → Casbin → ETag
```

### Casbin RBAC

The project uses **Casbin RBAC** with a five-dimensional model (user, role, resource, action, data scope). HTTP methods map to actions:

```go
var methodActionMap = map[string]string{
    "GET": "read", "POST": "write", "PUT": "write", "DELETE": "manage",
}
```

Data scope hierarchy: `self` → `dept` → `org` (max wins across roles).

| Env Var | Purpose | Default |
|---------|---------|---------|
| `CASBIN_ENABLED` | Enable Casbin | `true` |
| `CASBIN_SKIP_PATHS` | Paths to skip | `/login,/health` |
| `CASBIN_SYNC_INTERVAL` | Policy sync interval | `5m` |

---

## Steps

### 1. Confirm Requirements

Confirm with the user:
- HTTP method (GET/POST/PUT/DELETE)
- Route path (e.g. `/api/v1/departments`)
- Corresponding RPC method and service domain
- Whether authentication and Casbin permissions are needed

### 2. Define the HTTP IDL

Open the relevant Thrift IDL file under `idl/http/<service>/`. Define request/response structs and add the HTTP method annotation with route path. This IDL drives all code generation in subsequent steps.

### 3. Generate Code

Run the Hertz code generator. Pass `identity`, `permission`, or nothing (generates all):

```bash
cd gateway && ./script/gen_hertz_code.sh <service>
```

This regenerates `biz/handler/`, `biz/model/`, and `biz/router/` from the IDL. The handler will contain a stub method for you to fill in.

### 4. Implement the Domain Service

In `gateway/internal/domain/service/<service>/`:

1. Add the new method to the aggregate interface file.
2. Implement the method in the implementation file — call the RPC client, orchestrate business logic.
3. Handle RPC errors using the standard pattern:

```go
resp, err := s.client.SomeMethod(ctx, req)
if err != nil {
    bizErr := kerrors.FromBizStatusError(err)
    return nil, errors.ProcessRPCError(bizErr)
}
```

The domain service is where all business decisions live. Handlers should never contain business logic.

### 5. Create the Assembler

In `gateway/internal/application/assembler/<service>/`:

1. Add the conversion method to the interface file.
2. Implement the conversion in the implementation file (RPC DTO → HTTP DTO or vice versa).
3. Update the aggregate file `assembler.go` if it exports the new method.

Assemblers keep the mapping logic isolated so handlers and domain services stay focused on their own concerns.

### 6. Complete the Handler

In `gateway/biz/handler/<service>/`:

Fill in the generated handler stub. The handler should only:
- Bind and validate the HTTP request parameters.
- Call the domain service.
- Use the assembler to convert the response.
- Return the HTTP response.

Do **not** put business logic or direct RPC calls in handlers.

### 7. Wire Dependency Injection

Update `gateway/internal/wire/provider.go` if new providers are needed (e.g. a new domain service or assembler). Then regenerate:

```bash
cd gateway/internal/wire && wire
```

Wire ensures all dependencies are injected at compile time, catching wiring errors before runtime.

### 8. Configure Permissions

If the endpoint requires Casbin authorization:
- Add the appropriate role → resource → action mapping to the Casbin policy.
- Verify that the middleware chain applies JWT authentication before Casbin (see middleware order above).
- Test with `CASBIN_ENABLED=true` and confirm unauthorized requests are rejected.

### 9. Verify

```bash
cd gateway && go build ./...
cd gateway && go vet ./...
cd gateway && golangci-lint run
```

All three commands must pass with zero errors before the endpoint is considered complete.
