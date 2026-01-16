# Clean Code – ESP32

## What this is

A practical, opinionated guide to clean C++ for ESP32 firmware: SOLID-style
principles adapted for embedded development and best practices for ESP-IDF
projects.

## Who it's for

Engineers and AI agents building or refactoring ESP32 firmware in C++.

## Scope & portability

### General C++ Layer

These patterns apply broadly to embedded C++ development:

* Modern C++ idioms (C++20)
* RAII and smart pointers
* Dependency injection
* Pure virtual interfaces for abstraction

### ESP-IDF Specific Layer

These patterns are specific to ESP-IDF projects:

* `esp_err_t` return codes and error handling
* Component structure and CMake integration
* ESP logging macros (`ESP_LOGI`, `ESP_LOGE`)
* FreeRTOS task and queue patterns

### Opinions

We have established patterns for:

* **Memory management**: RAII with `std::unique_ptr` — see
  [memory-management.md](./best-practices/memory-management.md)
* **Error handling**: `esp_err_t` at boundaries, domain errors internally — see
  [error-handling.md](./best-practices/error-handling.md)
* **Testing**: Host-based googletest with dependency injection — see
  [testing.md](./best-practices/testing.md)
* **C isolation**: Wrap ESP-IDF C APIs in C++ classes — see
  [c-isolation.md](./best-practices/c-isolation.md)

No strong opinions yet on:

* Specific FreeRTOS task architectures
* Power management patterns
* OTA update strategies

## How to use this guide

### For AI agent research

Unless otherwise instructed, read all documents to get a full picture of best
practices & clean code before distilling for your task.

### For implementation guidance

Lead with the [Best Practices](./best-practices/), then apply these principles
first:
[DIP](./clean-code/dependency-inversion.md),
[SRP](./clean-code/single-responsibility.md),
[ISP](./clean-code/interface-segregation.md),
[DRY](./clean-code/dry.md),
[KISS](./clean-code/kiss.md).

Refer to [OCP](./clean-code/open-closed.md) and
[LSP](./clean-code/liskov-substitution.md) as needed.

## Clean code principles

* [Single Responsibility (SRP)](./clean-code/single-responsibility.md)
  – One reason to change per class/function/component.
* [Interface Segregation (ISP)](./clean-code/interface-segregation.md)
  – Small, capability-focused pure virtual interfaces.
* [Dependency Inversion (DIP)](./clean-code/dependency-inversion.md)
  – Depend on abstractions; inject implementations for testability.
* [Open–Closed (OCP)](./clean-code/open-closed.md)
  – Add behavior via new implementations, not by editing existing code.
* [Liskov Substitution (LSP)](./clean-code/liskov-substitution.md)
  – Derived classes honor base class contracts.
* [Don't Repeat Yourself (DRY)](./clean-code/dry.md)
  – Centralize rules and cross-cutting utilities.
* [Keep It Simple (KISS)](./clean-code/kiss.md)
  – Prefer the simplest working design; avoid premature abstraction.

## Check yourself before you ship

Use the quick [design-checklist.md](./design-checklist.md) before shipping.
