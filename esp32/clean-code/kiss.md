# Keep It Simple, Stupid (KISS) in C++

## Overview

The KISS principle states that systems work best when they are kept simple.
In embedded C++, this means avoiding over-engineering, premature abstraction,
and unnecessary complexity — especially important given memory and performance
constraints.

## Core Concept

In embedded C++, KISS means:

* Start simple, add complexity only when needed
* Avoid premature abstraction and over-generalization
* Prefer clear, direct code over clever tricks
* Consider memory and performance constraints
* Don't design for hypothetical future requirements

## Implementation Example

### BAD — Over-Engineered (KISS violation)

```cpp
// Over-abstracted for a simple LED toggle
template<typename GPIOPolicy, typename TimingPolicy, typename LoggingPolicy>
class GenericLedController {
    using GPIO = GPIOPolicy;
    using Timing = TimingPolicy;
    using Logger = LoggingPolicy;

    GPIO m_gpio;
    Timing m_timing;
    Logger m_logger;

public:
    template<typename... Args>
    void toggle(Args&&... args) {
        m_logger.log("Toggling LED");
        auto state = m_gpio.getState();
        m_gpio.setState(!state);
        m_timing.delay(std::forward<Args>(args)...);
    }
};

// 50 lines of template code for one GPIO toggle!
```

### GOOD — Simple and Direct (KISS compliant)

```cpp
class Led {
private:
    gpio_num_t m_pin;
    bool m_state = false;

public:
    explicit Led(gpio_num_t pin) : m_pin(pin) {
        gpio_set_direction(m_pin, GPIO_MODE_OUTPUT);
    }

    void toggle() {
        m_state = !m_state;
        gpio_set_level(m_pin, m_state);
    }

    void set(bool on) {
        m_state = on;
        gpio_set_level(m_pin, m_state);
    }
};

// 15 lines, does exactly what's needed
```

---

## Avoid Premature Abstraction

### Wait until you see the pattern

```cpp
// First implementation — just make it work
class SunriseMode {
    void update() {
        // Direct implementation
        float progress = getProgress();
        uint8_t brightness = progress * 255;
        m_driver.setBrightness(brightness);
    }
};

// Only abstract when you have 3+ similar cases
// AND they need to vary independently
```

### Don't abstract for hypothetical futures

```cpp
// BAD — "We might need different curve functions someday"
class BrightnessCurveStrategyFactoryBuilder { ... };

// GOOD — Just use what you need now
uint8_t calculateBrightness(float progress, CurveType curve) {
    switch (curve) {
        case CurveType::Linear:
            return progress * 255;
        case CurveType::EaseInOut:
            // Simple formula
            break;
    }
}
```

---

## Prefer Clear Over Clever

### BAD — Clever but obscure

```cpp
// Bit manipulation trick
uint8_t brightness = (level >> 3) | ((level & 0x7) << 5);

// Operator overloading abuse
auto result = led << 5 >> color | brightness;

// Template metaprogramming for simple task
template<int N, typename = std::enable_if_t<(N > 0)>>
struct LedArray { ... };
```

### GOOD — Clear and readable

```cpp
// Named constants
constexpr uint8_t BRIGHTNESS_SHIFT = 3;
uint8_t brightness = level >> BRIGHTNESS_SHIFT;

// Explicit method calls
led.setBrightness(brightness);
led.setColor(color);

// Simple class
class LedArray {
    std::array<Led, MAX_LEDS> m_leds;
};
```

---

## Memory-Conscious Simplicity

### Embedded constraints favor simplicity

```cpp
// BAD — STL containers with dynamic allocation
std::vector<LightCommand> commandHistory;
std::unordered_map<std::string, Handler> handlers;

// GOOD — Fixed-size alternatives
std::array<LightCommand, 10> commandHistory;
size_t historyCount = 0;

// Or simple struct
struct HandlerEntry {
    const char* name;
    Handler handler;
};
static constexpr HandlerEntry handlers[] = { ... };
```

---

## Complexity Budget

### Add complexity only where it pays off

**Worth the complexity:**

* Dependency injection for testability
* Interface abstraction for hardware mockability
* Error handling for robustness

**Usually not worth it:**

* Generic templates for single-use code
* Factory patterns for 2-3 types
* Observer patterns for single subscriber
* Builder patterns for simple objects

```cpp
// Simple construction is fine
LightCommand cmd{255, 128, 64, 1000};

// Builder is overkill
auto cmd = LightCommand::builder()
    .withRed(255)
    .withGreen(128)
    .withBlue(64)
    .withDuration(1000)
    .build();
```

---

## Signs of Over-Engineering

* Class has "Factory", "Manager", "Handler", "Builder" in name but does one
  simple thing
* Template parameters that are never varied
* Interfaces with only one implementation (and no tests)
* Abstraction layers that just pass through calls
* More setup code than actual logic

---

## Right-Sizing Abstractions

### Match abstraction to actual need

```cpp
// If you only have 3 light modes, a switch is fine
void processMode(ModeType mode) {
    switch (mode) {
        case ModeType::Sunrise: handleSunrise(); break;
        case ModeType::Candle: handleCandle(); break;
        case ModeType::Breathing: handleBreathing(); break;
    }
}

// Only introduce polymorphism when:
// - You have 5+ variations
// - Variations are added by different people/times
// - Each variation needs independent testing
```

---

## Refactoring Toward Simplicity

### Questions to ask:

1. Can I remove this abstraction without losing functionality?
2. Does this generalization solve a real problem or a hypothetical one?
3. Would a junior developer understand this in 5 minutes?
4. Is the code longer than it needs to be?

```cpp
// Before: Over-abstracted
class ICommand { virtual void execute() = 0; };
class SetBrightnessCommand : public ICommand { ... };
class CommandProcessor { void process(ICommand& cmd); };

// After: Simple function call
void setBrightness(uint8_t level) {
    m_driver.setBrightness(level);
}
```

---

## Key Takeaways

* **Start simple**, add complexity only when needed
* **Avoid premature abstraction** — wait for patterns to emerge
* **Prefer clear code** over clever tricks
* **Consider constraints** — memory, performance, maintainability
* **Right-size abstractions** to actual requirements

## Related Best Practices

* [DRY](./dry.md) — Balance with avoiding over-abstraction
* [memory-management.md](../best-practices/memory-management.md) — Simple
  memory patterns
