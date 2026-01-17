# Open-Closed Principle (OCP) in C++

## Overview

The Open-Closed Principle states that software entities should be open for
extension but closed for modification. In C++, this is achieved through
inheritance, virtual methods, and the template method pattern.

## Core Concept

In embedded C++, OCP means:

* Add new behavior by creating new classes, not modifying existing ones
* Use virtual methods for customization points
* Base classes define the algorithm structure
* Derived classes provide specific implementations

## Implementation Example

### BAD — Modification Required (OCP violation)

```cpp
class LightController {
public:
    void startMode(ModeType type) {
        switch (type) {
            case ModeType::Sunrise:
                // Sunrise implementation
                break;
            case ModeType::Candle:
                // Candle implementation
                break;
            case ModeType::Breathing:
                // Breathing implementation
                break;
            // VIOLATION: Must modify this switch for every new mode!
        }
    }
};
```

**Problems:**

* Adding new mode requires modifying `LightController`
* Switch statement grows unbounded
* All modes coupled to controller class
* Difficult to test individual modes

### GOOD — Extension Without Modification (OCP compliant)

```cpp
// Base class defines interface (closed for modification)
class LightMode {
protected:
    ILedDriver& m_driver;

public:
    explicit LightMode(ILedDriver& driver) : m_driver(driver) {}
    virtual ~LightMode() = default;

    virtual void start() = 0;
    virtual void stop() = 0;
    virtual void update() = 0;
};

// Extensions (open for extension)
class SunriseMode : public LightMode {
public:
    using LightMode::LightMode;

    void start() override {
        m_startTime = millis();
    }

    void update() override {
        // Sunrise-specific logic
    }

    void stop() override {
        m_driver.clear();
    }
};

class CandleMode : public LightMode {
    // Candle-specific implementation
};

class BreathingMode : public LightMode {
    // Breathing-specific implementation
};

// Controller uses base class (closed for modification)
class LightController {
private:
    std::unique_ptr<LightMode> m_currentMode;

public:
    void setMode(std::unique_ptr<LightMode> mode) {
        if (m_currentMode) {
            m_currentMode->stop();
        }
        m_currentMode = std::move(mode);
        m_currentMode->start();
    }
};
```

**Benefits:**

* Add new modes without touching existing code
* Each mode is independently testable
* Controller is stable — never needs modification

---

## Template Method Pattern

### Base class defines algorithm structure

```cpp
class BLEGattsProfileHandler {
public:
    // Template method — defines the algorithm
    void handleEvent(esp_gatts_cb_event_t event, esp_gatt_if_t gatts_if,
                     esp_ble_gatts_cb_param_t* param) {
        switch (event) {
            case ESP_GATTS_WRITE_EVT:
                // Common pre-processing
                logWrite(param);
                // Call customization point
                handleWriteImpl(param->write);
                // Common post-processing
                sendResponse(gatts_if, param);
                break;
            // ... other events
        }
    }

protected:
    // Customization point — derived classes override
    virtual void handleWriteImpl(
        const esp_ble_gatts_cb_param_t::write& write) = 0;
};

// Extension — specific profile
class BLELightControllerProfile : public BLEGattsProfileHandler {
protected:
    void handleWriteImpl(const esp_ble_gatts_cb_param_t::write& write) override {
        auto cmd = parseCommand(write.value, write.len);
        if (cmd) {
            m_controller.sendCommand(*cmd);
        }
    }
};
```

---

## Factory Pattern for Extension

### Create modes without switch statements

```cpp
// Mode factory (closed for modification)
class LightModeFactory {
public:
    using Creator = std::function<std::unique_ptr<LightMode>(ILedDriver&)>;

private:
    std::unordered_map<std::string, Creator> m_creators;

public:
    void registerMode(const std::string& name, Creator creator) {
        m_creators[name] = std::move(creator);
    }

    std::unique_ptr<LightMode> create(const std::string& name,
                                       ILedDriver& driver) {
        auto it = m_creators.find(name);
        if (it != m_creators.end()) {
            return it->second(driver);
        }
        return nullptr;
    }
};

// Registration (open for extension)
void registerModes(LightModeFactory& factory) {
    factory.registerMode("sunrise", [](ILedDriver& d) {
        return std::make_unique<SunriseMode>(d);
    });
    factory.registerMode("candle", [](ILedDriver& d) {
        return std::make_unique<CandleMode>(d);
    });
    // Add new modes here without modifying factory
}
```

---

## Strategy Pattern

### Swap algorithms without modification

```cpp
// Algorithm interface
class IBrightnessStrategy {
public:
    virtual ~IBrightnessStrategy() = default;
    virtual uint8_t calculate(float progress) = 0;
};

// Strategies
class LinearBrightness : public IBrightnessStrategy {
public:
    uint8_t calculate(float progress) override {
        return static_cast<uint8_t>(255 * progress);
    }
};

class EaseInOutBrightness : public IBrightnessStrategy {
public:
    uint8_t calculate(float progress) override {
        float t = progress < 0.5f
            ? 2 * progress * progress
            : 1 - std::pow(-2 * progress + 2, 2) / 2;
        return static_cast<uint8_t>(255 * t);
    }
};

// User of strategy (closed for modification)
class FadeController {
private:
    std::unique_ptr<IBrightnessStrategy> m_strategy;

public:
    void setStrategy(std::unique_ptr<IBrightnessStrategy> strategy) {
        m_strategy = std::move(strategy);
    }

    uint8_t getBrightness(float progress) {
        return m_strategy->calculate(progress);
    }
};
```

---

## When OCP Applies

Use OCP when:

* You expect multiple variations of behavior
* A switch statement keeps growing
* New features require editing stable code
* You want to test variations independently

Don't over-apply:

* Simple code that won't change
* Internal implementation details
* One-off scripts or tools

---

## Key Takeaways

* **Add behavior** by creating new classes, not editing existing ones
* **Virtual methods** provide customization points
* **Template method** defines algorithm structure with hooks
* **Factory pattern** creates instances without switch statements
* **Strategy pattern** swaps algorithms at runtime

## Related Best Practices

* [Liskov Substitution](./liskov-substitution.md) — Derived classes must be
  substitutable
* [Dependency Inversion](./dependency-inversion.md) — Depend on abstractions
