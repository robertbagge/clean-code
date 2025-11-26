# Best Practices — Index

Practical patterns and conventions for Go services. These complement the
[clean code principles](../clean-code/) with concrete guidance.

## Core Patterns

* [interfaces.md](./interfaces.md) — Named vs anonymous interfaces; where to
  define them; composition patterns.
* [error-handling.md](./error-handling.md) — Sentinel errors, mapping driver
  errors to domain errors, wrapping with context.
* [testing.md](./testing.md) — Black-box testing, fakes over mocks,
  table-driven tests, golden file patterns.
* [package-structure.md](./package-structure.md) — Domain/service/repo layout;
  dependency direction; inward-pointing graph.
* [go-style.md](./go-style.md) — General Go idioms: accept interfaces, return
  structs; concrete types for config/DTOs.

## Infrastructure Patterns

* [logging.md](./logging.md) — Structured logging with slog; JSON handlers;
  contextual attributes.
* [database.md](./database.md) — Type-safe queries with sqlc; configuration;
  connection pooling.
* [repository.md](./repository.md) — Domain-scoped repositories; DB-to-domain
  mapping; compile-time verification.
* [configuration.md](./configuration.md) — Environment-based config; required
  vs optional vars; pool tuning for cloud.
