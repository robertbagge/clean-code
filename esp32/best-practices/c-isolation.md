# C Isolation

Patterns for wrapping ESP-IDF C APIs in C++ classes, enabling type safety,
RAII, and testability.

---

## Why Isolate C APIs

ESP-IDF is primarily a C library. Wrapping C APIs in C++ provides:

* **Type safety**: Replace `#define` constants with `enum class`
* **RAII**: Automatic cleanup of C resources (handles, mutexes)
* **Testability**: Mock the C++ wrapper, not the C API
* **Abstraction**: Business logic doesn't know about ESP-IDF

---

## Wrapper Classes

### Wrap C APIs in C++ RAII classes

```cpp
// BAD — Raw C API usage scattered in code
void doSomething() {
    esp_timer_handle_t timer;
    esp_timer_create(&config, &timer);
    // ... use timer
    esp_timer_delete(timer);  // Easy to forget!
}

// GOOD — RAII wrapper
class Timer {
private:
    esp_timer_handle_t m_handle = nullptr;

public:
    Timer(esp_timer_create_args_t& config) {
        ESP_ERROR_CHECK(esp_timer_create(&config, &m_handle));
    }

    ~Timer() {
        if (m_handle) {
            esp_timer_stop(m_handle);
            esp_timer_delete(m_handle);
        }
    }

    void start(uint64_t period_us) {
        ESP_ERROR_CHECK(esp_timer_start_periodic(m_handle, period_us));
    }

    void stop() {
        esp_timer_stop(m_handle);
    }

    // Prevent copying
    Timer(const Timer&) = delete;
    Timer& operator=(const Timer&) = delete;
};
```

### Logger wrapper example (from codebase)

```cpp
// ILogger.h — Abstract interface
class ILogger {
public:
    virtual ~ILogger() = default;
    virtual void debug(const char* format, ...) = 0;
    virtual void info(const char* format, ...) = 0;
    virtual void error(const char* format, ...) = 0;
};

// ESP32Logger.h — Wraps esp_log_*
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
    // ... other methods wrap esp_log_*
};
```

---

## Boundary Isolation

### Keep `extern "C"` at module boundaries only

```cpp
// BAD — extern "C" scattered in business logic
class BLEController {
public:
    void init() {
        extern "C" {
            esp_bt_controller_init(&cfg);
        }
    }
};

// GOOD — extern "C" only in dedicated wrapper files
// bt_wrapper.cpp
extern "C" {
#include "esp_bt.h"
}

// BLEController.h — uses C++ wrapper
class BLEController {
private:
    BTWrapper m_bt;  // C++ wrapper handles extern "C"
public:
    void init() {
        m_bt.init();  // Clean C++ interface
    }
};
```

### Entry point isolation

```cpp
// main.cpp — only place with extern "C" for entry
extern "C" void app_main() {
    // All C++ from here
    auto app = std::make_unique<Application>();
    app->run();
}
```

---

## Type-Safe Enums

### Wrap `#define` constants in `enum class`

```cpp
// BAD — Using raw #define constants
#define BLE_ADV_TYPE_IND 0
#define BLE_ADV_TYPE_DIRECT_IND 1
#define BLE_ADV_TYPE_SCAN_IND 2

void setAdvType(int type) {  // No type safety
    // type could be any int!
}

// GOOD — enum class wrapper
enum class AdvertisingType : uint8_t {
    Connectable = BLE_ADV_TYPE_IND,
    DirectedConnectable = BLE_ADV_TYPE_DIRECT_IND,
    Scannable = BLE_ADV_TYPE_SCAN_IND
};

void setAdvType(AdvertisingType type) {  // Type safe
    esp_ble_adv_params_t params;
    params.adv_type = static_cast<uint8_t>(type);
    // ...
}
```

### GPIO pin wrapper

```cpp
// Instead of raw gpio_num_t everywhere
enum class LedPin : gpio_num_t {
    StatusLed = GPIO_NUM_2,
    PowerLed = GPIO_NUM_4,
    ErrorLed = GPIO_NUM_5
};

class Led {
private:
    LedPin m_pin;

public:
    explicit Led(LedPin pin) : m_pin(pin) {
        gpio_set_direction(static_cast<gpio_num_t>(m_pin), GPIO_MODE_OUTPUT);
    }

    void on() {
        gpio_set_level(static_cast<gpio_num_t>(m_pin), 1);
    }

    void off() {
        gpio_set_level(static_cast<gpio_num_t>(m_pin), 0);
    }
};
```

---

## Error Translation

### Convert `esp_err_t` at the wrapper boundary

```cpp
// Wrapper translates ESP errors to domain errors
enum class StorageError {
    None,
    NotFound,
    InvalidData,
    Full,
    Unknown
};

class NvsStorage {
public:
    StorageError getString(const char* key, std::string& out) {
        size_t len;
        esp_err_t ret = nvs_get_str(m_handle, key, nullptr, &len);

        switch (ret) {
            case ESP_OK:
                out.resize(len);
                nvs_get_str(m_handle, key, out.data(), &len);
                return StorageError::None;

            case ESP_ERR_NVS_NOT_FOUND:
                return StorageError::NotFound;

            case ESP_ERR_NVS_INVALID_LENGTH:
                return StorageError::InvalidData;

            default:
                m_logger.error("NVS error: %s", esp_err_to_name(ret));
                return StorageError::Unknown;
        }
    }
};
```

---

## Callback Pattern Wrapping

### Wrap C callbacks with type-safe C++ alternatives

```cpp
// C callback style
typedef void (*timer_callback_t)(void* arg);

// C++ wrapper with std::function
class Timer {
public:
    using Callback = std::function<void()>;

private:
    Callback m_callback;

    static void staticCallback(void* arg) {
        auto* self = static_cast<Timer*>(arg);
        if (self->m_callback) {
            self->m_callback();
        }
    }

public:
    void setCallback(Callback cb) {
        m_callback = std::move(cb);
        esp_timer_create_args_t args = {
            .callback = &Timer::staticCallback,
            .arg = this,
            .name = "timer"
        };
        // ...
    }
};

// Usage — type-safe lambda
timer.setCallback([this]() {
    handleTimeout();
});
```

---

## Testability Benefits

### Business logic testable without ESP-IDF

```cpp
// Production code
class LightController {
private:
    ILedDriver& m_driver;  // Abstract interface

public:
    void fadeIn(Duration duration) {
        for (int i = 0; i <= 100; i++) {
            m_driver.setBrightness(i);
            m_clock.delay(duration / 100);
        }
    }
};

// Test with mock
class MockLedDriver : public ILedDriver {
public:
    std::vector<int> brightnessHistory;

    void setBrightness(int level) override {
        brightnessHistory.push_back(level);
    }
};

TEST(LightController, FadeInSetsIncreasingBrightness) {
    MockLedDriver driver;
    MockClock clock;
    LightController controller(driver, clock);

    controller.fadeIn(100ms);

    EXPECT_EQ(driver.brightnessHistory.size(), 101);
    EXPECT_EQ(driver.brightnessHistory.front(), 0);
    EXPECT_EQ(driver.brightnessHistory.back(), 100);
}
```

---

## Project Structure

```
firmware/
├── interfaces/          # Abstract C++ interfaces
│   ├── Clock.h
│   ├── ILogger.h
│   └── ILedDriver.h
├── core/               # Platform-agnostic business logic
│   └── LightController.cpp
└── esp32/
    └── components/
        ├── clock/      # ESP32 Clock implementation
        ├── logger/     # ESP32 Logger (wraps esp_log)
        └── led/        # ESP32 LED driver (wraps ledc/gpio)
```

---

## Key Takeaways

* **Wrap C APIs** in C++ RAII classes for automatic cleanup
* **Keep `extern "C"`** at module boundaries only
* **Use `enum class`** instead of `#define` constants
* **Translate errors** to domain types at the wrapper layer
* **Abstract with interfaces** to enable host-based testing

## Related Best Practices

* [Dependency Inversion](../clean-code/dependency-inversion.md) — Interface
  design for wrappers
* [memory-management.md](./memory-management.md) — RAII for C resources
* [testing.md](./testing.md) — Mocking C++ wrappers
