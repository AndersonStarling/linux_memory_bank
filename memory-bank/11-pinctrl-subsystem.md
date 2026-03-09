# Pinctrl & Pinmux Subsystem

## Pinctrl Architecture Overview

Pinctrl (Pin Control) subsystem quản lý cấu hình pin multiplexing và các thuộc tính pin như pull-up/down, drive strength, bias. Đây là subsystem cực kỳ quan trọng vì mọi peripheral đều cần pin configuration đúng.

```
┌─────────────────────────────────────────┐
│   Client Device (I2C, SPI, etc.)        │  <- pinctrl_get(), pinctrl_select_state()
├─────────────────────────────────────────┤
│   Pinctrl Core (pinctrl.c)               │  <- State management
├─────────────────────────────────────────┤
│   Pinctrl Driver (pinctrl-single)       │  <- Hardware-specific implementation
└─────────────────────────────────────────┘
```

### Key Concepts
- **Pinctrl**: Configuration của pins (pull-up, drive strength, etc.)
- **Pinmux**: Multiplexing pins cho các functions khác nhau
- **Pin States**: Named configurations (default, active, idle, sleep)
- **Pin Groups**: Tập hợp pins cùng function
- **Pin Functions**: Chức năng cụ thể của pin group (uart, i2c, spi, gpio)

## Essential Headers

```c
// For pinctrl consumers (devices using pinctrl)
#include <linux/pinctrl/consumer.h>

// For pinctrl drivers (implementing pin controller)
#include <linux/pinctrl/pinctrl.h>
#include <linux/pinctrl/pinmux.h>
#include <linux/pinctrl/pinconf.h>
#include <linux/pinctrl/pinconf-generic.h>
```

## Pinctrl Consumer API

### Requesting Pinctrl States

```c
#include <linux/pinctrl/consumer.h>

struct device_data {
    struct pinctrl *pctrl;
    struct pinctrl_state *pins_default;
    struct pinctrl_state *pins_sleep;
};

static int mydev_probe(struct platform_device *pdev)
{
    struct device_data *data;
    int ret;
    
    data = devm_kzalloc(&pdev->dev, sizeof(*data), GFP_KERNEL);
    if (!data)
        return -ENOMEM;
    
    // Get pinctrl handle
    data->pctrl = devm_pinctrl_get(&pdev->dev);
    if (IS_ERR(data->pctrl)) {
        ret = PTR_ERR(data->pctrl);
        dev_err(&pdev->dev, "Failed to get pinctrl: %d\n", ret);
        return ret;
    }
    
    // Lookup named states
    data->pins_default = pinctrl_lookup_state(data->pctrl, PINCTRL_STATE_DEFAULT);
    if (IS_ERR(data->pins_default)) {
        dev_err(&pdev->dev, "No default pinctrl state\n");
        return PTR_ERR(data->pins_default);
    }
    
    data->pins_sleep = pinctrl_lookup_state(data->pctrl, PINCTRL_STATE_SLEEP);
    if (IS_ERR(data->pins_sleep)) {
        dev_warn(&pdev->dev, "No sleep pinctrl state\n");
        data->pins_sleep = NULL;
    }
    
    // Select default state
    ret = pinctrl_select_state(data->pctrl, data->pins_default);
    if (ret) {
        dev_err(&pdev->dev, "Failed to select default state\n");
        return ret;
    }
    
    return 0;
}
```

### Common Pinctrl State Names

```c
// Predefined state names (linux/pinctrl/consumer.h)
#define PINCTRL_STATE_DEFAULT       "default"
#define PINCTRL_STATE_IDLE          "idle"
#define PINCTRL_STATE_SLEEP         "sleep"

// Custom state names (device-specific)
// "active", "inactive", "uart", "gpio", etc.
```

### Power Management with Pinctrl

```c
#ifdef CONFIG_PM_SLEEP
static int mydev_suspend(struct device *dev)
{
    struct device_data *data = dev_get_drvdata(dev);
    
    // Switch to sleep state
    if (data->pins_sleep) {
        pinctrl_select_state(data->pctrl, data->pins_sleep);
    }
    
    return 0;
}

static int mydev_resume(struct device *dev)
{
    struct device_data *data = dev_get_drvdata(dev);
    
    // Restore default/active state
    pinctrl_select_state(data->pctrl, data->pins_default);
    
    return 0;
}

static SIMPLE_DEV_PM_OPS(mydev_pm_ops, mydev_suspend, mydev_resume);
#endif
```

### Automatic Pinctrl Management

```c
// Core framework automatically manages pinctrl for devices
// No manual pinctrl code needed in most cases!

static int mydev_probe(struct platform_device *pdev)
{
    // Core automatically selects "default" state during probe
    // if pinctrl-0 is defined in device tree
    
    // Core automatically manages state during suspend/resume
    // if "sleep" state is defined
    
    return 0;
}

// Only manual management needed for:
// - Custom states (not default/idle/sleep)
// - Runtime state switching
// - Error recovery
```

## Device Tree Bindings

### Pinctrl Consumer (Client Device)

```dts
&i2c0 {
    status = "okay";
    
    // Pinctrl configuration
    pinctrl-names = "default", "sleep";
    pinctrl-0 = <&i2c0_pins_default>;
    pinctrl-1 = <&i2c0_pins_sleep>;
    
    // I2C devices...
};

&uart0 {
    pinctrl-names = "default";
    pinctrl-0 = <&uart0_pins>;
};

&spi0 {
    pinctrl-names = "default", "gpio";
    pinctrl-0 = <&spi0_pins>;      // SPI function
    pinctrl-1 = <&spi0_pins_gpio>; // GPIO fallback
};
```

### Pinctrl Provider (Pin Controller)

```dts
// AM33xx pinctrl controller example
am33xx_pinmux: pinmux@44e10800 {
    compatible = "pinctrl-single";
    reg = <0x44e10800 0x0238>;
    #pinctrl-cells = <1>;
    pinctrl-single,register-width = <32>;
    pinctrl-single,function-mask = <0x7f>;
};

// Pin configurations
&am33xx_pinmux {
    // I2C0 pins configuration
    i2c0_pins_default: pinmux_i2c0_pins_default {
        pinctrl-single,pins = <
            AM33XX_IOPAD(0x988, PIN_INPUT_PULLUP | MUX_MODE0)  // i2c0_sda
            AM33XX_IOPAD(0x98c, PIN_INPUT_PULLUP | MUX_MODE0)  // i2c0_scl
        >;
    };
    
    i2c0_pins_sleep: pinmux_i2c0_pins_sleep {
        pinctrl-single,pins = <
            AM33XX_IOPAD(0x988, PIN_INPUT_PULLDOWN | MUX_MODE7) // gpio mode
            AM33XX_IOPAD(0x98c, PIN_INPUT_PULLDOWN | MUX_MODE7)
        >;
    };
    
    // UART0 pins
    uart0_pins: pinmux_uart0_pins {
        pinctrl-single,pins = <
            AM33XX_IOPAD(0x970, PIN_INPUT_PULLUP | MUX_MODE0)   // uart0_rxd
            AM33XX_IOPAD(0x974, PIN_OUTPUT_PULLDOWN | MUX_MODE0) // uart0_txd
        >;
    };
    
    // SPI0 pins
    spi0_pins: pinmux_spi0_pins {
        pinctrl-single,pins = <
            AM33XX_IOPAD(0x950, PIN_INPUT_PULLUP | MUX_MODE0)   // spi0_sclk
            AM33XX_IOPAD(0x954, PIN_INPUT_PULLUP | MUX_MODE0)   // spi0_d0
            AM33XX_IOPAD(0x958, PIN_OUTPUT_PULLUP | MUX_MODE0)  // spi0_d1
            AM33XX_IOPAD(0x95c, PIN_OUTPUT_PULLUP | MUX_MODE0)  // spi0_cs0
        >;
    };
};
```

### Pin Configuration Flags

```c
// Common pin configuration flags (include/dt-bindings/pinctrl/*)

// Direction
PIN_INPUT              // Input pin
PIN_OUTPUT             // Output pin

// Pull resistors
PIN_INPUT_PULLUP       // Input with pull-up
PIN_INPUT_PULLDOWN     // Input with pull-down
PIN_OUTPUT_PULLUP      // Output with pull-up
PIN_OUTPUT_PULLDOWN    // Output with pull-down

// Drive strength
PIN_OUTPUT_PULLUP_DRV  // Output with pull-up and drive
PIN_OUTPUT_PULLDOWN_DRV

// Mux modes (hardware-specific)
MUX_MODE0              // Function 0
MUX_MODE1              // Function 1
MUX_MODE7              // Usually GPIO mode
```

## Pinctrl Driver Implementation

### Basic Pinctrl Driver

```c
#include <linux/pinctrl/pinctrl.h>
#include <linux/pinctrl/pinmux.h>
#include <linux/pinctrl/pinconf.h>
#include <linux/pinctrl/pinconf-generic.h>

struct my_pinctrl {
    struct device *dev;
    void __iomem *base;
    struct pinctrl_dev *pctl;
};

// Pin descriptions
static const struct pinctrl_pin_desc my_pins[] = {
    PINCTRL_PIN(0, "GPIO0"),
    PINCTRL_PIN(1, "GPIO1"),
    PINCTRL_PIN(2, "GPIO2"),
    // ... more pins
};

// Pin groups
static const unsigned int uart0_pins[] = { 10, 11 };
static const unsigned int i2c0_pins[] = { 12, 13 };
static const unsigned int spi0_pins[] = { 20, 21, 22, 23 };

static const struct pingroup my_groups[] = {
    PINCTRL_PINGROUP("uart0_grp", uart0_pins, ARRAY_SIZE(uart0_pins)),
    PINCTRL_PINGROUP("i2c0_grp", i2c0_pins, ARRAY_SIZE(i2c0_pins)),
    PINCTRL_PINGROUP("spi0_grp", spi0_pins, ARRAY_SIZE(spi0_pins)),
};

// Functions
static const char * const uart_groups[] = { "uart0_grp" };
static const char * const i2c_groups[] = { "i2c0_grp" };
static const char * const spi_groups[] = { "spi0_grp" };

static const struct pinfunction my_functions[] = {
    PINCTRL_PINFUNCTION("uart", uart_groups, ARRAY_SIZE(uart_groups)),
    PINCTRL_PINFUNCTION("i2c", i2c_groups, ARRAY_SIZE(i2c_groups)),
    PINCTRL_PINFUNCTION("spi", spi_groups, ARRAY_SIZE(spi_groups)),
};

// Pinctrl operations
static int my_pctl_get_groups_count(struct pinctrl_dev *pctldev)
{
    return ARRAY_SIZE(my_groups);
}

static const char *my_pctl_get_group_name(struct pinctrl_dev *pctldev,
                                           unsigned selector)
{
    return my_groups[selector].name;
}

static int my_pctl_get_group_pins(struct pinctrl_dev *pctldev,
                                   unsigned selector,
                                   const unsigned **pins,
                                   unsigned *num_pins)
{
    *pins = my_groups[selector].pins;
    *num_pins = my_groups[selector].npins;
    return 0;
}

static const struct pinctrl_ops my_pctl_ops = {
    .get_groups_count = my_pctl_get_groups_count,
    .get_group_name = my_pctl_get_group_name,
    .get_group_pins = my_pctl_get_group_pins,
    .dt_node_to_map = pinconf_generic_dt_node_to_map_all,
    .dt_free_map = pinconf_generic_dt_free_map,
};

// Pinmux operations
static int my_pmx_get_functions_count(struct pinctrl_dev *pctldev)
{
    return ARRAY_SIZE(my_functions);
}

static const char *my_pmx_get_function_name(struct pinctrl_dev *pctldev,
                                             unsigned selector)
{
    return my_functions[selector].name;
}

static int my_pmx_get_function_groups(struct pinctrl_dev *pctldev,
                                       unsigned selector,
                                       const char * const **groups,
                                       unsigned *num_groups)
{
    *groups = my_functions[selector].groups;
    *num_groups = my_functions[selector].ngroups;
    return 0;
}

static int my_pmx_set_mux(struct pinctrl_dev *pctldev,
                          unsigned func_selector,
                          unsigned group_selector)
{
    struct my_pinctrl *pctl = pinctrl_dev_get_drvdata(pctldev);
    
    // Program hardware registers to set mux function
    // Example: write function value to mux register
    
    dev_dbg(pctl->dev, "Set function %s for group %s\n",
            my_functions[func_selector].name,
            my_groups[group_selector].name);
    
    return 0;
}

static const struct pinmux_ops my_pmx_ops = {
    .get_functions_count = my_pmx_get_functions_count,
    .get_function_name = my_pmx_get_function_name,
    .get_function_groups = my_pmx_get_function_groups,
    .set_mux = my_pmx_set_mux,
};

// Pin configuration operations
static int my_pinconf_set(struct pinctrl_dev *pctldev,
                          unsigned pin,
                          unsigned long *configs,
                          unsigned num_configs)
{
    struct my_pinctrl *pctl = pinctrl_dev_get_drvdata(pctldev);
    int i;
    
    for (i = 0; i < num_configs; i++) {
        enum pin_config_param param = pinconf_to_config_param(configs[i]);
        u32 arg = pinconf_to_config_argument(configs[i]);
        
        switch (param) {
        case PIN_CONFIG_BIAS_DISABLE:
            // Disable pull-up/down
            break;
        case PIN_CONFIG_BIAS_PULL_UP:
            // Enable pull-up
            break;
        case PIN_CONFIG_BIAS_PULL_DOWN:
            // Enable pull-down
            break;
        case PIN_CONFIG_DRIVE_STRENGTH:
            // Set drive strength (arg in mA)
            break;
        case PIN_CONFIG_INPUT_ENABLE:
            // Enable input
            break;
        case PIN_CONFIG_OUTPUT:
            // Set as output with value (arg)
            break;
        default:
            return -ENOTSUPP;
        }
    }
    
    return 0;
}

static const struct pinconf_ops my_pinconf_ops = {
    .pin_config_set = my_pinconf_set,
    .is_generic = true,
};

// Probe function
static int my_pinctrl_probe(struct platform_device *pdev)
{
    struct my_pinctrl *pctl;
    struct pinctrl_desc *pctl_desc;
    struct resource *res;
    int ret;
    
    pctl = devm_kzalloc(&pdev->dev, sizeof(*pctl), GFP_KERNEL);
    if (!pctl)
        return -ENOMEM;
    
    pctl->dev = &pdev->dev;
    
    res = platform_get_resource(pdev, IORESOURCE_MEM, 0);
    pctl->base = devm_ioremap_resource(&pdev->dev, res);
    if (IS_ERR(pctl->base))
        return PTR_ERR(pctl->base);
    
    // Setup pinctrl descriptor
    pctl_desc = devm_kzalloc(&pdev->dev, sizeof(*pctl_desc), GFP_KERNEL);
    if (!pctl_desc)
        return -ENOMEM;
    
    pctl_desc->name = dev_name(&pdev->dev);
    pctl_desc->pins = my_pins;
    pctl_desc->npins = ARRAY_SIZE(my_pins);
    pctl_desc->pctlops = &my_pctl_ops;
    pctl_desc->pmxops = &my_pmx_ops;
    pctl_desc->confops = &my_pinconf_ops;
    pctl_desc->owner = THIS_MODULE;
    
    // Register pinctrl device
    ret = devm_pinctrl_register_and_init(&pdev->dev, pctl_desc,
                                          pctl, &pctl->pctl);
    if (ret) {
        dev_err(&pdev->dev, "Failed to register pinctrl\n");
        return ret;
    }
    
    ret = pinctrl_enable(pctl->pctl);
    if (ret) {
        dev_err(&pdev->dev, "Failed to enable pinctrl\n");
        return ret;
    }
    
    platform_set_drvdata(pdev, pctl);
    
    dev_info(&pdev->dev, "Pinctrl driver registered\n");
    
    return 0;
}

static const struct of_device_id my_pinctrl_of_match[] = {
    { .compatible = "vendor,my-pinctrl", },
    { }
};
MODULE_DEVICE_TABLE(of, my_pinctrl_of_match);

static struct platform_driver my_pinctrl_driver = {
    .probe = my_pinctrl_probe,
    .driver = {
        .name = "my-pinctrl",
        .of_match_table = my_pinctrl_of_match,
    },
};
module_platform_driver(my_pinctrl_driver);
```

## Common Patterns

### Runtime Pin Switching

```c
// Switch between SPI and GPIO modes at runtime
struct device_data {
    struct pinctrl *pctrl;
    struct pinctrl_state *pins_spi;
    struct pinctrl_state *pins_gpio;
};

int switch_to_spi_mode(struct device_data *data)
{
    return pinctrl_select_state(data->pctrl, data->pins_spi);
}

int switch_to_gpio_mode(struct device_data *data)
{
    return pinctrl_select_state(data->pctrl, data->pins_gpio);
}
```

### GPIO Integration

```c
// Pinctrl controller can provide GPIO functionality
static int my_gpio_request(struct gpio_chip *chip, unsigned offset)
{
    struct my_pinctrl *pctl = gpiochip_get_data(chip);
    
    // Request pin from pinctrl
    return pinctrl_gpio_request(offset);
}

static void my_gpio_free(struct gpio_chip *chip, unsigned offset)
{
    pinctrl_gpio_free(offset);
}

static int my_gpio_direction_input(struct gpio_chip *chip, unsigned offset)
{
    // Set pin direction via pinctrl
    return pinctrl_gpio_direction_input(offset);
}

static int my_gpio_direction_output(struct gpio_chip *chip, unsigned offset,
                                     int value)
{
    return pinctrl_gpio_direction_output(offset);
}
```

## Debugging

### sysfs Interface

```bash
# List pinctrl devices
ls /sys/kernel/debug/pinctrl/

# View pinctrl configuration
cat /sys/kernel/debug/pinctrl/pinctrl-single/pins
cat /sys/kernel/debug/pinctrl/pinctrl-single/pinmux-pins
cat /sys/kernel/debug/pinctrl/pinctrl-single/pinconf-pins

# View pinctrl maps (device to pin mapping)
cat /sys/kernel/debug/pinctrl/pinctrl-maps

# View GPIO to pin mapping
cat /sys/kernel/debug/pinctrl/pinctrl-single/gpio-ranges
```

### Kernel Debugging

```c
// Enable dynamic debug
echo 'file drivers/pinctrl/* +p' > /sys/kernel/debug/dynamic_debug/control

// Kernel messages
dmesg | grep -i pinctrl
```

## Best Practices

✅ **DO:**
- Use `devm_pinctrl_get()` for automatic cleanup
- Let core framework manage default/sleep states automatically
- Define clear state names in device tree
- Use generic pinconf parameters when possible
- Check return values from all pinctrl operations

❌ **DON'T:**
- Manually manage pinctrl unless needed for custom states
- Forget to define default state in device tree
- Mix GPIO and pinmux on same pins without proper coordination
- Assume pinctrl is always available (check for -EPROBE_DEFER)
- Hardcode pin numbers (use device tree)

## Common Errors

```c
// Error: -EPROBE_DEFER
// Cause: Pinctrl provider not ready yet
// Solution: Driver will be probed again later

// Error: -EINVAL
// Cause: Invalid state name or configuration
// Solution: Check device tree pinctrl-names and pinctrl-N

// Error: Pin conflict
// Cause: Multiple devices requesting same pin
// Solution: Check device tree for duplicate pin assignments
```

## Pin Configuration Parameters

```c
// Generic pin config parameters (include/linux/pinctrl/pinconf-generic.h)

PIN_CONFIG_BIAS_DISABLE         // Disable pull-up/down
PIN_CONFIG_BIAS_HIGH_IMPEDANCE  // High-Z state
PIN_CONFIG_BIAS_BUS_HOLD        // Weak latch
PIN_CONFIG_BIAS_PULL_UP         // Pull-up enabled
PIN_CONFIG_BIAS_PULL_DOWN       // Pull-down enabled
PIN_CONFIG_BIAS_PULL_PIN_DEFAULT // Default pull state
PIN_CONFIG_DRIVE_PUSH_PULL      // Push-pull output
PIN_CONFIG_DRIVE_OPEN_DRAIN     // Open-drain output
PIN_CONFIG_DRIVE_OPEN_SOURCE    // Open-source output
PIN_CONFIG_DRIVE_STRENGTH       // Drive strength in mA
PIN_CONFIG_INPUT_ENABLE         // Enable input buffer
PIN_CONFIG_INPUT_SCHMITT_ENABLE // Schmitt trigger input
PIN_CONFIG_INPUT_DEBOUNCE       // Debounce time in usec
PIN_CONFIG_OUTPUT               // Set output value
PIN_CONFIG_OUTPUT_ENABLE        // Enable output buffer
PIN_CONFIG_SLEW_RATE            // Slew rate control
```
