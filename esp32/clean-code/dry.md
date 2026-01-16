# Don't Repeat Yourself (DRY) in C++

## Overview

The DRY principle states that every piece of knowledge should have a single,
unambiguous representation in a system. In C++, this means centralizing
constants, extracting common logic, and avoiding copy-paste code.

## Core Concept

In embedded C++, DRY means:

* Define constants once in configuration headers
* Extract repeated code into functions or base classes
* Centralize error handling and logging patterns
* Use templates for type variations of same logic

## Implementation Example

### BAD — Repeated Code (DRY violation)

```cpp
void BLEController::init() {
    esp_err_t ret = esp_bt_controller_init(&bt_cfg);
    if (ret != ESP_OK) {
        ESP_LOGE("BLE", "BT controller init failed: %s", esp_err_to_name(ret));
        return;
    }

    ret = esp_bt_controller_enable(ESP_BT_MODE_BLE);
    if (ret != ESP_OK) {
        ESP_LOGE("BLE", "BT controller enable failed: %s", esp_err_to_name(ret));
        return;
    }

    ret = esp_bluedroid_init();
    if (ret != ESP_OK) {
        ESP_LOGE("BLE", "Bluedroid init failed: %s", esp_err_to_name(ret));
        return;
    }

    ret = esp_bluedroid_enable();
    if (ret != ESP_OK) {
        ESP_LOGE("BLE", "Bluedroid enable failed: %s", esp_err_to_name(ret));
        return;
    }
    // Same pattern repeated 4 times!
}
```

### GOOD — Centralized Pattern (DRY compliant)

```cpp
// Helper macro or function for ESP error checking with logging
#define ESP_RETURN_ON_ERROR(x, tag, msg) do { \
    esp_err_t __err = (x); \
    if (__err != ESP_OK) { \
        ESP_LOGE(tag, "%s: %s", msg, esp_err_to_name(__err)); \
        return __err; \
    } \
} while(0)

esp_err_t BLEController::init() {
    ESP_RETURN_ON_ERROR(esp_bt_controller_init(&bt_cfg),
                        "BLE", "BT controller init failed");

    ESP_RETURN_ON_ERROR(esp_bt_controller_enable(ESP_BT_MODE_BLE),
                        "BLE", "BT controller enable failed");

    ESP_RETURN_ON_ERROR(esp_bluedroid_init(),
                        "BLE", "Bluedroid init failed");

    ESP_RETURN_ON_ERROR(esp_bluedroid_enable(),
                        "BLE", "Bluedroid enable failed");

    return ESP_OK;
}
```

---

## Centralize Constants

### BAD — Magic numbers scattered

```cpp
// In main.cpp
ledc_timer_config_t timer = { .freq_hz = 5000, ... };

// In led_controller.cpp
#define PWM_FREQ 5000

// In test_leds.cpp
constexpr uint32_t FREQ = 5000;
```

### GOOD — Single source of truth

```cpp
// hardware/config.h
namespace HardwareConfig {
    constexpr uint32_t PWM_FREQUENCY_HZ = 5000;
    constexpr uint8_t PWM_RESOLUTION_BITS = 13;
    constexpr gpio_num_t LED_PIN = GPIO_NUM_25;
}

// All files use the same source
#include "hardware/config.h"
ledc_timer_config_t timer = {
    .freq_hz = HardwareConfig::PWM_FREQUENCY_HZ,
    ...
};
```

---

## Extract Common Base Classes

### BAD — Duplicated mode logic

```cpp
class SunriseMode {
    ILedDriver& m_driver;
    bool m_running = false;
    unsigned long m_startTime = 0;

public:
    void start() {
        m_running = true;
        m_startTime = millis();
    }
    void stop() { m_running = false; m_driver.clear(); }
    // Sunrise-specific update...
};

class CandleMode {
    ILedDriver& m_driver;
    bool m_running = false;
    unsigned long m_startTime = 0;

public:
    void start() {
        m_running = true;
        m_startTime = millis();
    }
    void stop() { m_running = false; m_driver.clear(); }
    // Candle-specific update...
};
// Same start/stop duplicated!
```

### GOOD — Common base class

```cpp
class LightMode {
protected:
    ILedDriver& m_driver;
    bool m_running = false;
    unsigned long m_startTime = 0;

public:
    explicit LightMode(ILedDriver& driver) : m_driver(driver) {}
    virtual ~LightMode() = default;

    virtual void start() {
        m_running = true;
        m_startTime = millis();
    }

    virtual void stop() {
        m_running = false;
        m_driver.clear();
    }

    virtual void update() = 0;
};

class SunriseMode : public LightMode {
public:
    using LightMode::LightMode;
    void update() override { /* Sunrise-specific */ }
};

class CandleMode : public LightMode {
public:
    using LightMode::LightMode;
    void update() override { /* Candle-specific */ }
};
```

---

## Use Templates for Type Variations

### Avoid copy-paste for different types

```cpp
// BAD — Separate functions for each type
uint8_t clampU8(uint8_t value, uint8_t min, uint8_t max);
uint16_t clampU16(uint16_t value, uint16_t min, uint16_t max);
float clampFloat(float value, float min, float max);

// GOOD — Template
template<typename T>
T clamp(T value, T min, T max) {
    if (value < min) return min;
    if (value > max) return max;
    return value;
}
```

---

## Centralize Cross-Cutting Concerns

### Logging wrapper

```cpp
// Instead of repeating ESP_LOGI everywhere with same pattern
class ComponentLogger {
private:
    const char* m_tag;

public:
    explicit ComponentLogger(const char* tag) : m_tag(tag) {}

    template<typename... Args>
    void info(const char* format, Args... args) {
        ESP_LOGI(m_tag, format, args...);
    }

    template<typename... Args>
    void error(const char* format, Args... args) {
        ESP_LOGE(m_tag, format, args...);
    }
};

// Usage
class BLEController {
    ComponentLogger m_log{"BLE"};

public:
    void init() {
        m_log.info("Initializing BLE controller");
    }
};
```

---

## When NOT to DRY

Don't over-apply DRY:

* **Incidental similarity**: Code looks similar but serves different purposes
* **Premature abstraction**: Don't abstract until you see the pattern 3 times
* **Coupling unrelated code**: DRY shouldn't create dependencies between
  unrelated modules

```cpp
// These look similar but have different purposes — keep separate
void validateUserInput(const UserData& data);
void validateConfigFile(const ConfigData& data);
// Don't merge into generic validate() just because they both validate
```

---

## Key Takeaways

* **Single source** for constants and configuration
* **Extract base classes** when behavior is truly shared
* **Use templates** for type-independent logic
* **Centralize** cross-cutting concerns (logging, error handling)
* **Don't over-DRY** incidentally similar code

## Related Best Practices

* [Open-Closed](./open-closed.md) — Base classes for shared behavior
* [hardware-abstraction.md](../best-practices/hardware-abstraction.md) —
  Centralizing hardware config
