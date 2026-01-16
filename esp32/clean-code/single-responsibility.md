# Single Responsibility Principle (SRP) in C++

## Overview

The Single Responsibility Principle states that a class should have only one
reason to change. In embedded C++, this means separating hardware control,
business logic, and communication into distinct components.

## Core Concept

In embedded C++, SRP means:

* Each class handles one concern (hardware, logic, or communication)
* Changes to BLE protocol shouldn't affect LED control
* Hardware driver changes shouldn't touch business logic
* Testing is easier when responsibilities are isolated

## Implementation Example

### BAD — Multiple Responsibilities (SRP violation)

```cpp
class LightManager {
private:
    ledc_channel_t m_channel;
    esp_gatt_if_t m_gatts_if;
    uint8_t m_brightness;

public:
    // VIOLATION: Mixes hardware control, BLE handling, and business logic

    void handleBLEWrite(uint8_t* data, size_t len) {
        // BLE protocol parsing
        if (data[0] == 0x01) {
            m_brightness = data[1];
            // Direct hardware control
            ledc_set_duty(LEDC_HIGH_SPEED_MODE, m_channel, m_brightness * 32);
            ledc_update_duty(LEDC_HIGH_SPEED_MODE, m_channel);
        }
    }

    void advertise() {
        // BLE advertising logic mixed in
        esp_ble_gap_start_advertising(&adv_params);
    }
};
```

**Problems:**

* Changing BLE protocol requires touching LED code
* Can't test brightness logic without BLE stack
* Hardware changes affect communication code

### GOOD — Separated Responsibilities (SRP compliant)

```cpp
// 1. Hardware abstraction (one responsibility)
class LedDriver {
private:
    ledc_channel_t m_channel;

public:
    void setBrightness(uint8_t level) {
        ledc_set_duty(LEDC_HIGH_SPEED_MODE, m_channel, level * 32);
        ledc_update_duty(LEDC_HIGH_SPEED_MODE, m_channel);
    }
};

// 2. BLE communication (one responsibility)
class BLELightProfile {
private:
    LightController& m_controller;

public:
    void handleWrite(const uint8_t* data, size_t len) {
        auto cmd = parseCommand(data, len);
        if (cmd) {
            m_controller.sendCommand(*cmd);
        }
    }

private:
    std::optional<LightCommand> parseCommand(const uint8_t* data, size_t len);
};

// 3. Business logic (one responsibility)
class LightController {
private:
    LedDriver& m_driver;
    std::unique_ptr<LightMode> m_currentMode;

public:
    void sendCommand(const LightCommand& cmd) {
        processCommand(cmd);
    }

private:
    void processCommand(const LightCommand& cmd);
};
```

**Benefits:**

* Change LED driver without touching BLE code
* Test business logic with mock hardware
* BLE protocol changes isolated to profile class

---

## Component Separation Example

### BLE stack separation

```cpp
// BLEController — manages BT controller lifecycle
class BLEController {
public:
    esp_err_t init();
    esp_err_t enable();
    esp_err_t disable();
};

// BLEGapManager — manages advertising and device name
class BLEGapManager {
public:
    void setDeviceName(const char* name);
    esp_err_t startAdvertising();
    void handleGapEvent(esp_gap_ble_cb_event_t event);
};

// BLEGattsProfileHandler — handles GATT server operations
class BLEGattsProfileHandler {
public:
    virtual void handleWriteEvent(const WriteEvent& event) = 0;
};

// BLELightControllerProfile — specific profile for light control
class BLELightControllerProfile : public BLEGattsProfileHandler {
public:
    void handleWriteEvent(const WriteEvent& event) override;
};
```

---

## Signs of SRP Violation

* Class has "and" in its description ("manages LEDs **and** parses BLE commands")
* Multiple `#include` groups for unrelated concerns
* Changes to one feature require editing unrelated methods
* Can't test one feature without setting up another
* Class has many private helper methods for different domains

---

## When to Split a Class

Split when:

* The class is doing two distinct jobs
* You need to test one part but not another
* Different team members would work on different methods
* The class has grown beyond ~200-300 lines

Don't split when:

* The class is small and cohesive
* Splitting would create excessive indirection
* The "responsibilities" are tightly coupled by nature

---

## Key Takeaways

* **One reason to change** per class
* **Separate**: hardware drivers, business logic, communication
* **Compose** simple classes into larger systems
* **Test** each responsibility in isolation
* **Name classes** for their single purpose

## Related Best Practices

* [Dependency Inversion](./dependency-inversion.md) — Injecting separated
  components
* [hardware-abstraction.md](../best-practices/hardware-abstraction.md) —
  Separating hardware concerns
