---
name: gateway-logging
description: |
  Logging standards for the CloudWeGo/Hertz API gateway. Trigger this skill whenever the user writes log statements in the gateway codebase, adds logging to domain services or middleware, asks about zerolog usage, asks about trace-aware logging, mentions Jaeger span integration with logs, or needs to add diagnostic logging to any gateway layer (domain service, middleware, infrastructure). Covers zerolog + tracelog conventions, log level selection, Jaeger span linkage, and field naming rules.
---

# Gateway Logging Standards

Write log statements in the API gateway following these conventions. All logging uses **zerolog** with the `tracelog` package (`pkg/log/trace_logger.go`) to ensure every log entry carries trace context (trace_id, request_id, span_id).

**Input**: `$ARGUMENTS` — context about what you're logging and where (e.g. "domain service RPC failure", "middleware token validation").

---

## Core Rules

1. **Always use zerolog** via `s.Log(ctx)` (domain services) or `tracelog.Event(ctx, logger)` (middleware).
2. **Never use** `s.Logger()` printf-style API — it is deprecated and lacks trace context.
3. **Always pass `ctx`** — `OTelHook` (`pkg/log/otel_hook.go`) relies on `e.GetCtx()` to inject trace fields and sync to Jaeger. Without ctx, both tracing and Jaeger linkage are silently lost.

---

## Domain Service Layer

Use `s.Log(ctx)` to get a `*zerolog.Logger` pre-loaded with trace info:

```go
s.Log(ctx).Error().Err(err).Msg("RPC call failed")
s.Log(ctx).Info().Str("patient_id", id).Msg("Operation completed")
s.Log(ctx).Warn().Msg("Missing parameter")
```

## Middleware Layer

Use `tracelog.Event(ctx, ...)` to attach context to an existing logger:

```go
import tracelog "gitlab.manteia.com/radius/radius-backend/api/radius-api-gateway/pkg/log"

tracelog.Event(ctx, m.logger.Warn()).
    Str("component", "jwt_middleware").
    Msg("Token has been revoked")
```

**Never** call `m.logger.Warn().Msg(...)` directly without wrapping in `tracelog.Event(ctx, ...)` — doing so bypasses OTelHook and loses all trace correlation.

---

## Log Levels and Jaeger Integration

OTelHook automatically syncs log events to Jaeger spans based on level. Choosing the wrong level has real consequences in production monitoring:

| Level | Jaeger Effect | When to Use |
|-------|---------------|-------------|
| `Error` | `span.RecordError()` + `span.SetStatus(Error)` — **span turns red** | RPC call failures, business logic errors, unexpected errors |
| `Warn` | `span.AddEvent()` — visible in span detail, no red mark | Input validation failures, degraded conditions, retried operations |
| `Info` | Log file only | Successful operations, state changes |
| `Debug` | Log file only | Connection info, diagnostic details |

**Critical**: Do not use `Error` for input validation failures. This turns the Jaeger span red, making it indistinguishable from real failures and triggering false alerts. Use `Warn` instead.

---

## Field Conventions

Use zerolog's typed field methods — they provide structured, queryable log data:

| Data Type | Method | Example |
|-----------|--------|---------|
| Error value | `.Err(err)` | `.Err(err)` — **never** `.Str("error", err.Error())` |
| String | `.Str("key", val)` | `.Str("patient_id", id)` |
| Integer | `.Int("key", val)` | `.Int("retry_count", n)` |
| Component ID | `.Str("component", "xxx_middleware")` | Always identify the source |

Using `.Err(err)` instead of `.Str("error", ...)` ensures zerolog captures the full error stack trace and structured error metadata, which is essential for log aggregation tools.

---

## Quick Reference

```go
// Domain service — good
s.Log(ctx).Error().Err(err).Str("method", "CreateUser").Msg("RPC call failed")

// Domain service — bad (missing ctx, no trace correlation)
s.Logger().Error(err).Msg("RPC call failed")

// Middleware — good
tracelog.Event(ctx, m.logger.Warn()).Str("component", "auth_middleware").Msg("Token expired")

// Middleware — bad (no ctx, trace fields and Jaeger sync lost)
m.logger.Warn().Msg("Token expired")
```
