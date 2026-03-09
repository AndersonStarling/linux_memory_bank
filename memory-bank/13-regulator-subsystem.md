# Regulator Framework

## Regulator Architecture Overview

Regulator Framework quản lý nguồn điện (voltage/current regulators) trong hệ thống, cho phép drivers bật/tắt nguồn, điều chỉnh voltage, và giám sát trạng thái nguồn.

```
┌──────────────────────────────────────────┐
│   Consumer (Drivers)                     │  <- regulator_get(), regulator_enable()
├──────────────────────────────────────────┤
│   Regulator Core                         │  <- Voltage/current management
├──────────────────────────────────────────┤
│   Regulator Driver (PMIC)               │  <- Hardware-specific implementation
└──────────────────────────────────────────┘
```

### Key Concepts
- **Regulator**: Thiết bị điều chỉnh điện áp/dòng điện
- **Consumer**: Driver sử dụng regulator
- **PMIC**: Power Management IC (chứa nhiều regulators)
- **Voltage/Current**: Điều chỉnh điện áp (V) và dòng điện (A)
- **Operating Modes**: FAST, NORMAL, IDLE, STANDBY

## Essential Headers

```c
// For consumers
#include <linux/regulator/consumer.h>

// For providers
#include <linux/regulator/driver.h>
#include <linux/regulator/machine.h>
```

## Regulator Consumer API

### Basic Operations

```c
#include <linux/regulator/consumer.h>

struct mydev_data {
    struct regulator *vdd;
    struct regulator *vcc;
};

static int mydev_probe(struct platform_device *pdev)
{
    struct mydev_data *data;
    int ret;
    
    data = devm_kzalloc(&pdev->dev, sizeof(*data), GFP_KERNEL);
    if (!data)
        return -ENOMEM;
    
    // Get regulator
    data->vdd = devm_regulator_get(&pdev->dev, "vdd");
    if (IS_ERR(data->vdd)) {
        ret = PTR_ERR(data->vdd);
        dev_err(&pdev->dev, "Failed to get vdd regulator: %d\n", ret);
        return ret;
    }
    
    // Optional regulator (won't fail if missing)
    data->vcc = devm_regulator_get_optional(&pdev->dev, "vcc");
    if (IS_ERR(data->vcc)) {
        if (PTR_ERR(data->vcc) == -EPROBE_DEFER)
            return -EPROBE_DEFER;
        data->vcc = NULL;  // Not critical, continue
    }
    
    // Set voltage range
    ret = regulator_set_voltage(data->vdd, 3300000, 3300000);  // 3.3V
    if (ret) {
        dev_err(&pdev->dev, "Failed to set voltage\n");
        return ret;
    }
    
    // Enable regulator
    ret = regulator_enable(data->vdd);
    if (ret) {
        dev_err(&pdev->dev, "Failed to enable regulator\n");
        return ret;
    }
    
    return 0;
}

static int mydev_remove(struct platform_device *pdev)
{
    struct mydev_data *data = platform_get_drvdata(pdev);
    
    // Disable regulator
    regulator_disable(data->vdd);
    
    return 0;
}
```

### Voltage Control

```c
// Set exact voltage (min == max)
ret = regulator_set_voltage(reg, 1800000, 1800000);  // 1.8V

// Set voltage range (regulator will choose optimal)
ret = regulator_set_voltage(reg, 1700000, 1900000);  // 1.7V - 1.9V

// Get current voltage
int uV = regulator_get_voltage(reg);
if (uV < 0)
    return uV;
dev_info(dev, "Voltage: %d.%03dV\n", uV / 1000000, (uV % 1000000) / 1000);

// List supported voltages
int count = regulator_count_voltages(reg);
for (i = 0; i < count; i++) {
    int vol = regulator_list_voltage(reg, i);
    if (vol > 0)
        pr_info("Selector %d: %duV\n", i, vol);
}

// Check if voltage is supported
if (regulator_is_supported_voltage(reg, 1800000, 1800000))
    pr_info("1.8V is supported\n");
```

### Current Limit

```c
// Set current limit
ret = regulator_set_current_limit(reg, 500000, 1000000);  // 0.5A - 1A
if (ret) {
    dev_err(dev, "Failed to set current limit\n");
    return ret;
}

// Get current limit
int limit = regulator_get_current_limit(reg);
dev_info(dev, "Current limit: %dmA\n", limit / 1000);
```

### Operating Modes

```c
// Set operating mode for efficiency
ret = regulator_set_mode(reg, REGULATOR_MODE_NORMAL);
ret = regulator_set_mode(reg, REGULATOR_MODE_IDLE);
ret = regulator_set_mode(reg, REGULATOR_MODE_STANDBY);

// Get current mode
unsigned int mode = regulator_get_mode(reg);
switch (mode) {
case REGULATOR_MODE_FAST:
    pr_info("Mode: FAST\n");
    break;
case REGULATOR_MODE_NORMAL:
    pr_info("Mode: NORMAL\n");
    break;
case REGULATOR_MODE_IDLE:
    pr_info("Mode: IDLE\n");
    break;
case REGULATOR_MODE_STANDBY:
    pr_info("Mode: STANDBY\n");
    break;
}

// Set load (regulator may auto-switch mode)
ret = regulator_set_load(reg, 100000);  // 100mA load
```

### Bulk Regulator Operations

```c
struct regulator_bulk_data supplies[] = {
    { .supply = "vdd" },
    { .supply = "vcc" },
    { .supply = "vdda" },
};

static int mydev_probe(struct platform_device *pdev)
{
    int ret;
    
    // Get all regulators
    ret = devm_regulator_bulk_get(&pdev->dev,
                                   ARRAY_SIZE(supplies),
                                   supplies);
    if (ret) {
        dev_err(&pdev->dev, "Failed to get regulators\n");
        return ret;
    }
    
    // Enable all
    ret = regulator_bulk_enable(ARRAY_SIZE(supplies), supplies);
    if (ret) {
        dev_err(&pdev->dev, "Failed to enable regulators\n");
        return ret;
    }
    
    return 0;
}

static void mydev_remove(struct platform_device *pdev)
{
    // Disable all
    regulator_bulk_disable(ARRAY_SIZE(supplies), supplies);
}
```

### Convenience Functions

```c
// Get and enable in one call
ret = devm_regulator_get_enable(&pdev->dev, "vdd");
if (ret)
    return ret;

// Get optional and enable
ret = devm_regulator_get_enable_optional(&pdev->dev, "vcc");

// Bulk get and enable
const char *supply_names[] = { "vdd", "vcc", "vdda" };
ret = devm_regulator_bulk_get_enable(&pdev->dev,
                                      ARRAY_SIZE(supply_names),
                                      supply_names);
```

## Device Tree Bindings

### Regulator Consumer

```dts
&mmc0 {
    vmmc-supply = <&vdd_mmc>;      // Main voltage supply
    vqmmc-supply = <&vdd_sd>;      // I/O voltage supply
};

&usb0 {
    vbus-supply = <&vdd_usb>;
};

&codec {
    AVDD-supply = <&reg_1v8>;      // Analog VDD
    DVDD-supply = <&reg_3v3>;      // Digital VDD
};
```

### Regulator Provider (PMIC)

```dts
pmic: tps65217@24 {
    compatible = "ti,tps65217";
    reg = <0x24>;
    
    regulators {
        dcdc1_reg: dcdc1 {
            regulator-name = "vdds_dpr";
            regulator-min-microvolt = <1100000>;
            regulator-max-microvolt = <1100000>;
            regulator-boot-on;
            regulator-always-on;
        };
        
        dcdc2_reg: dcdc2 {
            regulator-name = "vdd_mpu";
            regulator-min-microvolt = <925000>;
            regulator-max-microvolt = <1375000>;
            regulator-boot-on;
            regulator-always-on;
        };
        
        ldo1_reg: ldo1 {
            regulator-name = "vdd_rtc";
            regulator-min-microvolt = <1800000>;
            regulator-max-microvolt = <1800000>;
            regulator-always-on;
        };
        
        ldo3_reg: ldo3 {
            regulator-name = "vdd_3v3a";
            regulator-min-microvolt = <3300000>;
            regulator-max-microvolt = <3300000>;
        };
    };
};

// Fixed regulator (always on, no control)
reg_3v3: regulator-3v3 {
    compatible = "regulator-fixed";
    regulator-name = "3V3";
    regulator-min-microvolt = <3300000>;
    regulator-max-microvolt = <3300000>;
    regulator-always-on;
};

// GPIO-controlled regulator
reg_usb_vbus: regulator-usb-vbus {
    compatible = "regulator-fixed";
    regulator-name = "usb_vbus";
    regulator-min-microvolt = <5000000>;
    regulator-max-microvolt = <5000000>;
    gpio = <&gpio3 10 GPIO_ACTIVE_HIGH>;
    enable-active-high;
};
```

## Regulator Driver API

### Basic Regulator Driver

```c
#include <linux/regulator/driver.h>

struct my_regulator {
    struct regulator_dev *rdev;
    void __iomem *base;
    int enable_reg;
    int enable_mask;
    int vsel_reg;
    int vsel_mask;
};

static int my_reg_enable(struct regulator_dev *rdev)
{
    struct my_regulator *reg = rdev_get_drvdata(rdev);
    
    writel(readl(reg->base + reg->enable_reg) | reg->enable_mask,
           reg->base + reg->enable_reg);
    
    return 0;
}

static int my_reg_disable(struct regulator_dev *rdev)
{
    struct my_regulator *reg = rdev_get_drvdata(rdev);
    
    writel(readl(reg->base + reg->enable_reg) & ~reg->enable_mask,
           reg->base + reg->enable_reg);
    
    return 0;
}

static int my_reg_is_enabled(struct regulator_dev *rdev)
{
    struct my_regulator *reg = rdev_get_drvdata(rdev);
    
    return !!(readl(reg->base + reg->enable_reg) & reg->enable_mask);
}

static int my_reg_set_voltage_sel(struct regulator_dev *rdev,
                                   unsigned selector)
{
    struct my_regulator *reg = rdev_get_drvdata(rdev);
    u32 val;
    
    val = readl(reg->base + reg->vsel_reg);
    val &= ~reg->vsel_mask;
    val |= selector << ffs(reg->vsel_mask);
    writel(val, reg->base + reg->vsel_reg);
    
    return 0;
}

static int my_reg_get_voltage_sel(struct regulator_dev *rdev)
{
    struct my_regulator *reg = rdev_get_drvdata(rdev);
    u32 val;
    
    val = readl(reg->base + reg->vsel_reg);
    val &= reg->vsel_mask;
    val >>= ffs(reg->vsel_mask);
    
    return val;
}

static const struct regulator_ops my_reg_ops = {
    .enable = my_reg_enable,
    .disable = my_reg_disable,
    .is_enabled = my_reg_is_enabled,
    .set_voltage_sel = my_reg_set_voltage_sel,
    .get_voltage_sel = my_reg_get_voltage_sel,
    .list_voltage = regulator_list_voltage_linear,
};

static const struct regulator_desc my_reg_desc = {
    .name = "my-regulator",
    .id = 0,
    .ops = &my_reg_ops,
    .type = REGULATOR_VOLTAGE,
    .owner = THIS_MODULE,
    .min_uV = 1000000,       // 1.0V
    .uV_step = 100000,       // 100mV steps
    .n_voltages = 16,        // 16 voltage levels
};

static int my_reg_probe(struct platform_device *pdev)
{
    struct my_regulator *reg;
    struct regulator_config config = { };
    struct resource *res;
    
    reg = devm_kzalloc(&pdev->dev, sizeof(*reg), GFP_KERNEL);
    if (!reg)
        return -ENOMEM;
    
    res = platform_get_resource(pdev, IORESOURCE_MEM, 0);
    reg->base = devm_ioremap_resource(&pdev->dev, res);
    if (IS_ERR(reg->base))
        return PTR_ERR(reg->base);
    
    config.dev = &pdev->dev;
    config.driver_data = reg;
    config.of_node = pdev->dev.of_node;
    
    reg->rdev = devm_regulator_register(&pdev->dev, &my_reg_desc,
                                         &config);
    if (IS_ERR(reg->rdev))
        return PTR_ERR(reg->rdev);
    
    platform_set_drvdata(pdev, reg);
    
    return 0;
}
```

## Power Management

```c
#ifdef CONFIG_PM_SLEEP
static int mydev_suspend(struct device *dev)
{
    struct mydev_data *data = dev_get_drvdata(dev);
    
    // Regulators typically managed automatically
    // Manual control only if needed
    regulator_disable(data->vdd);
    
    return 0;
}

static int mydev_resume(struct device *dev)
{
    struct mydev_data *data = dev_get_drvdata(dev);
    
    return regulator_enable(data->vdd);
}
#endif
```

## Debugging

```bash
# View regulators
ls /sys/class/regulator/

# View regulator details
cat /sys/class/regulator/regulator.0/name
cat /sys/class/regulator/regulator.0/microvolts
cat /sys/class/regulator/regulator.0/state
cat /sys/class/regulator/regulator.0/num_users

# Kernel messages
dmesg | grep -i regulator
```

## Best Practices

✅ **DO:**
- Use `devm_regulator_get()` for cleanup
- Check voltage range support before setting
- Use bulk operations for multiple regulators
- Enable regulators before using hardware
- Disable when not needed to save power

❌ **DON'T:**
- Assume regulator is always available
- Forget to handle -EPROBE_DEFER
- Set voltage while regulator disabled
- Ignore return values

## Common Patterns

### Voltage Scaling

```c
int scale_voltage(struct regulator *reg, unsigned long new_freq)
{
    int uV;
    
    if (new_freq > 800000000)
        uV = 1350000;  // 1.35V for high freq
    else if (new_freq > 400000000)
        uV = 1200000;  // 1.2V for medium
    else
        uV = 1100000;  // 1.1V for low freq
    
    return regulator_set_voltage(reg, uV, uV);
}
```
