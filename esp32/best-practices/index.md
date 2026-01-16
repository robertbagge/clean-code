# Best Practices — Index

Practical patterns and conventions for ESP32 firmware. These complement the
[clean code principles](../clean-code/) with concrete guidance.

## Core Patterns

* [memory-management.md](./memory-management.md) — RAII, smart pointers, stack
  vs heap, avoiding raw `new`/`delete`.
* [error-handling.md](./error-handling.md) — `esp_err_t` patterns, domain
  errors, recovery strategies, when to use exceptions.
* [testing.md](./testing.md) — Host-based googletest, mock patterns, dependency
  injection for testability.
* [c-isolation.md](./c-isolation.md) — Wrapping ESP-IDF C APIs in C++ classes,
  type-safe wrappers, boundary isolation.

## Platform Patterns

* [freertos-patterns.md](./freertos-patterns.md) — Task design, queue-based
  commands, ISR safety, timing patterns.
* [hardware-abstraction.md](./hardware-abstraction.md) — HAL layers, GPIO
  abstractions, driver encapsulation.

## Structure & Style

* [component-structure.md](./component-structure.md) — ESP-IDF component
  layout, CMakeLists.txt, header organization.
* [naming-style.md](./naming-style.md) — PascalCase classes, camelCase methods,
  `m_` prefix, clang-format conventions.
