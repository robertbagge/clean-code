# FreeRTOS Patterns

Patterns for task design, queue-based communication, and timing.

---

## Task Design

### Encapsulate task in a class

```cpp
class LightController {
private:
    QueueHandle_t m_commandQueue;
    TaskHandle_t m_taskHandle;
    int m_taskCore;

    static void taskFunction(void* pvParams) {
        auto* self = static_cast<LightController*>(pvParams);
        self->run();
    }

    void run() {
        while (true) {
            LightCommand cmd{};
            if (xQueueReceive(m_commandQueue, &cmd, pdMS_TO_TICKS(100)) == pdPASS) {
                processCommand(cmd);
            }
            vTaskDelay(10);
        }
    }

public:
    void init() {
        m_commandQueue = xQueueCreate(10, sizeof(LightCommand));
        xTaskCreatePinnedToCore(
            taskFunction,
            "LightController",
            4096,              // Stack size
            this,              // Pass this pointer
            5,                 // Priority
            &m_taskHandle,
            m_taskCore
        );
    }

    ~LightController() {
        if (m_taskHandle) vTaskDelete(m_taskHandle);
        if (m_commandQueue) vQueueDelete(m_commandQueue);
    }
};
```

### Clear ownership and lifecycle

```cpp
// Main owns the controller
static std::unique_ptr<LightController> lightController;

extern "C" void app_main() {
    lightController = std::make_unique<LightController>();
    lightController->init();

    // Controller lives for application lifetime
}
```

---

## Queue-Based Communication

### Use queues for inter-task communication

```cpp
// BAD — Shared state with mutex
class Controller {
private:
    LightCommand m_pendingCommand;
    SemaphoreHandle_t m_mutex;

public:
    void setCommand(LightCommand cmd) {
        xSemaphoreTake(m_mutex, portMAX_DELAY);
        m_pendingCommand = cmd;
        xSemaphoreGive(m_mutex);
    }
};

// GOOD — Queue-based message passing
class Controller {
private:
    QueueHandle_t m_queue;

public:
    bool sendCommand(const LightCommand& cmd) {
        return xQueueSend(m_queue, &cmd, pdMS_TO_TICKS(10)) == pdPASS;
    }
};
```

### Type-safe command pattern with std::variant

```cpp
// Define command types
struct TaskLightCommand {
    uint8_t red, green, blue;
};

struct StopLightCommand {};

struct StartSunriseCommand {
    uint32_t duration_s;
    uint8_t maxBrightness;
};

// Union type for all commands
using LightCommand = std::variant<
    TaskLightCommand,
    StopLightCommand,
    StartSunriseCommand
>;

// Process with std::visit
void processCommand(const LightCommand& cmd) {
    std::visit(overloaded{
        [this](const TaskLightCommand& c) {
            m_logger.info("Task light: R=%d G=%d B=%d", c.red, c.green, c.blue);
            setColor(c.red, c.green, c.blue);
        },
        [this](const StopLightCommand&) {
            m_logger.info("Stop command received");
            stop();
        },
        [this](const StartSunriseCommand& c) {
            m_logger.info("Sunrise: %d seconds", c.duration_s);
            startSunrise(c.duration_s);
        }
    }, cmd);
}
```

---

## ISR Safety

### Never block in ISR

```cpp
// BAD — Blocking in ISR
void IRAM_ATTR buttonISR(void* arg) {
    xSemaphoreTake(mutex, portMAX_DELAY);  // WRONG: Blocks!
}

// GOOD — Non-blocking from ISR
void IRAM_ATTR buttonISR(void* arg) {
    BaseType_t xHigherPriorityTaskWoken = pdFALSE;
    xQueueSendFromISR(eventQueue, &event, &xHigherPriorityTaskWoken);
    portYIELD_FROM_ISR(xHigherPriorityTaskWoken);
}
```

### Use ISR-safe variants

| Regular API | ISR-Safe Variant |
|-------------|------------------|
| `xQueueSend` | `xQueueSendFromISR` |
| `xSemaphoreGive` | `xSemaphoreGiveFromISR` |
| `xTaskNotify` | `xTaskNotifyFromISR` |
| `xEventGroupSetBits` | `xEventGroupSetBitsFromISR` |

---

## Timing Patterns

### Use `pdMS_TO_TICKS` for delays

```cpp
// BAD — Magic numbers
vTaskDelay(100);  // What unit? Ticks? Milliseconds?

// GOOD — Clear milliseconds
vTaskDelay(pdMS_TO_TICKS(100));  // 100 ms
```

### Periodic task pattern

```cpp
void periodicTask(void* pvParams) {
    TickType_t xLastWakeTime = xTaskGetTickCount();
    const TickType_t xPeriod = pdMS_TO_TICKS(50);  // 50ms period

    while (true) {
        // Do work...
        updateLEDs();

        // Sleep until next period (more accurate than vTaskDelay)
        vTaskDelayUntil(&xLastWakeTime, xPeriod);
    }
}
```

---

## Task Parameters

### Define constants for task configuration

```cpp
// Centralized task configuration
namespace TaskConfig {
    constexpr size_t LIGHT_STACK_SIZE = 4096;
    constexpr UBaseType_t LIGHT_PRIORITY = 5;
    constexpr size_t LIGHT_QUEUE_SIZE = 10;
    constexpr BaseType_t LIGHT_CORE = 1;

    constexpr size_t BLE_STACK_SIZE = 8192;
    constexpr UBaseType_t BLE_PRIORITY = 10;
    constexpr BaseType_t BLE_CORE = 0;
}

void LightController::init() {
    m_commandQueue = xQueueCreate(
        TaskConfig::LIGHT_QUEUE_SIZE,
        sizeof(LightCommand)
    );

    xTaskCreatePinnedToCore(
        taskFunction,
        "LightController",
        TaskConfig::LIGHT_STACK_SIZE,
        this,
        TaskConfig::LIGHT_PRIORITY,
        &m_taskHandle,
        TaskConfig::LIGHT_CORE
    );
}
```

---

## Resource Cleanup

### Clean up in destructor

```cpp
LightController::~LightController() {
    // Stop task first
    if (m_taskHandle != nullptr) {
        vTaskDelete(m_taskHandle);
        m_taskHandle = nullptr;
    }

    // Then free queue
    if (m_commandQueue != nullptr) {
        vQueueDelete(m_commandQueue);
        m_commandQueue = nullptr;
    }
}
```

### Graceful shutdown pattern

```cpp
class Controller {
private:
    bool m_running = true;

public:
    void stop() {
        m_running = false;
        // Wait for task to finish current iteration
        vTaskDelay(pdMS_TO_TICKS(100));
    }

private:
    void run() {
        while (m_running) {
            // Process...
        }
        // Cleanup and exit
    }
};
```

---

## Anti-patterns to Avoid

### Don't use global shared state

```cpp
// BAD
LightCommand g_currentCommand;  // Race condition!

// GOOD
class Controller {
    QueueHandle_t m_queue;  // Thread-safe communication
};
```

### Don't poll without delay

```cpp
// BAD — Wastes CPU
while (true) {
    if (checkCondition()) {
        doWork();
    }
    // No delay!
}

// GOOD — Yield to other tasks
while (true) {
    if (checkCondition()) {
        doWork();
    }
    vTaskDelay(pdMS_TO_TICKS(10));
}
```

---

## Key Takeaways

* **Encapsulate tasks** in classes with clear ownership
* **Use queues** for inter-task communication, not shared state
* **Never block in ISRs** — use ISR-safe API variants
* **Use `pdMS_TO_TICKS`** for readable timing
* **Clean up resources** in destructors

## Related Best Practices

* [memory-management.md](./memory-management.md) — RAII for FreeRTOS resources
* [c-isolation.md](./c-isolation.md) — Wrapping FreeRTOS in C++ patterns
