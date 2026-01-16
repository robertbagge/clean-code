# Interface Segregation Principle (ISP) in C++

## Overview

The Interface Segregation Principle states that clients should not be forced
to depend on interfaces they don't use. In C++, prefer small, focused pure
virtual interfaces over large "god" interfaces.

## Core Concept

In embedded C++, ISP means:

* Define narrow interfaces with few methods
* Clients depend only on methods they actually call
* Split fat interfaces into focused capabilities
* Easier to mock in tests (fewer methods to implement)

## Implementation Example

### BAD — Fat Interface (ISP violation)

```cpp
// God interface with too many responsibilities
class ILightSystem {
public:
    virtual ~ILightSystem() = default;

    // LED control
    virtual void setRGB(uint8_t r, uint8_t g, uint8_t b) = 0;
    virtual void setBrightness(uint8_t level) = 0;
    virtual void clear() = 0;

    // Animation control
    virtual void startFade(uint32_t duration) = 0;
    virtual void stopAnimation() = 0;
    virtual bool isAnimating() = 0;

    // Configuration
    virtual void saveSettings() = 0;
    virtual void loadSettings() = 0;

    // Diagnostics
    virtual float getTemperature() = 0;
    virtual uint32_t getUptime() = 0;
};

// Client only needs LED control, but must implement everything
class SimpleLedController : public ILightSystem {
    // Must implement 10 methods even though it only uses 3!
};
```

### GOOD — Segregated Interfaces (ISP compliant)

```cpp
// Focused interface for basic LED control
class ILedDriver {
public:
    virtual ~ILedDriver() = default;
    virtual void setRGB(uint8_t r, uint8_t g, uint8_t b) = 0;
    virtual void setBrightness(uint8_t level) = 0;
    virtual void clear() = 0;
};

// Focused interface for animations
class IAnimatable {
public:
    virtual ~IAnimatable() = default;
    virtual void startFade(uint32_t duration) = 0;
    virtual void stopAnimation() = 0;
    virtual bool isAnimating() = 0;
};

// Focused interface for persistence
class IPersistable {
public:
    virtual ~IPersistable() = default;
    virtual void save() = 0;
    virtual void load() = 0;
};

// Implementation can choose which interfaces to support
class OwllarkLedDriver : public ILedDriver {
    // Only implements what it needs
};

class AnimatedLedController : public ILedDriver, public IAnimatable {
    // Implements both LED and animation
};
```

---

## Client-Focused Interfaces

### Define interfaces where they're consumed

```cpp
// Business logic defines what it needs
class LightMode {
protected:
    // Only needs basic LED control
    ILedDriver& m_driver;

public:
    explicit LightMode(ILedDriver& driver) : m_driver(driver) {}
};

// Animation logic uses different interface
class FadeController {
private:
    IAnimatable& m_animator;

public:
    explicit FadeController(IAnimatable& animator) : m_animator(animator) {}
};
```

---

## Composition Over Inheritance

### Combine small interfaces as needed

```cpp
// Component implements multiple small interfaces
class FullFeaturedLedSystem
    : public ILedDriver
    , public IAnimatable
    , public IPersistable
{
public:
    // ILedDriver
    void setRGB(uint8_t r, uint8_t g, uint8_t b) override;
    void setBrightness(uint8_t level) override;
    void clear() override;

    // IAnimatable
    void startFade(uint32_t duration) override;
    void stopAnimation() override;
    bool isAnimating() override;

    // IPersistable
    void save() override;
    void load() override;
};

// Can pass to any client expecting a subset
void useLeds(ILedDriver& driver);
void useAnimation(IAnimatable& animator);
void usePersistence(IPersistable& storage);

FullFeaturedLedSystem system;
useLeds(system);       // Only sees ILedDriver methods
useAnimation(system);  // Only sees IAnimatable methods
```

---

## Testing Benefits

### Smaller mocks

```cpp
// Easy to mock — only 3 methods
class MockLedDriver : public ILedDriver {
public:
    MOCK_METHOD(void, setRGB, (uint8_t, uint8_t, uint8_t), (override));
    MOCK_METHOD(void, setBrightness, (uint8_t), (override));
    MOCK_METHOD(void, clear, (), (override));
};

// Compare to mocking a 10-method god interface!
```

---

## Signs of ISP Violation

* Interface has more than 5-7 methods
* Implementations have empty or `throw`-ing methods
* Clients only use a subset of interface methods
* Interface name includes "and" or is very generic ("Manager", "Controller")
* Adding a new client requires adding methods it doesn't need

---

## Splitting an Existing Interface

Before:
```cpp
class IDeviceController {
    virtual void init() = 0;
    virtual void shutdown() = 0;
    virtual void setMode(Mode m) = 0;
    virtual Mode getMode() = 0;
    virtual void saveToDisk() = 0;
    virtual void loadFromDisk() = 0;
};
```

After:
```cpp
class ILifecycle {
    virtual void init() = 0;
    virtual void shutdown() = 0;
};

class IModeController {
    virtual void setMode(Mode m) = 0;
    virtual Mode getMode() = 0;
};

class IPersistable {
    virtual void save() = 0;
    virtual void load() = 0;
};
```

---

## Key Takeaways

* **Small interfaces** (3-5 methods) are easier to implement and mock
* **Clients define** what they need, not providers
* **Compose** multiple interfaces when full functionality needed
* **Split** fat interfaces into focused capabilities
* **Easier testing** with smaller mock implementations

## Related Best Practices

* [Dependency Inversion](./dependency-inversion.md) — Defining consumer-owned
  interfaces
* [Single Responsibility](./single-responsibility.md) — Each interface for one
  purpose
