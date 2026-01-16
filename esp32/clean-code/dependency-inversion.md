# Dependency Inversion Principle (DIP) in C++

## Overview

The Dependency Inversion Principle states that high-level modules should not
depend on low-level modules; both should depend on abstractions. In C++, this
is achieved through pure virtual base classes (interfaces), allowing business
logic to depend on abstract contracts rather than concrete implementations.
This makes code testable on the host without ESP-IDF dependencies.

## Core Concept

In embedded C++, DIP means:

* Depend on abstract interfaces, not concrete ESP-IDF implementations
* High-level business logic defines the interfaces it needs
* Low-level implementations (ESP32-specific) satisfy those interfaces
* Use constructor injection to provide implementations
* Enable host-based unit testing with mock implementations

## Implementation Example

### BAD — Direct Hardware Dependency (DIP violation)

> Business logic directly uses ESP-IDF timing functions, making it untestable
> without hardware.

```cpp
#include "esp_timer.h"

class Program {
private:
    unsigned long m_startedAt;
    bool m_started = false;

public:
    void start() {
        // VIOLATION: Direct dependency on ESP-IDF
        m_startedAt = esp_timer_get_time() / 1000;
        m_started = true;
    }

    bool isExpired(unsigned long durationMs) {
        // VIOLATION: Cannot test timing logic without hardware
        unsigned long now = esp_timer_get_time() / 1000;
        return (now - m_startedAt) >= durationMs;
    }
};
```

**Problems:**

* Cannot run unit tests on host (no `esp_timer` available)
* Cannot test edge cases (time rollover, specific durations)
* Tightly coupled to ESP32 platform

### GOOD — Abstraction-Based (DIP compliant)

> High-level code declares the behavior it needs via an abstract interface.

#### Step 1: Define the abstraction (interface)

```cpp
// interfaces/Clock.h
class Clock {
public:
    virtual ~Clock() = default;
    [[nodiscard]] virtual unsigned long now() const = 0;
};
```

#### Step 2: Business logic depends on abstraction

```cpp
// core/Program.h
#include "Clock.h"

class Program {
private:
    Clock& m_clock;  // Depends on abstraction
    unsigned long m_startedAt;
    bool m_started = false;

public:
    explicit Program(Clock& clock) : m_clock(clock) {}

    void start() {
        m_startedAt = m_clock.now();
        m_started = true;
    }

    bool isExpired(unsigned long durationMs) {
        return (m_clock.now() - m_startedAt) >= durationMs;
    }
};
```

#### Step 3: ESP32 implementation

```cpp
// esp32/components/clock/ESP32Clock.h
#include "Clock.h"
#include "esp_timer.h"

class ESP32Clock : public Clock {
public:
    unsigned long now() const override {
        return esp_timer_get_time() / 1000;  // Convert to milliseconds
    }
};
```

#### Step 4: Host-based test implementation

```cpp
// core/ChronoClock.h (for host tests)
#include "Clock.h"
#include <chrono>

class ChronoClock : public Clock {
public:
    unsigned long now() const override {
        using namespace std::chrono;
        return duration_cast<milliseconds>(
            steady_clock::now().time_since_epoch()
        ).count();
    }
};
```

#### Step 5: Mock for unit tests

```cpp
// tests/mocks/MockClock.h
#include "Clock.h"

class MockClock : public Clock {
private:
    unsigned long m_time = 0;

public:
    unsigned long now() const override { return m_time; }

    // Test helpers
    void setTime(unsigned long time) { m_time = time; }
    void advance(unsigned long ms) { m_time += ms; }
};
```

#### Why this works

* **Testable**: Unit tests use `MockClock` to control time precisely
* **Portable**: Core logic runs on host without ESP-IDF
* **Flexible**: Can swap implementations (ESP32, STM32, host)
* **Decoupled**: Business logic doesn't know about `esp_timer`

---

## Logging Example

### Interface definition

```cpp
// interfaces/ILogger.h
namespace logging {

class ILogger {
public:
    virtual ~ILogger() = default;
    virtual void debug(const char* format, ...) = 0;
    virtual void info(const char* format, ...) = 0;
    virtual void warning(const char* format, ...) = 0;
    virtual void error(const char* format, ...) = 0;
};

} // namespace logging
```

### ESP32 implementation

```cpp
// esp32/components/logger/ESP32Logger.h
#include "ILogger.h"
#include "esp_log.h"

namespace logging {

class ESP32Logger : public ILogger {
private:
    const char* m_tag;

public:
    explicit ESP32Logger(const char* tag) : m_tag(tag) {}

    void info(const char* format, ...) override {
        va_list args;
        va_start(args, format);
        esp_log_writev(ESP_LOG_INFO, m_tag, format, args);
        va_end(args);
    }
    // ... other methods
};

} // namespace logging
```

### Usage with injection

```cpp
class BLEController {
private:
    logging::ILogger& m_logger;

public:
    explicit BLEController(logging::ILogger& logger) : m_logger(logger) {}

    void initialize() {
        m_logger.info("Initializing BLE controller");
        // ... initialization code
    }
};
```

---

## When to Apply DIP

### Use DIP for

* **Hardware access**: Timers, GPIO, peripherals, sensors
* **Platform services**: Logging, storage (NVS), networking
* **External dependencies**: BLE stack, WiFi, file systems
* **Anything that prevents host testing**

### Prefer Concrete Types for

* **Data structures**: Configuration structs, command payloads
* **Pure algorithms**: Math functions, parsers, state machines
* **Value types**: Colors, coordinates, durations

```cpp
// Behavior abstraction (good DIP)
class ILogger {
public:
    virtual void info(const char* msg) = 0;
};

// Concrete data (no abstraction needed)
struct LightCommand {
    uint8_t red;
    uint8_t green;
    uint8_t blue;
    uint16_t duration_ms;
};
```

---

## Anti-patterns to Avoid

1. **Over-abstraction**: Creating interfaces for pure algorithms that don't
   need mocking
2. **Leaky abstractions**: Interfaces that expose ESP-IDF types
   (e.g., `esp_err_t`, `gpio_num_t`)
3. **God interfaces**: Huge interfaces with 20+ methods — split them
4. **Unused abstractions**: Interfaces with only one implementation and no
   tests

---

## C++ DIP Techniques

1. **Pure virtual base classes**: Use `virtual ~Destructor() = default` and
   `= 0` for abstract methods
2. **Constructor injection**: Pass dependencies as references
   (`Clock& clock`)
3. **Reference members**: Store injected dependencies as references, not
   pointers
4. **`[[nodiscard]]`**: Mark query methods to prevent ignoring return values

```cpp
class Program {
private:
    Clock& m_clock;           // Reference, not pointer
    logging::ILogger& m_log;  // Reference, not pointer

public:
    Program(Clock& clock, logging::ILogger& log)
        : m_clock(clock), m_log(log) {}
};
```

---

## Key Takeaways

* Depend on **abstract interfaces**, not ESP-IDF concrete types
* Let **business logic** declare the contracts it needs
* **Inject implementations** via constructor — don't construct inside classes
* Keep interfaces **small and focused** (see ISP)
* Mocking is trivial; **tests run on host** without hardware

## Related Best Practices

* [c-isolation.md](../best-practices/c-isolation.md) — Wrapping ESP-IDF C APIs
* [testing.md](../best-practices/testing.md) — Host-based unit testing patterns
* [Interface Segregation](./interface-segregation.md) — Keeping interfaces small
