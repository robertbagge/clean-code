# Hardware Abstraction

Patterns for HAL layers, GPIO abstractions, and driver encapsulation.

---

## Why Abstract Hardware

* **Testability**: Mock hardware in unit tests
* **Portability**: Swap implementations (ESP32, simulator, test stub)
* **Decoupling**: Business logic doesn't know about GPIO pins
* **Maintainability**: Centralize hardware configuration

---

## Interface-Based Abstraction

### Define abstract interface

```cpp
// interfaces/ILedDriver.h
class ILedDriver {
public:
    virtual ~ILedDriver() = default;

    virtual void setRGB(uint8_t r, uint8_t g, uint8_t b) = 0;
    virtual void setBrightness(uint8_t level) = 0;
    virtual void clear() = 0;
    virtual void refresh() = 0;
};
```

### ESP32 implementation

```cpp
// esp32/components/led/OwllarkLedDriver.h
class OwllarkLedDriver : public ILedDriver {
private:
    ledc_channel_t m_channel;
    gpio_num_t m_pin;

public:
    OwllarkLedDriver(gpio_num_t pin, ledc_channel_t channel);

    void setBrightness(uint8_t level) override {
        uint32_t duty = (level * 8191) / 255;
        ledc_set_duty(LEDC_HIGH_SPEED_MODE, m_channel, duty);
        ledc_update_duty(LEDC_HIGH_SPEED_MODE, m_channel);
    }
    // ... other methods
};
```

### Use in business logic

```cpp
class LightMode {
protected:
    ILedDriver& m_driver;

public:
    explicit LightMode(ILedDriver& driver) : m_driver(driver) {}

    void fadeIn(uint32_t duration_ms) {
        for (int i = 0; i <= 255; i++) {
            m_driver.setBrightness(i);
            vTaskDelay(pdMS_TO_TICKS(duration_ms / 255));
        }
    }
};
```

---

## Multi-Interface Orchestrator

### Compose multiple hardware interfaces

```cpp
// Coordinates multiple LED subsystems
class OwllarkInterface {
private:
    AmbientOwllarkPWMInterface m_ambientPWM;   // Yellow/white LEDs
    AmbientOwllarkRGBInterface m_ambientRGB;   // Neopixel ring
    FrontFacingOwllarkInterface m_frontFacing; // COB LED

public:
    void init() {
        m_ambientPWM.init();
        m_ambientRGB.init();
        m_frontFacing.init();
    }

    void setAmbient(uint8_t yellow, uint8_t white) {
        m_ambientPWM.setYellow(yellow);
        m_ambientPWM.setWhite(white);
    }

    void setRGB(uint8_t r, uint8_t g, uint8_t b) {
        m_ambientRGB.setColor(r, g, b);
    }

    void setFrontFacing(uint8_t brightness) {
        m_frontFacing.setBrightness(brightness);
    }

    void clear() {
        m_ambientPWM.clear();
        m_ambientRGB.clear();
        m_frontFacing.clear();
    }
};
```

---

## GPIO Abstraction

### Type-safe pin definitions

```cpp
// hardware/pins.h
enum class LedPin : gpio_num_t {
    YellowPWM = GPIO_NUM_25,
    WhitePWM = GPIO_NUM_26,
    NeopixelData = GPIO_NUM_27,
    FrontFacing = GPIO_NUM_33
};

enum class ButtonPin : gpio_num_t {
    Main = GPIO_NUM_0,
    Mode = GPIO_NUM_35
};
```

### Pin wrapper class

```cpp
class OutputPin {
private:
    gpio_num_t m_pin;

public:
    explicit OutputPin(LedPin pin)
        : m_pin(static_cast<gpio_num_t>(pin)) {
        gpio_set_direction(m_pin, GPIO_MODE_OUTPUT);
    }

    void set(bool high) {
        gpio_set_level(m_pin, high ? 1 : 0);
    }

    void toggle() {
        int current = gpio_get_level(m_pin);
        gpio_set_level(m_pin, !current);
    }
};
```

---

## PWM Abstraction

### LEDC wrapper

```cpp
class PWMChannel {
private:
    ledc_channel_t m_channel;
    ledc_mode_t m_mode;
    uint32_t m_maxDuty;

public:
    PWMChannel(gpio_num_t pin, ledc_channel_t channel,
               ledc_timer_t timer, uint32_t freq_hz)
        : m_channel(channel)
        , m_mode(LEDC_HIGH_SPEED_MODE)
        , m_maxDuty((1 << 13) - 1)  // 13-bit resolution
    {
        ledc_timer_config_t timer_conf = {
            .speed_mode = m_mode,
            .duty_resolution = LEDC_TIMER_13_BIT,
            .timer_num = timer,
            .freq_hz = freq_hz,
            .clk_cfg = LEDC_AUTO_CLK
        };
        ESP_ERROR_CHECK(ledc_timer_config(&timer_conf));

        ledc_channel_config_t channel_conf = {
            .gpio_num = pin,
            .speed_mode = m_mode,
            .channel = m_channel,
            .timer_sel = timer,
            .duty = 0,
            .hpoint = 0
        };
        ESP_ERROR_CHECK(ledc_channel_config(&channel_conf));
    }

    void setDutyPercent(uint8_t percent) {
        uint32_t duty = (percent * m_maxDuty) / 100;
        ledc_set_duty(m_mode, m_channel, duty);
        ledc_update_duty(m_mode, m_channel);
    }

    void setDuty8Bit(uint8_t value) {
        uint32_t duty = (value * m_maxDuty) / 255;
        ledc_set_duty(m_mode, m_channel, duty);
        ledc_update_duty(m_mode, m_channel);
    }
};
```

---

## Sensor Abstraction

### Abstract sensor interface

```cpp
class ITemperatureSensor {
public:
    virtual ~ITemperatureSensor() = default;
    virtual float readCelsius() = 0;
    virtual bool isAvailable() = 0;
};

// ESP32 internal temperature sensor
class InternalTempSensor : public ITemperatureSensor {
public:
    float readCelsius() override {
        // ESP32 internal sensor implementation
    }

    bool isAvailable() override { return true; }
};

// External I2C sensor
class BME280Sensor : public ITemperatureSensor {
private:
    i2c_port_t m_port;
    uint8_t m_address;

public:
    float readCelsius() override {
        // I2C read implementation
    }

    bool isAvailable() override {
        // Check if device responds
    }
};
```

---

## Configuration Centralization

### Hardware configuration struct

```cpp
// hardware/config.h
namespace HardwareConfig {
    // LED configuration
    constexpr gpio_num_t YELLOW_LED_PIN = GPIO_NUM_25;
    constexpr gpio_num_t WHITE_LED_PIN = GPIO_NUM_26;
    constexpr ledc_channel_t YELLOW_CHANNEL = LEDC_CHANNEL_0;
    constexpr ledc_channel_t WHITE_CHANNEL = LEDC_CHANNEL_1;
    constexpr uint32_t PWM_FREQUENCY_HZ = 5000;

    // Neopixel configuration
    constexpr gpio_num_t NEOPIXEL_PIN = GPIO_NUM_27;
    constexpr size_t NEOPIXEL_COUNT = 24;

    // Task configuration
    constexpr size_t LED_TASK_STACK = 4096;
    constexpr UBaseType_t LED_TASK_PRIORITY = 5;
}
```

### Use configuration in initialization

```cpp
void LightController::init() {
    m_yellowPWM = std::make_unique<PWMChannel>(
        HardwareConfig::YELLOW_LED_PIN,
        HardwareConfig::YELLOW_CHANNEL,
        LEDC_TIMER_0,
        HardwareConfig::PWM_FREQUENCY_HZ
    );

    m_neopixel = std::make_unique<NeopixelRing>(
        HardwareConfig::NEOPIXEL_PIN,
        HardwareConfig::NEOPIXEL_COUNT
    );
}
```

---

## Testability

### Mock hardware for tests

```cpp
class MockLedDriver : public ILedDriver {
public:
    std::vector<std::tuple<uint8_t, uint8_t, uint8_t>> rgbHistory;
    uint8_t lastBrightness = 0;

    void setRGB(uint8_t r, uint8_t g, uint8_t b) override {
        rgbHistory.push_back({r, g, b});
    }

    void setBrightness(uint8_t level) override {
        lastBrightness = level;
    }

    void clear() override { lastBrightness = 0; }
    void refresh() override {}
};

TEST(LightMode, FadeInReachesMaxBrightness) {
    MockLedDriver driver;
    LightMode mode(driver);

    mode.fadeIn(100);

    EXPECT_EQ(driver.lastBrightness, 255);
}
```

---

## Key Takeaways

* **Define abstract interfaces** for hardware access
* **Centralize pin/channel definitions** in config files
* **Use type-safe enums** for GPIO pins
* **Compose interfaces** for complex hardware orchestration
* **Enable testing** with mock implementations

## Related Best Practices

* [Dependency Inversion](../clean-code/dependency-inversion.md) — Interface
  design for HAL
* [c-isolation.md](./c-isolation.md) — Wrapping C driver APIs
* [testing.md](./testing.md) — Mocking hardware
