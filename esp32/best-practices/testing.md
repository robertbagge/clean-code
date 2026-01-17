# Testing

Patterns for host-based unit testing with googletest and gmock.

---

## Host-Based Testing

Run tests on your development machine without ESP-IDF or hardware:

```text
firmware/
├── core/                  # Platform-agnostic code
│   ├── src/
│   └── tests/            # Tests run on host
├── esp32/
│   ├── components/       # ESP32-specific implementations
│   └── tests/            # Component tests (also on host)
└── interfaces/           # Abstract interfaces
```

### Why host-based testing

* **Fast**: No flash/boot cycle, runs in milliseconds
* **Debuggable**: Use standard debuggers (GDB, LLDB)
* **CI-friendly**: Run in any build environment
* **Isolated**: Test logic without hardware dependencies

---

## Mock Pattern

### Simple mock with GMock

```cpp
#include <gmock/gmock.h>
#include "Clock.h"

class MockClock : public Clock {
public:
    MOCK_METHOD(unsigned long, now, (), (const, override));
};
```

### Usage in tests

```cpp
TEST(Program, Start) {
    MockClock mockClock;
    ProgramDefinition definition = { /* ... */ };
    Program program(mockClock, definition);

    // Set expectation: when now() is called, return 123456
    EXPECT_CALL(mockClock, now).WillOnce(testing::Return(123456));

    program.start();

    EXPECT_EQ(program.state.started_at, 123456);
    EXPECT_TRUE(program.state.started);
}
```

---

## Variadic Function Mocking

GMock can't directly mock variadic functions like `printf`-style loggers.
Use a helper pattern:

```cpp
class MockLogger : public logging::ILogger {
public:
    // Mock non-variadic helpers
    MOCK_METHOD(void, debugImpl, (const std::string& message));
    MOCK_METHOD(void, infoImpl, (const std::string& message));
    MOCK_METHOD(void, errorImpl, (const std::string& message));

    // Variadic methods format and delegate
    void debug(const char* format, ...) override {
        va_list args;
        va_start(args, format);
        char buffer[256];
        vsnprintf(buffer, sizeof(buffer), format, args);
        va_end(args);
        debugImpl(std::string(buffer));
    }

    void info(const char* format, ...) override {
        va_list args;
        va_start(args, format);
        char buffer[256];
        vsnprintf(buffer, sizeof(buffer), format, args);
        va_end(args);
        infoImpl(std::string(buffer));
    }
    // ... similar for warning, error
};
```

### Testing log output

```cpp
TEST(BLEController, LogsInitialization) {
    MockLogger logger;
    BLEController controller(logger);

    EXPECT_CALL(logger, infoImpl(testing::HasSubstr("Initializing")));

    controller.init();
}
```

---

## Time-Based Testing

### Control time in tests

```cpp
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

### Test time-dependent behavior

```cpp
TEST(Program, GetCurrentState_5PercentComplete) {
    MockClock mockClock;
    ProgramDefinition definition = {
        ProgramType::LIGHT_THERAPY,
        { /* phases totaling 100 seconds */ },
        1
    };
    Program program(mockClock, definition);

    // Start at time 5000
    EXPECT_CALL(mockClock, now).WillOnce(testing::Return(5000));
    program.start();

    // Check state at 5% through (5 seconds later)
    EXPECT_CALL(mockClock, now).WillOnce(testing::Return(10000));
    ProgramState& state = program.getCurrentState();

    EXPECT_FLOAT_EQ(state.phase_completion, 0.05f);
}
```

---

## Test Organization

### One test file per component

```text
firmware/core/tests/programs/
├── ProgramTests.cpp
└── ProgramManagerTests.cpp

firmware/esp32/tests/ble/
├── BLECommandParserTests.cpp
└── MockLoggerTests.cpp
```

### Clear test names

```cpp
// Pattern: ComponentTest_Scenario_ExpectedBehavior
TEST(Program, Create) { }
TEST(Program, Start) { }
TEST(Program, Stop) { }
TEST(ProgramTest, GetCurrentCycle_ZeroElapsedTime) { }
TEST(ProgramTest, GetCurrentCycle_OneCycleElapsed) { }
TEST(ProgramTest, GetCurrentState_ProgramDurationLapsed) { }
```

---

## Testing Pure Functions

Pure functions don't need mocks — just test input/output:

```cpp
TEST(BLECommandParser, ParseValidCommand) {
    auto result = parseCommand("SET_COLOR:255,128,64");

    ASSERT_TRUE(result.has_value());
    EXPECT_EQ(result->r, 255);
    EXPECT_EQ(result->g, 128);
    EXPECT_EQ(result->b, 64);
}

TEST(BLECommandParser, ParseInvalidCommand) {
    auto result = parseCommand("INVALID");

    EXPECT_FALSE(result.has_value());
}
```

---

## Boundary Condition Testing

Test edge cases explicitly:

```cpp
TEST(ProgramTest, GetCurrentCycle_BoundaryConditions) {
    MockClock mockClock;
    // Cycle duration: 100 seconds
    ProgramDefinition definition = { /* ... */ };
    Program program(mockClock, definition);

    EXPECT_CALL(mockClock, now).WillOnce(testing::Return(0));
    program.start();

    // Just before cycle boundary
    EXPECT_EQ(program.getCurrentCycle(100 * 1000 - 1), 0);

    // Exactly at boundary
    EXPECT_EQ(program.getCurrentCycle(100 * 1000), 1);

    // Just after boundary
    EXPECT_EQ(program.getCurrentCycle(100 * 1000 + 1), 1);
}
```

---

## Avoid Mocking Implementation Details

### BAD — Testing implementation

```cpp
TEST(LightController, FadeIn_CallsSetBrightness100Times) {
    MockLedDriver driver;

    // Too specific — breaks if implementation changes
    EXPECT_CALL(driver, setBrightness(testing::_))
        .Times(100);

    controller.fadeIn(100ms);
}
```

### GOOD — Testing behavior

```cpp
TEST(LightController, FadeIn_EndsAtFullBrightness) {
    MockLedDriver driver;
    int lastBrightness = 0;

    EXPECT_CALL(driver, setBrightness(testing::_))
        .WillRepeatedly([&](int level) { lastBrightness = level; });

    controller.fadeIn(100ms);

    EXPECT_EQ(lastBrightness, 100);
}
```

---

## Running Tests

```bash
# Run all unit tests
task test:unit

# Run core tests only
task test:core

# Run ESP32 component tests
task test:esp32
```

---

## CMake Configuration

```cmake
# Top-level CMakeLists.txt
include(FetchContent)
FetchContent_Declare(
    googletest
    GIT_REPOSITORY https://github.com/google/googletest.git
    GIT_TAG release-1.12.1
)
FetchContent_MakeAvailable(googletest)

# Add test executable
add_executable(core_tests
    core/tests/programs/ProgramTests.cpp
    core/tests/programs/ProgramManagerTests.cpp
)

target_link_libraries(core_tests
    PRIVATE
    core_lib
    GTest::gtest_main
    GTest::gmock
)
```

---

## Key Takeaways

* **Run tests on host** — fast iteration without hardware
* **Mock interfaces** with GMock, not implementations
* **Control time** with MockClock for deterministic tests
* **Test boundaries** — zero, one, many, edge cases
* **Clear naming** — `Component_Scenario_ExpectedBehavior`

## Related Best Practices

* [Dependency Inversion](../clean-code/dependency-inversion.md) — Making code
  testable
* [c-isolation.md](./c-isolation.md) — Wrapping C APIs for mockability
