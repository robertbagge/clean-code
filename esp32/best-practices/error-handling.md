# Error Handling

Patterns for `esp_err_t`, domain errors, and recovery strategies.

---

## ESP-IDF Error Codes

ESP-IDF functions return `esp_err_t`. Always check and handle these:

```cpp
esp_err_t ret = esp_bt_controller_init(&bt_cfg);
if (ret != ESP_OK) {
    m_logger.error("BT controller init failed: %s", esp_err_to_name(ret));
    return ret;
}
```

### Use `ESP_ERROR_CHECK` for unrecoverable errors

```cpp
// Aborts on failure — use only during initialization
ESP_ERROR_CHECK(ledc_timer_config(&ledc_timer));
ESP_ERROR_CHECK(ledc_channel_config(&ledc_channel));
```

### Check explicitly for recoverable errors

```cpp
esp_err_t ret = nvs_get_str(handle, key, buffer, &len);
if (ret == ESP_ERR_NVS_NOT_FOUND) {
    // Expected case — use default value
    return defaultValue;
} else if (ret != ESP_OK) {
    // Unexpected error — log and propagate
    m_logger.error("NVS read failed: %s", esp_err_to_name(ret));
    return ret;
}
```

---

## Domain Errors

Don't leak `esp_err_t` to business logic. Map to domain-level errors:

### Define domain error types

```cpp
// domain/errors.h
enum class LightError {
    None,
    InvalidCommand,
    HardwareFailure,
    Timeout
};

// Convert to string for logging
const char* toString(LightError err);
```

### Map at the boundary

```cpp
// At the hardware abstraction layer
LightError setColor(uint8_t r, uint8_t g, uint8_t b) {
    esp_err_t ret = ledc_set_duty(channel, calculateDuty(r));
    if (ret != ESP_OK) {
        m_logger.error("LEDC set duty failed: %s", esp_err_to_name(ret));
        return LightError::HardwareFailure;
    }
    return LightError::None;
}

// Business logic uses domain errors
void processCommand(LightCommand cmd) {
    LightError err = m_light.setColor(cmd.r, cmd.g, cmd.b);
    if (err != LightError::None) {
        m_logger.warning("Light command failed: %s", toString(err));
        // Handle at business logic level
    }
}
```

---

## Return Value Patterns

### Use `std::optional` for "might not exist"

```cpp
std::optional<int> parseInt(const std::string& str) {
    try {
        return std::stoi(str);
    } catch (...) {
        return std::nullopt;
    }
}

// Usage
if (auto value = parseInt(input)) {
    processValue(*value);
} else {
    m_logger.warning("Invalid input: %s", input.c_str());
}
```

### Use result types for operations that can fail

```cpp
template<typename T>
struct Result {
    std::optional<T> value;
    LightError error = LightError::None;

    bool ok() const { return error == LightError::None; }
};

Result<LightState> getLightState() {
    esp_err_t ret = readHardware(&state);
    if (ret != ESP_OK) {
        return { std::nullopt, LightError::HardwareFailure };
    }
    return { state, LightError::None };
}
```

---

## Logging Errors

Always log with context before propagating:

```cpp
esp_err_t BLEApplication::init() {
    esp_err_t ret = esp_bt_controller_init(&bt_cfg);
    if (ret != ESP_OK) {
        // Log the specific failure point
        m_logger.error("BT controller init failed: %s", esp_err_to_name(ret));
        return ret;
    }

    ret = esp_bt_controller_enable(ESP_BT_MODE_BLE);
    if (ret != ESP_OK) {
        // Log with context about what we were trying to do
        m_logger.error("BT controller enable (BLE mode) failed: %s",
                       esp_err_to_name(ret));
        return ret;
    }

    return ESP_OK;
}
```

---

## Recovery Strategies

### Retry with backoff

```cpp
esp_err_t sendWithRetry(const uint8_t* data, size_t len) {
    const int maxRetries = 3;
    const int baseDelayMs = 100;

    for (int attempt = 0; attempt < maxRetries; attempt++) {
        esp_err_t ret = sendData(data, len);
        if (ret == ESP_OK) {
            return ESP_OK;
        }

        if (ret != ESP_ERR_TIMEOUT) {
            // Non-retryable error
            return ret;
        }

        m_logger.warning("Send timeout, retry %d/%d", attempt + 1, maxRetries);
        vTaskDelay(pdMS_TO_TICKS(baseDelayMs * (1 << attempt)));
    }

    return ESP_ERR_TIMEOUT;
}
```

### Graceful degradation

```cpp
void initPeripherals() {
    // Critical — abort if fails
    ESP_ERROR_CHECK(initCriticalHardware());

    // Optional — continue without if fails
    esp_err_t ret = initOptionalSensor();
    if (ret != ESP_OK) {
        m_logger.warning("Optional sensor unavailable, continuing without");
        m_sensorAvailable = false;
    }
}
```

---

## Exception Handling

ESP-IDF supports C++ exceptions but they have overhead. Use sparingly:

### When to use exceptions

* Truly exceptional conditions (out of memory, corruption)
* Constructors that can fail (no return value)
* When RAII cleanup is critical

### When to avoid exceptions

* Normal control flow (use return codes)
* Hot paths / real-time critical code
* ISR context (never throw from ISR)

```cpp
// OK — Constructor failure
class ConfigLoader {
public:
    ConfigLoader(const char* path) {
        if (!loadFromNVS(path)) {
            throw std::runtime_error("Failed to load config");
        }
    }
};

// Better for hot paths — return error
LightError LightController::setColor(RGB color) {
    if (!validateColor(color)) {
        return LightError::InvalidCommand;
    }
    // ...
}
```

---

## Anti-patterns to Avoid

### Silent failures

```cpp
// BAD — Error ignored
void init() {
    esp_wifi_init(&cfg);  // Return value ignored!
}

// GOOD — Error handled
void init() {
    esp_err_t ret = esp_wifi_init(&cfg);
    if (ret != ESP_OK) {
        m_logger.error("WiFi init failed: %s", esp_err_to_name(ret));
        // Either abort or handle gracefully
    }
}
```

### Leaking implementation errors

```cpp
// BAD — Caller must know about ESP-IDF
esp_err_t LightController::setColor(RGB color) {
    return ledc_set_duty(...);  // Leaks esp_err_t
}

// GOOD — Domain error
LightError LightController::setColor(RGB color) {
    esp_err_t ret = ledc_set_duty(...);
    if (ret != ESP_OK) {
        return LightError::HardwareFailure;
    }
    return LightError::None;
}
```

---

## Key Takeaways

* **Always check `esp_err_t`** — no silent failures
* **Map to domain errors** at abstraction boundaries
* **Log with context** before propagating errors
* **Use `std::optional`** for "might not exist" values
* **Reserve exceptions** for truly exceptional conditions

## Related Best Practices

* [c-isolation.md](./c-isolation.md) — Error translation at C/C++ boundary
* [Dependency Inversion](../clean-code/dependency-inversion.md) — Abstracting
  error types
