# Memory Management

Patterns for safe, efficient memory management in embedded C++.

---

## RAII (Resource Acquisition Is Initialization)

Wrap resources in objects whose destructors clean up automatically:

```cpp
// BAD — Manual cleanup, easy to leak
void processData() {
    uint8_t* buffer = new uint8_t[1024];

    if (someCondition) {
        return;  // LEAK: buffer never freed
    }

    delete[] buffer;
}

// GOOD — RAII with unique_ptr
void processData() {
    auto buffer = std::make_unique<uint8_t[]>(1024);

    if (someCondition) {
        return;  // OK: unique_ptr cleans up automatically
    }
    // No manual delete needed
}
```

---

## Smart Pointers

### Use `std::unique_ptr` for exclusive ownership

```cpp
// Component owns its mode exclusively
class LightController {
private:
    std::unique_ptr<LightMode> m_currentMode;

public:
    void setMode(LightCommand cmd) {
        // Old mode automatically destroyed
        m_currentMode = std::make_unique<OwllarkSunrise>(m_ledInterface, cmd);
    }
};
```

### Factory pattern with ownership transfer

```cpp
std::unique_ptr<LightMode> createMode(ModeType type) {
    switch (type) {
        case ModeType::Sunrise:
            return std::make_unique<OwllarkSunrise>();
        case ModeType::Candle:
            return std::make_unique<OwllarkCandle>();
        default:
            return nullptr;
    }
}

// Caller takes ownership
auto mode = createMode(ModeType::Sunrise);
```

### Use references for non-owning dependencies

```cpp
class Program {
private:
    Clock& m_clock;           // Non-owning reference
    ILogger& m_logger;        // Non-owning reference

public:
    // References injected, caller manages lifetime
    Program(Clock& clock, ILogger& logger)
        : m_clock(clock), m_logger(logger) {}
};
```

---

## Stack vs Heap

### Prefer stack allocation for small, fixed-size data

```cpp
// GOOD — Stack allocated, no heap overhead
void processPacket() {
    std::array<uint8_t, 64> buffer;
    // Use buffer...
}  // Automatically cleaned up

// AVOID — Unnecessary heap allocation
void processPacket() {
    auto buffer = std::make_unique<std::array<uint8_t, 64>>();
    // ...
}
```

### Use heap for dynamic or large allocations

```cpp
// OK — Size determined at runtime
std::unique_ptr<uint8_t[]> createBuffer(size_t size) {
    return std::make_unique<uint8_t[]>(size);
}

// OK — Large buffer that would overflow stack
auto largeBuffer = std::make_unique<std::array<uint8_t, 8192>>();
```

---

## Containers

### Use `std::vector` for dynamic arrays

```cpp
// BAD — Raw array with manual management
uint8_t* data = new uint8_t[count];
// ... use data
delete[] data;

// GOOD — Vector manages memory
std::vector<uint8_t> data(count);
// ... use data
// Automatic cleanup
```

### Use `std::array` for fixed-size arrays

```cpp
// BAD — C-style array, no bounds checking
uint8_t buffer[64];

// GOOD — std::array with bounds checking
std::array<uint8_t, 64> buffer;
buffer.at(i);  // Throws on out-of-bounds
```

---

## Application-Level Ownership

At the application level, use `unique_ptr` for top-level components:

```cpp
// main.cpp
static std::unique_ptr<BLEApplication> bleApp;
static std::unique_ptr<LightController> lightController;
static std::unique_ptr<OwllarkInterface> ledInterface;

extern "C" void app_main() {
    ledInterface = std::make_unique<OwllarkInterface>();
    lightController = std::make_unique<LightController>(*ledInterface);
    bleApp = std::make_unique<BLEApplication>(APP_TAG, DEVICE_NAME);

    // Components live for application lifetime
    bleApp->init();
    bleApp->startAdvertising();
}
```

---

## Avoiding Common Pitfalls

### Never use raw `new`/`delete`

```cpp
// BAD
Widget* w = new Widget();
delete w;

// GOOD
auto w = std::make_unique<Widget>();
```

### Don't mix smart pointers with raw ownership

```cpp
// BAD — Unclear ownership
class Controller {
private:
    Widget* m_widget;  // Who owns this? Who deletes it?
};

// GOOD — Clear ownership
class Controller {
private:
    std::unique_ptr<Widget> m_owned;  // I own it
    Widget& m_borrowed;               // Someone else owns it
};
```

### Avoid `shared_ptr` unless truly needed

`shared_ptr` has overhead (reference counting, thread safety). Use
`unique_ptr` by default; only use `shared_ptr` for true shared ownership:

```cpp
// Prefer unique_ptr
std::unique_ptr<Mode> currentMode;

// Only when multiple owners are required
std::shared_ptr<Config> sharedConfig;
```

---

## Memory-Constrained Patterns

### Preallocate buffers

```cpp
class CommandProcessor {
private:
    std::array<uint8_t, 256> m_buffer;  // Preallocated
    size_t m_bufferLen = 0;

public:
    void receive(const uint8_t* data, size_t len) {
        if (len <= m_buffer.size()) {
            std::memcpy(m_buffer.data(), data, len);
            m_bufferLen = len;
        }
    }
};
```

### Use placement new for memory pools (advanced)

```cpp
// Preallocated memory pool for frequently created objects
alignas(Command) std::array<uint8_t, sizeof(Command) * 10> pool;
size_t poolIndex = 0;

Command* allocateCommand() {
    void* mem = &pool[poolIndex * sizeof(Command)];
    poolIndex++;
    return new(mem) Command();
}
```

---

## Key Takeaways

* **Always use RAII** — resources cleaned up automatically
* **Prefer `std::unique_ptr`** for owned heap objects
* **Use references** for non-owning dependencies
* **Stack-allocate** small, fixed-size data
* **Never use raw `new`/`delete`** in application code

## Related Best Practices

* [c-isolation.md](./c-isolation.md) — RAII wrappers for C resources
* [error-handling.md](./error-handling.md) — Cleanup on error paths
