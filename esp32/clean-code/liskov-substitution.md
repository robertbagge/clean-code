# Liskov Substitution Principle (LSP) in C++

## Overview

The Liskov Substitution Principle states that derived classes must be
substitutable for their base classes. Code using a base class reference should
work correctly with any derived class without knowing the difference.

## Core Concept

In embedded C++, LSP means:

* Derived classes honor base class contracts
* Override methods don't change expected behavior
* Preconditions can be weakened, postconditions strengthened
* Mocks and real implementations behave consistently

## Implementation Example

### BAD — Broken Substitutability (LSP violation)

```cpp
class ILedDriver {
public:
    virtual ~ILedDriver() = default;
    virtual void setBrightness(uint8_t level) = 0;  // 0-255
    virtual uint8_t getBrightness() = 0;
};

class PWMLedDriver : public ILedDriver {
public:
    void setBrightness(uint8_t level) override {
        m_brightness = level;
        ledc_set_duty(...);
    }

    uint8_t getBrightness() override {
        return m_brightness;
    }
};

class DimmableLedDriver : public ILedDriver {
public:
    void setBrightness(uint8_t level) override {
        // VIOLATION: Clamps input differently than base contract
        m_brightness = (level > 200) ? 200 : level;  // Max 200!
    }

    uint8_t getBrightness() override {
        return m_brightness;
    }
};
```

**Problems:**

* Code expecting `setBrightness(255)` to work gets unexpected behavior
* Caller can't rely on contract (0-255 range)
* Mocks might pass but real implementation fails

### GOOD — Consistent Behavior (LSP compliant)

```cpp
class ILedDriver {
public:
    virtual ~ILedDriver() = default;

    // Contract: level 0-255, always accepted
    virtual void setBrightness(uint8_t level) = 0;
    virtual uint8_t getBrightness() = 0;
};

class PWMLedDriver : public ILedDriver {
private:
    uint8_t m_brightness = 0;

public:
    void setBrightness(uint8_t level) override {
        m_brightness = level;
        // Full 0-255 range supported
    }

    uint8_t getBrightness() override {
        return m_brightness;
    }
};

class LimitedLedDriver : public ILedDriver {
private:
    uint8_t m_brightness = 0;
    uint8_t m_maxBrightness;

public:
    explicit LimitedLedDriver(uint8_t maxBrightness)
        : m_maxBrightness(maxBrightness) {}

    void setBrightness(uint8_t level) override {
        // Scale to available range, honoring contract
        m_brightness = level;
        uint8_t scaled = (level * m_maxBrightness) / 255;
        applyHardware(scaled);
    }

    uint8_t getBrightness() override {
        return m_brightness;  // Returns what was set
    }
};
```

**Benefits:**

* All implementations honor 0-255 contract
* Callers can rely on consistent behavior
* Limited driver scales internally but honors interface

---

## Mock Compatibility

### Mocks must behave like real implementations

```cpp
// Real implementation
class ESP32Clock : public Clock {
public:
    unsigned long now() const override {
        return esp_timer_get_time() / 1000;
    }
};

// Mock must honor same contract
class MockClock : public Clock {
private:
    unsigned long m_time = 0;

public:
    unsigned long now() const override {
        return m_time;  // Same contract: returns milliseconds
    }

    void setTime(unsigned long time) { m_time = time; }
};

// Tests pass with mock, work with real
TEST(Program, TimingWorks) {
    MockClock clock;
    Program program(clock);

    clock.setTime(0);
    program.start();

    clock.setTime(1000);
    EXPECT_TRUE(program.hasRunFor(1000));
}
```

---

## Error Handling Consistency

### All implementations handle errors the same way

```cpp
class IStorage {
public:
    virtual ~IStorage() = default;

    // Contract: Returns nullopt if key not found
    virtual std::optional<std::string> get(const std::string& key) = 0;

    // Contract: Throws on write failure
    virtual void set(const std::string& key, const std::string& value) = 0;
};

class NvsStorage : public IStorage {
public:
    std::optional<std::string> get(const std::string& key) override {
        // Honors contract: nullopt for missing key
        esp_err_t err = nvs_get_str(...);
        if (err == ESP_ERR_NVS_NOT_FOUND) {
            return std::nullopt;
        }
        if (err != ESP_OK) {
            throw StorageException("NVS read error");
        }
        return value;
    }
};

class MockStorage : public IStorage {
private:
    std::unordered_map<std::string, std::string> m_data;

public:
    std::optional<std::string> get(const std::string& key) override {
        // Honors same contract
        auto it = m_data.find(key);
        if (it == m_data.end()) {
            return std::nullopt;
        }
        return it->second;
    }
};
```

---

## Virtual Method Overrides

### Don't weaken postconditions

```cpp
// BAD — Weakens postcondition
class BaseSensor {
public:
    // Contract: Returns temperature in Celsius, always valid
    virtual float readTemperature() = 0;
};

class BrokenSensor : public BaseSensor {
public:
    float readTemperature() override {
        // VIOLATION: Returns NaN on error instead of valid value
        if (!isConnected()) {
            return std::numeric_limits<float>::quiet_NaN();
        }
        return actualRead();
    }
};

// GOOD — Honors postcondition
class SafeSensor : public BaseSensor {
public:
    float readTemperature() override {
        if (!isConnected()) {
            throw SensorException("Not connected");
        }
        return actualRead();
    }
};
```

---

## Signs of LSP Violation

* Type checks before using derived class (`if (dynamic_cast<Derived*>(...)`)
* Different error handling in derived classes
* Derived class throws where base doesn't
* Some methods "don't apply" to derived class
* Tests pass with mock but fail with real implementation

---

## Designing for LSP

1. **Document contracts** in base class
2. **Test with multiple implementations** to verify consistency
3. **Prefer composition** when substitution doesn't make sense
4. **Use `override`** to catch signature mismatches
5. **Mark final** classes that shouldn't be extended

```cpp
class IDriver {
public:
    virtual ~IDriver() = default;

    /**
     * @brief Set output level
     * @param level Value 0-255 (always accepted)
     * @post getBrightness() returns level
     */
    virtual void setBrightness(uint8_t level) = 0;
};

class ConcreteDriver final : public IDriver {
    // final prevents broken substitution
};
```

---

## Key Takeaways

* **Derived classes** must honor base class contracts
* **Mocks** should behave like real implementations
* **Error handling** must be consistent across implementations
* **Don't weaken postconditions** in overrides
* **Test substitutability** with multiple implementations

## Related Best Practices

* [Open-Closed](./open-closed.md) — Extension through substitutable classes
* [testing.md](../best-practices/testing.md) — Mocks as substitutes
