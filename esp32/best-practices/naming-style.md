# Naming & Style

Conventions for naming, formatting, and code style in ESP32 firmware.

---

## Naming Conventions

### Classes and Structs — PascalCase

```cpp
class LightController { };
class BLEGattsProfileHandler { };
struct ProgramDefinition { };
struct LightCommand { };
```

### Methods and Functions — camelCase

```cpp
void sendCommand(const LightCommand& cmd);
unsigned long calculateCycleDuration();
void handleWriteEvent(const WriteEvent& event);
```

### Member Variables — `m_` prefix with camelCase

```cpp
class LightController {
private:
    QueueHandle_t m_commandQueue;
    TaskHandle_t m_taskHandle;
    int m_taskCore;
    std::unique_ptr<LightMode> m_currentMode;
};
```

### Constants — `UPPER_SNAKE_CASE` or `constexpr camelCase`

```cpp
// Preprocessor defines (legacy)
#define LIGHT_PROGRAM_TASK_STACK_SIZE 4096
#define LIGHT_PROGRAM_QUEUE_SIZE 10

// Prefer constexpr (modern C++)
constexpr size_t kLightTaskStackSize = 4096;
constexpr size_t kLightQueueSize = 10;

// Namespace-scoped constants
namespace HardwareConfig {
    constexpr uint32_t PWM_FREQUENCY_HZ = 5000;
    constexpr gpio_num_t LED_PIN = GPIO_NUM_25;
}
```

### Enum Classes — PascalCase type, PascalCase values

```cpp
enum class LightModeType {
    Sunrise,
    Candle,
    BoxBreathing,
    TaskLight
};

enum class ProgramType {
    LightTherapy,
    Breathing,
    Custom
};
```

### Namespaces — lowercase

```cpp
namespace logging { }
namespace ble { }
namespace light { }
```

---

## File Naming

### Headers — PascalCase matching class name

```text
LightController.h
BLEApplication.h
ESP32Logger.h
ILogger.h           // Interface with I prefix
```

### Source files — matching header

```text
LightController.cpp
BLEApplication.cpp
ESP32Logger.cpp
```

### Test files — `*Tests.cpp` suffix

```text
ProgramTests.cpp
BLECommandParserTests.cpp
MockLoggerTests.cpp
```

---

## Clang-Format Configuration

### Key settings from `.clang-format`

```yaml
BasedOnStyle: Mozilla
IndentWidth: 2
ColumnLimit: 80
BreakBeforeBraces: Custom
BraceWrapping:
  AfterClass: true
  AfterFunction: true
  AfterStruct: true
AllowShortFunctionsOnASingleLine: Inline
```

### Formatting examples

```cpp
// 2-space indentation
class LightController {
public:
  void init();
  bool sendCommand(const LightCommand& cmd);

private:
  QueueHandle_t m_commandQueue;
};

// 80 character line limit
void LightController::processCommand(
    const LightCommand& cmd,
    const ProcessingOptions& options)
{
  // Implementation
}

// Constructor initializer list
LightController::LightController()
  : m_commandQueue(nullptr)
  , m_taskHandle(nullptr)
  , m_taskCore(1)
  , m_logger(logging::ESP32Logger::getInstance())
{
}
```

---

## Brace Style

### Function definitions — brace on new line

```cpp
void LightController::init()
{
  // Implementation
}
```

### Control structures — brace on same line

```cpp
if (condition) {
  doSomething();
} else {
  doSomethingElse();
}

for (int i = 0; i < count; i++) {
  process(i);
}
```

### Short inline functions — single line OK

```cpp
class Clock {
public:
  virtual ~Clock() = default;
  [[nodiscard]] virtual unsigned long now() const = 0;
};
```

---

## Comment Style

### File headers

```cpp
//
// Created by Robert Bagge on 28/04/2025.
//
```

### Doxygen for public APIs

```cpp
/**
 * @brief Send a command to the light controller
 * @param cmd The command to execute
 * @return true if command was queued successfully
 */
bool sendCommand(const LightCommand& cmd);
```

### Inline comments — explain "why", not "what"

```cpp
// Scale brightness to available hardware range (max 200 for this LED)
uint8_t scaled = (level * 200) / 255;

// Avoid:
// Set scaled to level times 200 divided by 255
```

### TODO comments

```cpp
// TODO: Add error handling for queue full condition
// TODO(rbagge): Refactor to use strategy pattern
```

---

## Include Guards

### Prefer `#ifndef` style

```cpp
#ifndef LIGHT_CONTROLLER_H
#define LIGHT_CONTROLLER_H

// Content

#endif // LIGHT_CONTROLLER_H
```

### Guard name matches file path

```cpp
// firmware/esp32/components/ble/include/BLEController.h
#ifndef BLE_CONTROLLER_H
#define BLE_CONTROLLER_H
```

---

## Pointer and Reference Style

### Attach `*` and `&` to type

```cpp
// Preferred
LightCommand* cmd;
const LightCommand& cmdRef;
std::unique_ptr<LightMode> mode;

// Avoid
LightCommand *cmd;
const LightCommand &cmdRef;
```

---

## Modern C++ Style

### Use `auto` appropriately

```cpp
// Good — type is clear from context
auto ptr = std::make_unique<LightMode>();
auto it = m_commands.find(key);

// Avoid — type is not obvious
auto x = getValue();  // What type is x?
```

### Use `override` for virtual methods

```cpp
class ESP32Clock : public Clock {
public:
    unsigned long now() const override;  // Always use override
};
```

### Use `[[nodiscard]]` for important return values

```cpp
[[nodiscard]] virtual unsigned long now() const = 0;
[[nodiscard]] bool sendCommand(const LightCommand& cmd);
```

### Use `= default` and `= delete`

```cpp
class NonCopyable {
public:
    NonCopyable() = default;
    ~NonCopyable() = default;

    NonCopyable(const NonCopyable&) = delete;
    NonCopyable& operator=(const NonCopyable&) = delete;
};
```

---

## Running Format

```bash
# Format all files
task format

# Or directly
clang-format -i src/*.cpp include/*.h
```

---

## Key Takeaways

* **PascalCase** for classes, **camelCase** for methods
* **`m_`** prefix for member variables
* **2-space** indentation, **80** character limit
* **`override`** on all virtual overrides
* **Run clang-format** before committing

## Related Best Practices

* [component-structure.md](./component-structure.md) — File organization
