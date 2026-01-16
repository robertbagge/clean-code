# Design Checklist — ESP32 Firmware

Quick validation before shipping. Run through this checklist before creating a
PR.

---

## Memory Safety

- [ ] No raw `new`/`delete` — using `std::unique_ptr` or `std::make_unique`
- [ ] No raw C-style arrays for dynamic data — using `std::vector` or
      `std::array`
- [ ] Stack allocations are bounded (no VLAs, no large stack buffers)
- [ ] RAII wrappers for any C resources (file handles, mutexes, etc.)

## Error Handling

- [ ] `esp_err_t` checked at all ESP-IDF call sites
- [ ] Domain errors defined and used (not leaking `ESP_ERR_*` to business logic)
- [ ] Error context logged before propagating
- [ ] No silent failures — errors either handled or propagated

## Testability

- [ ] External dependencies injected via constructor (Clock, Logger, hardware)
- [ ] Pure virtual interfaces for mockable components
- [ ] No direct ESP-IDF calls in business logic (wrapped in abstractions)
- [ ] Tests run on host without ESP-IDF runtime

## C Isolation

- [ ] ESP-IDF C APIs wrapped in C++ classes
- [ ] `extern "C"` only at component boundaries, not in business logic
- [ ] C `#define` constants wrapped in `enum class` or `constexpr`
- [ ] C callback patterns wrapped with type-safe C++ alternatives

## FreeRTOS Patterns

- [ ] Tasks have clear ownership and lifecycle
- [ ] Queues used for inter-task communication (not shared state)
- [ ] ISR-safe functions used from interrupt context
- [ ] No blocking calls in ISRs

## Code Quality

- [ ] Single responsibility — each class/function does one thing
- [ ] Naming follows conventions (PascalCase classes, camelCase methods)
- [ ] No magic numbers — constants named and documented
- [ ] clang-format applied (`task format` or pre-commit hook)

## Documentation

- [ ] Public APIs have brief documentation
- [ ] Complex algorithms have inline comments explaining "why"
- [ ] Component README exists for non-trivial components
