# PWM Subsystem

## PWM Architecture Overview

PWM (Pulse Width Modulation) subsystem trong Linux kernel cung cấp giao diện chuẩn hóa để điều khiển PWM signals. PWM thường dùng cho điều khiển LED, quạt, motor, hoặc backlight.

```
┌─────────────────────────────────────┐
│   Consumer API (Drivers)            │  <- pwm_get(), pwm_apply_might_sleep()
├─────────────────────────────────────┤
│   PWM Core (pwm/core.c)              │  <- struct pwm_chip, pwm_device
├─────────────────────────────────────┤
│   PWM Controller Driver              │  <- Hardware-specific ops
└─────────────────────────────────────┘
```

### Key Concepts
- **PWM Chip**: Represents a PWM controller (can provide multiple PWM channels)
- **PWM Device**: Individual PWM channel/output
- **PWM State**: Configuration (period, duty_cycle, polarity, enabled)
- **Period**: Total cycle time (in nanoseconds)
- **Duty Cycle**: Active (high) time within the period (in nanoseconds)
- **Polarity**: Normal (high first) or Inversed (low first)

## Essential Headers

```c
// For PWM consumers (drivers using PWM)
#include <linux/pwm.h>

// For PWM chip drivers (implementing PWM controller)
// Already included in <linux/pwm.h>
```

## PWM Consumer API (For Drivers Using PWM)

### PWM State Structure

```c
struct pwm_state {
    u64 period;              // PWM period in nanoseconds
    u64 duty_cycle;          // PWM duty cycle in nanoseconds
    enum pwm_polarity polarity;  // PWM_POLARITY_NORMAL or INVERSED
    bool enabled;            // PWM enabled status
    bool usage_power;        // Power optimization hint
};

enum pwm_polarity {
    PWM_POLARITY_NORMAL,     // High for duty_cycle, low for remainder
    PWM_POLARITY_INVERSED,   // Low for duty_cycle, high for remainder
};
```

### Requesting PWM Device

```c
#include <linux/pwm.h>

// Get PWM device (basic)
struct pwm_device *pwm;
pwm = pwm_get(&pdev->dev, NULL);
if (IS_ERR(pwm))
    return PTR_ERR(pwm);

// Get PWM device with name (when multiple PWMs)
pwm = pwm_get(&pdev->dev, "backlight");

// Managed variants (auto cleanup)
pwm = devm_pwm_get(&pdev->dev, NULL);
if (IS_ERR(pwm))
    return PTR_ERR(pwm);

// Get from device tree/firmware node
pwm = devm_fwnode_pwm_get(&pdev->dev, fwnode, NULL);
```

### Configuring PWM

```c
// Modern API - Apply complete state (atomic update)
struct pwm_state state;

// Get current state
pwm_get_state(pwm, &state);

// Modify state
state.period = 1000000;          // 1ms period (1kHz)
state.duty_cycle = 500000;       // 50% duty cycle
state.polarity = PWM_POLARITY_NORMAL;
state.enabled = true;

// Apply state (may sleep)
ret = pwm_apply_might_sleep(pwm, &state);
if (ret)
    return ret;

// For atomic context (check first!)
if (!pwm_might_sleep(pwm)) {
    ret = pwm_apply_atomic(pwm, &state);
}
```

### Helper Functions

```c
// Check if PWM operations might sleep
bool can_sleep = pwm_might_sleep(pwm);

// Get current state
struct pwm_state state;
pwm_get_state(pwm, &state);

// Check if enabled
if (pwm_is_enabled(pwm))
    pr_info("PWM is running\n");

// Get period
u64 period = pwm_get_period(pwm);

// Get duty cycle
u64 duty = pwm_get_duty_cycle(pwm);

// Get polarity
enum pwm_polarity pol = pwm_get_polarity(pwm);
```

### Legacy API (Avoid in New Code)

```c
// These functions are wrappers around pwm_apply_might_sleep()
// Prefer using pwm_apply_might_sleep() directly

// Configure period and duty cycle
pwm_config(pwm, duty_ns, period_ns);

// Enable PWM
pwm_enable(pwm);

// Disable PWM
pwm_disable(pwm);

// Set polarity
pwm_set_polarity(pwm, PWM_POLARITY_INVERSED);
```

### Releasing PWM

```c
// Manual release
pwm_put(pwm);

// With devm_pwm_get(), automatic cleanup
// No explicit release needed!
```

## Device Tree Bindings

### PWM Controller Node

```dts
// PWM controller
ehrpwm0: pwm@48300200 {
    compatible = "ti,am33xx-ehrpwm";
    reg = <0x48300200 0x80>;
    #pwm-cells = <3>;
    clocks = <&ehrpwm0_tbclk>;
    clock-names = "fck";
};

// Generic PWM controller
pwm: pwm@12345000 {
    compatible = "vendor,pwm";
    reg = <0x12345000 0x1000>;
    #pwm-cells = <2>;
    clocks = <&pwm_clk>;
};
```

### PWM Consumer Node

```dts
#include <dt-bindings/pwm/pwm.h>

// Single PWM usage
backlight {
    compatible = "pwm-backlight";
    pwms = <&ehrpwm0 0 50000 PWM_POLARITY_INVERTED>;
    //       ^        ^ ^     ^
    //       |        | |     +-- Polarity flag
    //       |        | +-------- Period (50us = 20kHz)
    //       |        +---------- PWM channel number
    //       +------------------- PWM controller phandle
    
    brightness-levels = <0 4 8 16 32 64 128 255>;
    default-brightness-level = <6>;
};

// Multiple PWMs
device {
    compatible = "vendor,device";
    pwms = <&pwm 0 1000000 0>,      // Channel 0: 1ms, normal polarity
           <&pwm 1 2000000 PWM_POLARITY_INVERTED>;  // Channel 1: 2ms, inverted
    pwm-names = "motor", "led";
};
```

### PWM Flags

```c
#include <dt-bindings/pwm/pwm.h>

PWM_POLARITY_INVERTED    // Invert the PWM signal polarity
// (0 means normal polarity in device tree)
```

### Parsing in Driver

```c
// PWM name comes from device tree pwm-names property
struct pwm_device *pwm;

// Get single PWM (first in pwms list)
pwm = devm_pwm_get(&pdev->dev, NULL);

// Get named PWM
pwm = devm_pwm_get(&pdev->dev, "backlight");

// If driver needs initial period/polarity from DT
struct pwm_args args;
pwm_get_args(pwm, &args);
// args.period and args.polarity contain DT values
```

## PWM Chip Driver API (For Implementing Controllers)

### PWM Operations Structure

```c
struct pwm_ops {
    // Apply PWM state (preferred method)
    int (*apply)(struct pwm_chip *chip, struct pwm_device *pwm,
                 const struct pwm_state *state);
    
    // Get current hardware state
    int (*get_state)(struct pwm_chip *chip, struct pwm_device *pwm,
                     struct pwm_state *state);
    
    // Legacy methods (deprecated, use apply instead)
    int (*config)(struct pwm_chip *chip, struct pwm_device *pwm,
                  int duty_ns, int period_ns);
    int (*enable)(struct pwm_chip *chip, struct pwm_device *pwm);
    void (*disable)(struct pwm_chip *chip, struct pwm_device *pwm);
    int (*set_polarity)(struct pwm_chip *chip, struct pwm_device *pwm,
                        enum pwm_polarity polarity);
    
    // Optional: Request/free PWM
    int (*request)(struct pwm_chip *chip, struct pwm_device *pwm);
    void (*free)(struct pwm_chip *chip, struct pwm_device *pwm);
    
    // Module owner
    struct module *owner;
};
```

### Implementing a PWM Chip Driver

```c
#include <linux/pwm.h>
#include <linux/platform_device.h>
#include <linux/io.h>
#include <linux/clk.h>

struct my_pwm_chip {
    void __iomem *base;
    struct clk *clk;
    unsigned long clk_rate;
};

static inline struct my_pwm_chip *to_my_pwm_chip(struct pwm_chip *chip)
{
    return pwmchip_get_drvdata(chip);
}

static int my_pwm_apply(struct pwm_chip *chip, struct pwm_device *pwm,
                        const struct pwm_state *state)
{
    struct my_pwm_chip *mpc = to_my_pwm_chip(chip);
    u64 period_cycles, duty_cycles;
    u32 val;
    
    // Calculate cycles from nanoseconds
    period_cycles = state->period * mpc->clk_rate;
    do_div(period_cycles, NSEC_PER_SEC);
    
    duty_cycles = state->duty_cycle * mpc->clk_rate;
    do_div(duty_cycles, NSEC_PER_SEC);
    
    // Program hardware registers
    writel(period_cycles, mpc->base + PERIOD_REG);
    writel(duty_cycles, mpc->base + DUTY_REG);
    
    // Configure polarity
    val = readl(mpc->base + CTRL_REG);
    if (state->polarity == PWM_POLARITY_INVERSED)
        val |= CTRL_POLARITY_BIT;
    else
        val &= ~CTRL_POLARITY_BIT;
    
    // Enable/disable
    if (state->enabled)
        val |= CTRL_ENABLE_BIT;
    else
        val &= ~CTRL_ENABLE_BIT;
    
    writel(val, mpc->base + CTRL_REG);
    
    return 0;
}

static int my_pwm_get_state(struct pwm_chip *chip, struct pwm_device *pwm,
                            struct pwm_state *state)
{
    struct my_pwm_chip *mpc = to_my_pwm_chip(chip);
    u64 period_cycles, duty_cycles;
    u32 val;
    
    // Read hardware state
    period_cycles = readl(mpc->base + PERIOD_REG);
    duty_cycles = readl(mpc->base + DUTY_REG);
    val = readl(mpc->base + CTRL_REG);
    
    // Convert to nanoseconds
    state->period = period_cycles * NSEC_PER_SEC;
    do_div(state->period, mpc->clk_rate);
    
    state->duty_cycle = duty_cycles * NSEC_PER_SEC;
    do_div(state->duty_cycle, mpc->clk_rate);
    
    state->polarity = (val & CTRL_POLARITY_BIT) ?
                      PWM_POLARITY_INVERSED : PWM_POLARITY_NORMAL;
    state->enabled = !!(val & CTRL_ENABLE_BIT);
    
    return 0;
}

static const struct pwm_ops my_pwm_ops = {
    .apply = my_pwm_apply,
    .get_state = my_pwm_get_state,
    .owner = THIS_MODULE,
};

static int my_pwm_probe(struct platform_device *pdev)
{
    struct pwm_chip *chip;
    struct my_pwm_chip *mpc;
    struct resource *res;
    int ret;
    
    // Allocate PWM chip with driver data
    chip = pwmchip_alloc(&pdev->dev, 4, sizeof(*mpc));
    if (IS_ERR(chip))
        return PTR_ERR(chip);
    
    mpc = to_my_pwm_chip(chip);
    
    // Map registers
    res = platform_get_resource(pdev, IORESOURCE_MEM, 0);
    mpc->base = devm_ioremap_resource(&pdev->dev, res);
    if (IS_ERR(mpc->base)) {
        ret = PTR_ERR(mpc->base);
        goto err_free_chip;
    }
    
    // Get clock
    mpc->clk = devm_clk_get(&pdev->dev, "pwm");
    if (IS_ERR(mpc->clk)) {
        ret = PTR_ERR(mpc->clk);
        goto err_free_chip;
    }
    
    ret = clk_prepare_enable(mpc->clk);
    if (ret)
        goto err_free_chip;
    
    mpc->clk_rate = clk_get_rate(mpc->clk);
    
    // Setup PWM chip
    chip->ops = &my_pwm_ops;
    chip->atomic = false;  // Set true if operations don't sleep
    
    // Register PWM chip
    ret = pwmchip_add(chip);
    if (ret < 0) {
        dev_err(&pdev->dev, "Failed to add PWM chip: %d\n", ret);
        goto err_disable_clk;
    }
    
    platform_set_drvdata(pdev, chip);
    
    dev_info(&pdev->dev, "PWM chip registered with %u channels\n",
             chip->npwm);
    
    return 0;

err_disable_clk:
    clk_disable_unprepare(mpc->clk);
err_free_chip:
    pwmchip_put(chip);
    return ret;
}

static void my_pwm_remove(struct platform_device *pdev)
{
    struct pwm_chip *chip = platform_get_drvdata(pdev);
    struct my_pwm_chip *mpc = to_my_pwm_chip(chip);
    
    pwmchip_remove(chip);
    clk_disable_unprepare(mpc->clk);
    pwmchip_put(chip);
}

static const struct of_device_id my_pwm_of_match[] = {
    { .compatible = "vendor,my-pwm", },
    { }
};
MODULE_DEVICE_TABLE(of, my_pwm_of_match);

static struct platform_driver my_pwm_driver = {
    .probe = my_pwm_probe,
    .remove = my_pwm_remove,
    .driver = {
        .name = "my-pwm",
        .of_match_table = my_pwm_of_match,
    },
};
module_platform_driver(my_pwm_driver);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Your Name");
MODULE_DESCRIPTION("My PWM driver");
```

## Common PWM Use Cases

### LED Brightness Control

```c
struct pwm_device *pwm;
struct pwm_state state;

pwm = devm_pwm_get(&pdev->dev, NULL);
if (IS_ERR(pwm))
    return PTR_ERR(pwm);

// Initialize state
pwm_init_state(pwm, &state);
state.period = 1000000;  // 1ms (1kHz)
state.duty_cycle = 0;
state.enabled = true;

// Apply initial state
pwm_apply_might_sleep(pwm, &state);

// Set brightness (0-255)
void set_brightness(u8 brightness)
{
    pwm_get_state(pwm, &state);
    state.duty_cycle = (state.period * brightness) / 255;
    pwm_apply_might_sleep(pwm, &state);
}
```

### Motor Speed Control

```c
// Set motor speed (0-100%)
void set_motor_speed(unsigned int speed_percent)
{
    struct pwm_state state;
    
    if (speed_percent > 100)
        speed_percent = 100;
    
    pwm_get_state(motor_pwm, &state);
    state.duty_cycle = (state.period * speed_percent) / 100;
    state.enabled = (speed_percent > 0);
    pwm_apply_might_sleep(motor_pwm, &state);
}
```

### Servo Control

```c
// Servo typically: 20ms period, 1-2ms pulse
void set_servo_angle(int angle_degrees)
{
    struct pwm_state state;
    u64 pulse_ns;
    
    // Angle: -90 to +90 degrees
    // Pulse: 1ms (-90°) to 2ms (+90°)
    pulse_ns = 1500000 + (angle_degrees * 500000 / 90);
    
    pwm_get_state(servo_pwm, &state);
    state.period = 20000000;      // 20ms
    state.duty_cycle = pulse_ns;
    state.polarity = PWM_POLARITY_NORMAL;
    state.enabled = true;
    
    pwm_apply_might_sleep(servo_pwm, &state);
}
```

## sysfs Interface

### Exporting PWM

```bash
# Navigate to PWM chip
cd /sys/class/pwm/pwmchip0

# See number of channels
cat npwm

# Export PWM channel 0
echo 0 > export

# Now PWM0 directory appears
cd pwm0

# Configure period (nanoseconds)
echo 1000000 > period  # 1ms = 1kHz

# Configure duty cycle (nanoseconds)
echo 500000 > duty_cycle  # 50%

# Set polarity
echo "normal" > polarity
echo "inversed" > polarity

# Enable PWM
echo 1 > enable

# Disable PWM
echo 0 > enable

# Unexport when done
cd ..
echo 0 > unexport
```

## Calculating PWM Parameters

### Frequency to Period

```c
// Frequency (Hz) to Period (ns)
// period_ns = 10^9 / frequency_hz

u64 freq_to_period_ns(u32 freq_hz)
{
    return NSEC_PER_SEC / freq_hz;
}

// Example: 1kHz = 1,000,000 ns period
u64 period = freq_to_period_ns(1000);  // 1,000,000 ns
```

### Duty Cycle Percentage

```c
// Calculate duty cycle from percentage
u64 duty_cycle_from_percent(u64 period_ns, unsigned int percent)
{
    if (percent > 100)
        percent = 100;
    
    return (period_ns * percent) / 100;
}

// Example: 50% of 1ms period = 500us
u64 duty = duty_cycle_from_percent(1000000, 50);  // 500,000 ns
```

## Power Management

```c
#ifdef CONFIG_PM_SLEEP
static int my_pwm_suspend(struct device *dev)
{
    struct pwm_chip *chip = dev_get_drvdata(dev);
    struct my_pwm_chip *mpc = to_my_pwm_chip(chip);
    
    // Save PWM state if needed
    // Hardware state is preserved in pwm->state by core
    
    clk_disable_unprepare(mpc->clk);
    
    return 0;
}

static int my_pwm_resume(struct device *dev)
{
    struct pwm_chip *chip = dev_get_drvdata(dev);
    struct my_pwm_chip *mpc = to_my_pwm_chip(chip);
    int ret;
    
    ret = clk_prepare_enable(mpc->clk);
    if (ret)
        return ret;
    
    // PWM state will be reapplied by consumers
    // Driver doesn't need to restore state
    
    return 0;
}

static SIMPLE_DEV_PM_OPS(my_pwm_pm_ops, my_pwm_suspend, my_pwm_resume);

static struct platform_driver my_pwm_driver = {
    .driver = {
        .name = "my-pwm",
        .pm = &my_pwm_pm_ops,
        .of_match_table = my_pwm_of_match,
    },
};
#endif
```

## Best Practices

✅ **DO:**
- Use `pwm_apply_might_sleep()` for atomic state updates
- Implement `.apply()` and `.get_state()` ops in new drivers
- Check `pwm_might_sleep()` before calling from atomic context
- Use `devm_pwm_get()` for automatic cleanup
- Document PWM polarity in device tree bindings
- Let consumers handle PWM state across suspend/resume

❌ **DON'T:**
- Use legacy `pwm_config()`, `pwm_enable()`, `pwm_disable()` separately
- Call `pwm_apply_might_sleep()` from atomic context
- Assume PWM is disabled when duty_cycle = 0 (use enabled flag)
- Forget to enable clock before accessing PWM registers
- Implement power management in PWM driver (consumers handle it)

## Common Patterns

### Gradual Brightness Fade

```c
void fade_brightness(struct pwm_device *pwm, u8 from, u8 to, int steps, int delay_ms)
{
    struct pwm_state state;
    int i, step_size;
    
    pwm_get_state(pwm, &state);
    step_size = (to - from) / steps;
    
    for (i = 0; i < steps; i++) {
        u8 brightness = from + (step_size * i);
        state.duty_cycle = (state.period * brightness) / 255;
        pwm_apply_might_sleep(pwm, &state);
        msleep(delay_ms);
    }
}
```

### PWM Frequency Measurement

```c
// In driver with .get_state() support
void measure_pwm_frequency(struct pwm_device *pwm)
{
    struct pwm_state state;
    u64 freq_hz;
    
    pwm_get_state(pwm, &state);
    
    if (state.period == 0) {
        pr_info("PWM disabled\n");
        return;
    }
    
    freq_hz = NSEC_PER_SEC;
    do_div(freq_hz, state.period);
    
    pr_info("PWM frequency: %llu Hz\n", freq_hz);
    pr_info("Duty cycle: %llu%%\n", 
            (state.duty_cycle * 100) / state.period);
}
```

## Debugging

```bash
# List all PWM chips
ls -la /sys/class/pwm/

# Check PWM chip details
cat /sys/class/pwm/pwmchip0/npwm

# Check exported PWMs
ls -la /sys/class/pwm/pwmchip0/

# View PWM configuration
cat /sys/class/pwm/pwmchip0/pwm0/period
cat /sys/class/pwm/pwmchip0/pwm0/duty_cycle
cat /sys/class/pwm/pwmchip0/pwm0/polarity
cat /sys/class/pwm/pwmchip0/pwm0/enable

# Kernel messages
dmesg | grep -i pwm
```
