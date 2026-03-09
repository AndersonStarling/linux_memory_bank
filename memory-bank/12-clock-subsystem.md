# Clock Framework

## Clock Architecture Overview

Clock Framework quản lý tất cả clock sources trong hệ thống, cho phép drivers enable/disable clocks, điều chỉnh tần số, và quản lý power. Mọi peripheral đều cần clock để hoạt động.

```
┌─────────────────────────────────────────┐
│   Consumer (Drivers)                    │  <- clk_get(), clk_prepare_enable()
├─────────────────────────────────────────┤
│   Clock Core (clk.c)                     │  <- Clock tree management
├─────────────────────────────────────────┤
│   Clock Provider (clk-provider)         │  <- Hardware-specific implementation
└─────────────────────────────────────────┘
```

### Key Concepts
- **Clock**: Nguồn tín hiệu clock (oscillator, PLL, divider, gate)
- **Clock Tree**: Hierarchy của clocks (parent-child relationships)
- **Clock Rate**: Tần số (Hz)
- **Clock Enable/Disable**: Bật/tắt clock để tiết kiệm năng lượng
- **Prepare/Unprepare**: Non-atomic preparation (có thể sleep)
- **Enable/Disable**: Atomic enable (không sleep, có thể gọi từ IRQ)

## Essential Headers

```c
// For clock consumers
#include <linux/clk.h>

// For clock providers
#include <linux/clk-provider.h>
```

## Clock Consumer API

### Basic Clock Operations

```c
#include <linux/clk.h>

struct mydev_data {
    struct clk *clk;
    unsigned long clk_rate;
};

static int mydev_probe(struct platform_device *pdev)
{
    struct mydev_data *data;
    int ret;
    
    data = devm_kzalloc(&pdev->dev, sizeof(*data), GFP_KERNEL);
    if (!data)
        return -ENOMEM;
    
    // Get clock
    data->clk = devm_clk_get(&pdev->dev, "fck");
    if (IS_ERR(data->clk)) {
        ret = PTR_ERR(data->clk);
        dev_err(&pdev->dev, "Failed to get clock: %d\n", ret);
        return ret;
    }
    
    // Get clock rate
    data->clk_rate = clk_get_rate(data->clk);
    dev_info(&pdev->dev, "Clock rate: %lu Hz\n", data->clk_rate);
    
    // Prepare and enable clock (atomic + non-atomic)
    ret = clk_prepare_enable(data->clk);
    if (ret) {
        dev_err(&pdev->dev, "Failed to enable clock: %d\n", ret);
        return ret;
    }
    
    return 0;
}

static int mydev_remove(struct platform_device *pdev)
{
    struct mydev_data *data = platform_get_drvdata(pdev);
    
    // Disable and unprepare clock
    clk_disable_unprepare(data->clk);
    
    return 0;
}
```

### Clock Rate Management

```c
// Set clock rate
unsigned long rate = 48000000;  // 48 MHz
ret = clk_set_rate(clk, rate);
if (ret) {
    dev_err(dev, "Failed to set rate to %lu\n", rate);
    return ret;
}

// Get actual rate (may differ from requested)
unsigned long actual_rate = clk_get_rate(clk);
dev_info(dev, "Requested: %lu Hz, Actual: %lu Hz\n", rate, actual_rate);

// Round rate (find closest supported rate)
long rounded_rate = clk_round_rate(clk, rate);
if (rounded_rate < 0)
    return rounded_rate;

// Set parent clock
struct clk *parent = devm_clk_get(dev, "parent");
ret = clk_set_parent(clk, parent);
```

### Prepare vs Enable

```c
// IMPORTANT: Understand the difference!

// clk_prepare() - May sleep, can call from process context
ret = clk_prepare(clk);

// clk_enable() - Atomic, can call from IRQ context
ret = clk_enable(clk);

// clk_prepare_enable() - Convenience function (prepare + enable)
ret = clk_prepare_enable(clk);

// Reverse order for disable
clk_disable(clk);         // Atomic
clk_unprepare(clk);       // May sleep

// Or combined
clk_disable_unprepare(clk);
```

### Bulk Clock Operations

```c
#include <linux/clk.h>

struct clk_bulk_data clks[] = {
    { .id = "fck" },
    { .id = "ick" },
    { .id = "sys_clk" },
};

static int mydev_probe(struct platform_device *pdev)
{
    int ret;
    
    // Get all clocks
    ret = devm_clk_bulk_get(&pdev->dev, ARRAY_SIZE(clks), clks);
    if (ret) {
        dev_err(&pdev->dev, "Failed to get clocks\n");
        return ret;
    }
    
    // Enable all clocks
    ret = clk_bulk_prepare_enable(ARRAY_SIZE(clks), clks);
    if (ret) {
        dev_err(&pdev->dev, "Failed to enable clocks\n");
        return ret;
    }
    
    return 0;
}

static void mydev_remove(struct platform_device *pdev)
{
    // Disable all clocks
    clk_bulk_disable_unprepare(ARRAY_SIZE(clks), clks);
}
```

## Device Tree Bindings

### Clock Consumer

```dts
&uart0 {
    clocks = <&clk_48mhz>;
    clock-names = "fck";
};

&mmc0 {
    clocks = <&clk_96mhz>, <&clk_32khz>;
    clock-names = "fck", "dbclk";
};

&i2c0 {
    clocks = <&dpll_per_m2_ck>;
    clock-names = "fck";
};
```

### Clock Provider

```dts
// Fixed rate clock
clk_32khz: clk_32k {
    compatible = "fixed-clock";
    #clock-cells = <0>;
    clock-frequency = <32768>;
};

// Divider clock
clk_div: clock-divider {
    compatible = "divider-clock";
    #clock-cells = <0>;
    clocks = <&parent_clk>;
    reg = <0x44e00000 0x4>;
    bit-shift = <8>;
    bit-width = <3>;
};

// Gate clock
clk_gate: clock-gate {
    compatible = "gate-clock";
    #clock-cells = <0>;
    clocks = <&parent_clk>;
    reg = <0x44e00004 0x4>;
    bit-shift = <0>;
};
```

## Clock Provider API

### Basic Clock Provider

```c
#include <linux/clk-provider.h>

// Fixed rate clock
struct clk *clk_fixed;
clk_fixed = clk_register_fixed_rate(NULL, "fixed_clk", NULL,
                                     0, 48000000);

// Gate clock
struct clk *clk_gate;
clk_gate = clk_register_gate(NULL, "gate_clk", "parent_clk",
                              0, base + 0x10, 0, 0, NULL);

// Divider clock
struct clk *clk_div;
clk_div = clk_register_divider(NULL, "div_clk", "parent_clk",
                                0, base + 0x20, 8, 3, 0, NULL);

// Mux clock
const char *parents[] = { "clk0", "clk1", "clk2" };
struct clk *clk_mux;
clk_mux = clk_register_mux(NULL, "mux_clk", parents,
                            ARRAY_SIZE(parents), 0,
                            base + 0x30, 0, 2, 0, NULL);
```

### Custom Clock Implementation

```c
struct my_clk {
    struct clk_hw hw;
    void __iomem *reg;
    spinlock_t *lock;
};

static unsigned long my_clk_recalc_rate(struct clk_hw *hw,
                                         unsigned long parent_rate)
{
    struct my_clk *clk = container_of(hw, struct my_clk, hw);
    u32 div;
    
    div = readl(clk->reg) & 0xFF;
    if (div == 0)
        div = 1;
    
    return parent_rate / div;
}

static long my_clk_round_rate(struct clk_hw *hw, unsigned long rate,
                               unsigned long *parent_rate)
{
    unsigned long div = DIV_ROUND_UP(*parent_rate, rate);
    
    if (div > 256)
        div = 256;
    if (div < 1)
        div = 1;
    
    return *parent_rate / div;
}

static int my_clk_set_rate(struct clk_hw *hw, unsigned long rate,
                            unsigned long parent_rate)
{
    struct my_clk *clk = container_of(hw, struct my_clk, hw);
    unsigned long div = parent_rate / rate;
    unsigned long flags;
    u32 val;
    
    if (div > 256)
        return -EINVAL;
    
    spin_lock_irqsave(clk->lock, flags);
    val = readl(clk->reg);
    val &= ~0xFF;
    val |= (div - 1) & 0xFF;
    writel(val, clk->reg);
    spin_unlock_irqrestore(clk->lock, flags);
    
    return 0;
}

static int my_clk_enable(struct clk_hw *hw)
{
    struct my_clk *clk = container_of(hw, struct my_clk, hw);
    unsigned long flags;
    u32 val;
    
    spin_lock_irqsave(clk->lock, flags);
    val = readl(clk->reg);
    val |= BIT(31);  // Enable bit
    writel(val, clk->reg);
    spin_unlock_irqrestore(clk->lock, flags);
    
    return 0;
}

static void my_clk_disable(struct clk_hw *hw)
{
    struct my_clk *clk = container_of(hw, struct my_clk, hw);
    unsigned long flags;
    u32 val;
    
    spin_lock_irqsave(clk->lock, flags);
    val = readl(clk->reg);
    val &= ~BIT(31);
    writel(val, clk->reg);
    spin_unlock_irqrestore(clk->lock, flags);
}

static const struct clk_ops my_clk_ops = {
    .recalc_rate = my_clk_recalc_rate,
    .round_rate = my_clk_round_rate,
    .set_rate = my_clk_set_rate,
    .enable = my_clk_enable,
    .disable = my_clk_disable,
};

static int my_clk_probe(struct platform_device *pdev)
{
    struct my_clk *clk;
    struct clk_init_data init;
    struct resource *res;
    int ret;
    
    clk = devm_kzalloc(&pdev->dev, sizeof(*clk), GFP_KERNEL);
    if (!clk)
        return -ENOMEM;
    
    res = platform_get_resource(pdev, IORESOURCE_MEM, 0);
    clk->reg = devm_ioremap_resource(&pdev->dev, res);
    if (IS_ERR(clk->reg))
        return PTR_ERR(clk->reg);
    
    init.name = "my_clk";
    init.ops = &my_clk_ops;
    init.parent_names = NULL;
    init.num_parents = 0;
    init.flags = 0;
    
    clk->hw.init = &init;
    
    ret = devm_clk_hw_register(&pdev->dev, &clk->hw);
    if (ret)
        return ret;
    
    return of_clk_add_hw_provider(pdev->dev.of_node,
                                   of_clk_hw_simple_get, &clk->hw);
}
```

## Clock Notifiers

```c
// Monitor clock rate changes
static int my_clk_notifier_cb(struct notifier_block *nb,
                               unsigned long event, void *data)
{
    struct clk_notifier_data *ndata = data;
    
    switch (event) {
    case PRE_RATE_CHANGE:
        pr_info("Clock rate changing from %lu to %lu\n",
                ndata->old_rate, ndata->new_rate);
        // Prepare for rate change
        break;
    
    case POST_RATE_CHANGE:
        pr_info("Clock rate changed to %lu\n", ndata->new_rate);
        // Update after rate change
        break;
    
    case ABORT_RATE_CHANGE:
        pr_info("Clock rate change aborted\n");
        // Restore state
        break;
    }
    
    return NOTIFY_OK;
}

static struct notifier_block my_clk_nb = {
    .notifier_call = my_clk_notifier_cb,
};

// Register notifier
clk_notifier_register(clk, &my_clk_nb);

// Unregister when done
clk_notifier_unregister(clk, &my_clk_nb);
```

## Power Management

```c
#ifdef CONFIG_PM_SLEEP
static int mydev_suspend(struct device *dev)
{
    struct mydev_data *data = dev_get_drvdata(dev);
    
    // Disable clocks during suspend
    clk_disable_unprepare(data->clk);
    
    return 0;
}

static int mydev_resume(struct device *dev)
{
    struct mydev_data *data = dev_get_drvdata(dev);
    
    // Re-enable clocks on resume
    return clk_prepare_enable(data->clk);
}

static SIMPLE_DEV_PM_OPS(mydev_pm_ops, mydev_suspend, mydev_resume);
#endif
```

## Debugging

```bash
# View clock tree
cat /sys/kernel/debug/clk/clk_summary

# View specific clock
cat /sys/kernel/debug/clk/uart0_fck/clk_rate
cat /sys/kernel/debug/clk/uart0_fck/clk_enable_count
cat /sys/kernel/debug/clk/uart0_fck/clk_parent

# Kernel messages
dmesg | grep -i clock
```

## Best Practices

✅ **DO:**
- Always use `devm_clk_get()` for automatic cleanup
- Call `clk_prepare_enable()` before using hardware
- Check return values from all clock operations
- Use `clk_disable_unprepare()` when done
- Use bulk operations for multiple clocks
- Understand prepare vs enable semantics

❌ **DON'T:**
- Call `clk_enable()` from atomic context without `clk_prepare()`
- Forget to disable clocks (wastes power)
- Assume exact rate (use `clk_get_rate()` to verify)
- Change rate while clock is in use by hardware
- Mix atomic and non-atomic operations incorrectly

## Common Patterns

### Runtime PM with Clocks

```c
#ifdef CONFIG_PM
static int mydev_runtime_suspend(struct device *dev)
{
    struct mydev_data *data = dev_get_drvdata(dev);
    
    clk_disable_unprepare(data->clk);
    return 0;
}

static int mydev_runtime_resume(struct device *dev)
{
    struct mydev_data *data = dev_get_drvdata(dev);
    
    return clk_prepare_enable(data->clk);
}

static const struct dev_pm_ops mydev_pm_ops = {
    SET_RUNTIME_PM_OPS(mydev_runtime_suspend,
                       mydev_runtime_resume, NULL)
};
#endif
```

### Clock Gating for Power Savings

```c
void mydev_idle(struct mydev_data *data)
{
    // Disable clock when idle
    clk_disable_unprepare(data->clk);
}

int mydev_activate(struct mydev_data *data)
{
    // Enable clock when active
    return clk_prepare_enable(data->clk);
}
```
