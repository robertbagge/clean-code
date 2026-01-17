# Memory Management

Patterns for safe, efficient memory management in embedded C++.

---

## Stack First — Heap as Fallback

**The fundamental rule**: Prefer stack allocation. Use heap only when stack
won't work.

### Why stack first?

* **Deterministic**: No allocation failures, no fragmentation
* **Fast**: No allocator overhead, just stack pointer adjustment
* **Safe**: Automatic cleanup when scope exits
* **Predictable**: Memory usage known at compile time

### When heap is acceptable

Use heap **only** when:

1. **Size unknown at compile time** (truly dynamic data)
2. **Object outlives current scope** (ownership transfer)
3. **Polymorphism required** (virtual dispatch needs pointer)
4. **Stack would overflow** (very large buffers, >1KB typical threshold)

```cpp
// STACK FIRST — Default choice
std::array<uint8_t, 256> buffer;           // Fixed size, stack
CommandParser parser;                       // Value type, stack
RGB color{255, 128, 64};                   // Small struct, stack

// HEAP ONLY WHEN NECESSARY
std::unique_ptr<LightMode> mode;           // Polymorphic, needs heap
std::vector<uint8_t> dynamicData(size);    // Runtime size, needs heap
```

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

## Stack Allocation Patterns

### Default to stack for all fixed-size data

```cpp
// GOOD — Stack allocated, deterministic
void processPacket() {
    std::array<uint8_t, 64> buffer;
    PacketHeader header;
    // Use buffer...
}  // Automatic cleanup, no fragmentation

// BAD — Unnecessary heap allocation
void processPacket() {
    auto buffer = std::make_unique<std::array<uint8_t, 64>>();
    auto header = std::make_unique<PacketHeader>();
    // Why heap? Fixed size, doesn't outlive scope
}
```

### Stack-based class members

```cpp
// GOOD — Members on stack (inside object)
class CommandProcessor {
private:
    std::array<uint8_t, 256> m_buffer;  // Stack when object is stack
    CommandParser m_parser;              // Value member, not pointer
    size_t m_bufferLen = 0;
};

// Usage — entire object on stack
void handleCommand() {
    CommandProcessor processor;  // All members stack-allocated
    processor.process(data);
}
```

---

## When Heap Is Necessary

### Polymorphism requires indirection

```cpp
// Heap needed — runtime type selection
class LightController {
private:
    std::unique_ptr<LightMode> m_currentMode;  // Could be Sunrise, Candle, etc.

public:
    void setMode(ModeType type) {
        // Must use heap for polymorphic ownership
        m_currentMode = createMode(type);
    }
};
```

### Ownership transfer across scopes

```cpp
// Heap needed — object outlives creating function
std::unique_ptr<LightMode> createMode(ModeType type) {
    return std::make_unique<OwllarkSunrise>();  // Caller owns result
}
```

### Truly dynamic size

```cpp
// Heap needed — size unknown at compile time
void receiveData(size_t len) {
    std::vector<uint8_t> buffer(len);  // Runtime size requires heap
    // ...
}
```

### Very large buffers

```cpp
// Heap acceptable — would overflow typical 4-8KB task stack
auto largeBuffer = std::make_unique<std::array<uint8_t, 8192>>();
```

---

## Containers

### Prefer `std::array` (stack) over `std::vector` (heap)

```cpp
// BEST — Fixed size known at compile time → std::array (stack)
std::array<uint8_t, 64> buffer;
std::array<LightCommand, 10> commandHistory;

// ACCEPTABLE — Size truly dynamic → std::vector (heap)
std::vector<uint8_t> data(runtimeSize);

// BAD — Using vector when size is fixed
std::vector<uint8_t> buffer(64);  // Why not std::array?
```

### Avoid C-style arrays

```cpp
// BAD — C-style array, no bounds checking, decays to pointer
uint8_t buffer[64];

// GOOD — std::array with bounds checking, doesn't decay
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

### Preallocate fixed buffers as members

```cpp
class CommandProcessor {
private:
    std::array<uint8_t, 256> m_buffer;  // Preallocated, no runtime allocation
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

### Fixed-capacity collections

```cpp
// Instead of std::vector that grows dynamically
template<typename T, size_t Capacity>
class FixedVector {
private:
    std::array<T, Capacity> m_data;
    size_t m_size = 0;

public:
    bool push_back(const T& value) {
        if (m_size >= Capacity) return false;
        m_data[m_size++] = value;
        return true;
    }
    // ...
};

// Usage — all stack, no heap fragmentation
FixedVector<LightCommand, 10> commandQueue;
```

### Object pools on stack (advanced)

```cpp
// Preallocated pool avoids repeated allocation/deallocation
template<typename T, size_t Size>
class ObjectPool {
private:
    std::array<T, Size> m_objects;
    std::array<bool, Size> m_inUse{};

public:
    T* acquire() {
        for (size_t i = 0; i < Size; i++) {
            if (!m_inUse[i]) {
                m_inUse[i] = true;
                return &m_objects[i];
            }
        }
        return nullptr;
    }

    void release(T* obj) {
        size_t idx = obj - m_objects.data();
        if (idx < Size) m_inUse[idx] = false;
    }
};
```

---

## Key Takeaways

* **Stack first** — heap is a fallback, not the default
* **`std::array` over `std::vector`** when size is known
* **Heap only when necessary** — polymorphism, ownership transfer, dynamic size
* **RAII always** — resources cleaned up automatically
* **`std::unique_ptr`** when heap is required
* **Never raw `new`/`delete`** in application code

## Related Best Practices

* [c-isolation.md](./c-isolation.md) — RAII wrappers for C resources
* [error-handling.md](./error-handling.md) — Cleanup on error paths
