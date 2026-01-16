# Component Structure

Patterns for ESP-IDF component layout, CMake configuration, and file
organization.

---

## ESP-IDF Component Layout

### Standard component structure

```
firmware/esp32/components/
└── my-component/
    ├── CMakeLists.txt      # Component build configuration
    ├── Kconfig             # Optional: menuconfig options
    ├── include/            # Public headers (exported to dependents)
    │   └── MyComponent.h
    ├── src/                # Private implementation
    │   └── MyComponent.cpp
    └── test/               # Component-specific tests (optional)
        └── test_my_component.cpp
```

---

## CMakeLists.txt Structure

### Basic component

```cmake
idf_component_register(
    SRCS
        "src/MyComponent.cpp"
    INCLUDE_DIRS
        "include"
    REQUIRES
        driver          # ESP-IDF components
        freertos
    PRIV_REQUIRES
        esp_timer       # Only needed internally
)
```

### Component with dependencies

```cmake
idf_component_register(
    SRCS
        "src/BLEApplication.cpp"
        "src/BLEController.cpp"
        "src/BLEGapManager.cpp"
    INCLUDE_DIRS
        "include"
        "${CMAKE_SOURCE_DIR}/../interfaces"   # Abstract interfaces
    REQUIRES
        bt              # Bluetooth stack
        nvs_flash       # Non-volatile storage
        logger          # Our logging component
)
```

---

## Public vs Private Headers

### Public headers (in `include/`)

* Exported to components that depend on this one
* Define the interface other components use
* Keep minimal — only expose what's needed

```cpp
// include/LightController.h — Public interface
#ifndef LIGHT_CONTROLLER_H
#define LIGHT_CONTROLLER_H

#include "LightCommand.h"

class LightController {
public:
    LightController();
    ~LightController();

    void init();
    bool sendCommand(const LightCommand& cmd);

private:
    class Impl;  // Pimpl for hiding implementation
    std::unique_ptr<Impl> m_impl;
};

#endif
```

### Private headers (in `src/`)

* Implementation details not exposed to dependents
* Internal classes and helpers
* Platform-specific details

```cpp
// src/LightControllerImpl.h — Private implementation
class LightController::Impl {
    QueueHandle_t m_queue;
    TaskHandle_t m_task;
    // ... internal details
};
```

---

## Interface Organization

### Abstract interfaces in separate directory

```
firmware/
├── interfaces/           # Platform-agnostic abstractions
│   ├── Clock.h
│   ├── ILogger.h
│   └── ILedDriver.h
├── core/                 # Platform-agnostic business logic
│   └── src/
│       └── programs/
└── esp32/                # ESP32-specific implementations
    └── components/
        ├── clock/        # ESP32Clock implementation
        └── logger/       # ESP32Logger implementation
```

### Interface header pattern

```cpp
// interfaces/ILogger.h
#ifndef ILOGGER_H
#define ILOGGER_H

namespace logging {

class ILogger {
public:
    virtual ~ILogger() = default;
    virtual void debug(const char* format, ...) = 0;
    virtual void info(const char* format, ...) = 0;
    virtual void error(const char* format, ...) = 0;
};

} // namespace logging

#endif
```

---

## Header Include Order

### Consistent ordering

```cpp
// 1. Corresponding header (for .cpp files)
#include "MyComponent.h"

// 2. C system headers
#include <cstdint>
#include <cstring>

// 3. C++ standard library
#include <memory>
#include <vector>
#include <optional>

// 4. ESP-IDF headers
#include "esp_log.h"
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"

// 5. Project interfaces
#include "ILogger.h"
#include "Clock.h"

// 6. Other project headers
#include "LightCommand.h"
```

---

## Include Guards

### Prefer `#ifndef` guards

```cpp
#ifndef MY_COMPONENT_H
#define MY_COMPONENT_H

// ... content

#endif // MY_COMPONENT_H
```

### Also acceptable: `#pragma once`

```cpp
#pragma once

// ... content
```

---

## Namespace Organization

### Component-based namespaces

```cpp
// Logging component
namespace logging {
    class ILogger { };
    class ESP32Logger : public ILogger { };
}

// BLE component
namespace ble {
    class BLEApplication { };
    class BLEController { };
}

// Light control
namespace light {
    class LightController { };
    class LightMode { };
}
```

---

## Test Organization

### Component tests alongside source

```
firmware/esp32/components/ble/
├── include/
├── src/
└── tests/
    ├── BLECommandParserTests.cpp
    └── MockBLEStack.h
```

### Core tests separate from ESP32 code

```
firmware/
├── core/
│   ├── src/
│   │   └── programs/
│   │       ├── Program.h
│   │       └── Program.cpp
│   └── tests/
│       └── programs/
│           ├── ProgramTests.cpp
│           └── ProgramManagerTests.cpp
```

---

## Main Application Structure

```
firmware/esp32/main/
├── CMakeLists.txt
├── main.cpp              # app_main entry point
├── Kconfig.projbuild     # Project-specific config
└── idf_component.yml     # Component dependencies
```

### Main CMakeLists.txt

```cmake
idf_component_register(
    SRCS
        "main.cpp"
    INCLUDE_DIRS
        "."
        "${CMAKE_SOURCE_DIR}/../interfaces"
    REQUIRES
        ble
        light-control
        logger
)
```

---

## Dependency Direction

### Inward-pointing dependencies

```
                    main
                     │
         ┌──────────┼──────────┐
         ▼          ▼          ▼
    light-control  ble       logger
         │          │          │
         └────┬─────┘          │
              ▼                │
          interfaces ◄─────────┘
```

* Components depend on `interfaces`, not each other
* `main` orchestrates components
* No circular dependencies

---

## Key Takeaways

* **Standard layout**: `include/`, `src/`, `CMakeLists.txt`
* **Public vs private**: Only expose what's needed
* **Interfaces separate**: Platform-agnostic in `interfaces/`
* **Consistent includes**: Ordered by scope
* **Inward dependencies**: Components depend on abstractions

## Related Best Practices

* [Dependency Inversion](../clean-code/dependency-inversion.md) — Interface
  organization
* [naming-style.md](./naming-style.md) — File and class naming
