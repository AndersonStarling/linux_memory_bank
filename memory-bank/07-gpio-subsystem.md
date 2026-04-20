# GPIO Subsystem

## GPIO Architecture Overview

The GPIO subsystem trong Linux kernel cung cấp giao diện chuẩn hóa để điều khiển General Purpose Input/Output pins. Kiến trúc gồm 3 layers chính:

```
┌─────────────────────────────────────┐
│   Consumer API (Drivers)            │  <- gpiod_get(), gpiod_set_value()
├─────────────────────────────────────┤
│   GPIO Core (gpiolib)                │  <- gpio_chip, gpio_device
├─────────────────────────────────────┤
│   GPIO Controller Driver             │  <- Hardware-specific implementation
└─────────────────────────────────────┘
```

### Key Concepts
- **GPIO Chip**: Represents a GPIO controller (typically one hardware block)
- **GPIO Line**: Individual GPIO pin (identified by offset 0..ngpio-1)
- **GPIO Descriptor**: Opaque handle to a GPIO line (`struct gpio_desc`)
- **Legacy vs Descriptor API**: Old integer-based vs new descriptor-based API

## Essential Headers

```c
// For GPIO consumers (drivers using GPIO)
#include <linux/gpio/consumer.h>

// For GPIO chip drivers (implementing GPIO controller)
#include <linux/gpio/driver.h>

// Legacy API (DEPRECATED - don't use in new code)
#include <linux/gpio.h>
```

## GPIO Consumer API (For Drivers Using GPIO)

### Requesting GPIO Lines

```c
#include <linux/gpio/consumer.h>

// Get single GPIO
struct gpio_desc *desc;
desc = devm_gpiod_get(&pdev->dev, "reset", GPIOD_OUT_HIGH);
if (IS_ERR(desc))
    return PTR_ERR(desc);

// Get GPIO by index
desc = devm_gpiod_get_index(&pdev->dev, "data", 0, GPIOD_IN);

// Get optional GPIO (won't fail if missing)
desc = devm_gpiod_get_optional(&pdev->dev, "enable", GPIOD_OUT_LOW);

// Get array of GPIOs
struct gpio_descs *descs;
descs = devm_gpiod_get_array(&pdev->dev, "addr", GPIOD_OUT_LOW);
```

### GPIO Direction Flags

```c
enum gpiod_flags {
    GPIOD_ASIS          = 0,        // Don't change direction
    GPIOD_IN            = ...,      // Configure as input
    GPIOD_OUT_LOW       = ...,      // Output driving low
    GPIOD_OUT_HIGH      = ...,      // Output driving high
    GPIOD_OUT_LOW_OPEN_DRAIN  = ..., // Open-drain output low
    GPIOD_OUT_HIGH_OPEN_DRAIN = ..., // Open-drain output high
};
```

### Setting/Getting GPIO Values

```c
// Set output value (non-sleeping context)
gpiod_set_value(desc, 1);        // Drive high
gpiod_set_value(desc, 0);        // Drive low

// Get input value (non-sleeping context)
int value = gpiod_get_value(desc);

// Sleeping variants (can be used with I2C/SPI GPIO expanders)
gpiod_set_value_cansleep(desc, 1);
int value = gpiod_get_value_cansleep(desc);

// Array operations
unsigned long values_bitmap = 0b1010;  // Binary pattern
gpiod_set_array_value(4, desc_array, NULL, &values_bitmap);
```

### Changing Direction at Runtime

```c
// Change to input
int ret = gpiod_direction_input(desc);

// Change to output with initial value
ret = gpiod_direction_output(desc, 1);  // Output high

// Check current direction
int dir = gpiod_get_direction(desc);
if (dir == 0)
    pr_info("GPIO is output\n");
else
    pr_info("GPIO is input\n");
```

### GPIO to IRQ Conversion

```c
// Get IRQ number for GPIO line
int irq = gpiod_to_irq(desc);
if (irq < 0)
    return irq;

// Request IRQ
ret = devm_request_irq(&pdev->dev, irq, my_irq_handler,
                       IRQF_TRIGGER_RISING, "my-gpio-irq", priv);
```

### Releasing GPIOs

```c
// Manual release (rarely needed)
gpiod_put(desc);
gpiod_put_array(descs);

// With devm_*, automatic cleanup on driver detach
// No explicit release needed!
```

## Device Tree Bindings

### In Device Tree Source (.dts)

```dts
// GPIO controller node
gpio1: gpio@48310000 {
    compatible = "ti,omap4-gpio";
    reg = <0x48310000 0x200>;
    interrupts = <GIC_SPI 29 IRQ_TYPE_LEVEL_HIGH>;
    gpio-controller;
    #gpio-cells = <2>;
    interrupt-controller;
    #interrupt-cells = <2>;
};

// Consumer device using GPIO
mydevice@0 {
    compatible = "vendor,mydevice";
    
    // Single GPIO (name-gpios = <&controller pin flags>)
    reset-gpios = <&gpio1 5 GPIO_ACTIVE_LOW>;
    enable-gpios = <&gpio1 12 GPIO_ACTIVE_HIGH>;
    
    // Multiple GPIOs for same function
    data-gpios = <&gpio1 0 0>,
                 <&gpio1 1 0>,
                 <&gpio1 2 0>,
                 <&gpio1 3 0>;
    
    // GPIO with interrupt
    interrupt-parent = <&gpio1>;
    interrupts = <7 IRQ_TYPE_EDGE_RISING>;
};
```

### GPIO Flags in Device Tree

```c
#include <dt-bindings/gpio/gpio.h>

GPIO_ACTIVE_HIGH    (0)    // Active high (default)
GPIO_ACTIVE_LOW     (1)    // Active low (inverted)
GPIO_OPEN_DRAIN     (2)    // Open-drain output
GPIO_OPEN_SOURCE    (4)    // Open-source output
GPIO_PULL_UP        (8)    // Enable pull-up resistor
GPIO_PULL_DOWN      (16)   // Enable pull-down resistor
```

### Parsing in Driver

```c
// GPIO name comes from device tree property name
// "reset-gpios" in DT -> "reset" in driver
struct gpio_desc *reset_gpio;
reset_gpio = devm_gpiod_get(&pdev->dev, "reset", GPIOD_OUT_HIGH);

// Get by index for arrays
desc = devm_gpiod_get_index(&pdev->dev, "data", 2, GPIOD_OUT_LOW);
```

## GPIO Chip Driver API (For Implementing Controllers)

### Implementing a GPIO Chip

```c
#include <linux/gpio/driver.h>

struct my_gpio_chip {
    struct gpio_chip chip;
    void __iomem *base;
    spinlock_t lock;
    // ... hardware-specific fields
};

static int my_gpio_get(struct gpio_chip *gc, unsigned offset)
{
    struct my_gpio_chip *mgc = gpiochip_get_data(gc);
    return !!(readl(mgc->base + DATA_REG) & BIT(offset));
}

static void my_gpio_set(struct gpio_chip *gc, unsigned offset, int value)
{
    struct my_gpio_chip *mgc = gpiochip_get_data(gc);
    unsigned long flags;
    u32 val;
    
    spin_lock_irqsave(&mgc->lock, flags);
    val = readl(mgc->base + DATA_REG);
    if (value)
        val |= BIT(offset);
    else
        val &= ~BIT(offset);
    writel(val, mgc->base + DATA_REG);
    spin_unlock_irqrestore(&mgc->lock, flags);
}

static int my_gpio_direction_input(struct gpio_chip *gc, unsigned offset)
{
    struct my_gpio_chip *mgc = gpiochip_get_data(gc);
    unsigned long flags;
    u32 val;
    
    spin_lock_irqsave(&mgc->lock, flags);
    val = readl(mgc->base + DIR_REG);
    val |= BIT(offset);  // 1 = input
    writel(val, mgc->base + DIR_REG);
    spin_unlock_irqrestore(&mgc->lock, flags);
    
    return 0;
}

static int my_gpio_direction_output(struct gpio_chip *gc, unsigned offset,
                                    int value)
{
    struct my_gpio_chip *mgc = gpiochip_get_data(gc);
    unsigned long flags;
    u32 val;
    
    spin_lock_irqsave(&mgc->lock, flags);
    
    // Set output value first
    val = readl(mgc->base + DATA_REG);
    if (value)
        val |= BIT(offset);
    else
        val &= ~BIT(offset);
    writel(val, mgc->base + DATA_REG);
    
    // Then set direction
    val = readl(mgc->base + DIR_REG);
    val &= ~BIT(offset);  // 0 = output
    writel(val, mgc->base + DIR_REG);
    
    spin_unlock_irqrestore(&mgc->lock, flags);
    
    return 0;
}

static int my_gpio_probe(struct platform_device *pdev)
{
    struct my_gpio_chip *mgc;
    struct resource *res;
    int ret;
    
    mgc = devm_kzalloc(&pdev->dev, sizeof(*mgc), GFP_KERNEL);
    if (!mgc)
        return -ENOMEM;
    
    res = platform_get_resource(pdev, IORESOURCE_MEM, 0);
    mgc->base = devm_ioremap_resource(&pdev->dev, res);
    if (IS_ERR(mgc->base))
        return PTR_ERR(mgc->base);
    
    spin_lock_init(&mgc->lock);
    
    // Setup gpio_chip structure
    mgc->chip.label = "my-gpio";
    mgc->chip.parent = &pdev->dev;
    mgc->chip.owner = THIS_MODULE;
    mgc->chip.base = -1;  // Dynamic allocation
    mgc->chip.ngpio = 32;  // Number of GPIO lines
    
    // Required callbacks
    mgc->chip.get = my_gpio_get;
    mgc->chip.set = my_gpio_set;
    mgc->chip.direction_input = my_gpio_direction_input;
    mgc->chip.direction_output = my_gpio_direction_output;
    
    // Optional callbacks
    mgc->chip.get_direction = my_gpio_get_direction;
    mgc->chip.set_config = my_gpio_set_config;
    mgc->chip.to_irq = my_gpio_to_irq;
    
    // Register the GPIO chip
    ret = devm_gpiochip_add_data(&pdev->dev, &mgc->chip, mgc);
    if (ret)
        return dev_err_probe(&pdev->dev, ret, "Failed to add GPIO chip\n");
    
    platform_set_drvdata(pdev, mgc);
    
    dev_info(&pdev->dev, "GPIO chip registered with %d lines\n",
             mgc->chip.ngpio);
    
    return 0;
}

static const struct of_device_id my_gpio_of_match[] = {
    { .compatible = "vendor,my-gpio", },
    { }
};
MODULE_DEVICE_TABLE(of, my_gpio_of_match);

static struct platform_driver my_gpio_driver = {
    .probe = my_gpio_probe,
    .driver = {
        .name = "my-gpio",
        .of_match_table = my_gpio_of_match,
    },
};
module_platform_driver(my_gpio_driver);
```

### GPIO IRQ Chip Integration

```c
static int my_gpio_to_irq(struct gpio_chip *gc, unsigned offset)
{
    struct my_gpio_chip *mgc = gpiochip_get_data(gc);
    return irq_create_mapping(mgc->irq_domain, offset);
}

static void my_gpio_irq_mask(struct irq_data *d)
{
    struct gpio_chip *gc = irq_data_get_irq_chip_data(d);
    struct my_gpio_chip *mgc = gpiochip_get_data(gc);
    unsigned long flags;
    
    spin_lock_irqsave(&mgc->lock, flags);
    // Clear interrupt enable bit
    writel(readl(mgc->base + IRQ_EN) & ~BIT(d->hwirq),
           mgc->base + IRQ_EN);
    spin_unlock_irqrestore(&mgc->lock, flags);
}

static void my_gpio_irq_unmask(struct irq_data *d)
{
    struct gpio_chip *gc = irq_data_get_irq_chip_data(d);
    struct my_gpio_chip *mgc = gpiochip_get_data(gc);
    unsigned long flags;
    
    spin_lock_irqsave(&mgc->lock, flags);
    // Set interrupt enable bit
    writel(readl(mgc->base + IRQ_EN) | BIT(d->hwirq),
           mgc->base + IRQ_EN);
    spin_unlock_irqrestore(&mgc->lock, flags);
}

static int my_gpio_irq_set_type(struct irq_data *d, unsigned int type)
{
    struct gpio_chip *gc = irq_data_get_irq_chip_data(d);
    struct my_gpio_chip *mgc = gpiochip_get_data(gc);
    unsigned long flags;
    u32 rising, falling;
    
    spin_lock_irqsave(&mgc->lock, flags);
    
    rising = readl(mgc->base + IRQ_RISING);
    falling = readl(mgc->base + IRQ_FALLING);
    
    rising &= ~BIT(d->hwirq);
    falling &= ~BIT(d->hwirq);
    
    if (type & IRQ_TYPE_EDGE_RISING)
        rising |= BIT(d->hwirq);
    if (type & IRQ_TYPE_EDGE_FALLING)
        falling |= BIT(d->hwirq);
    
    writel(rising, mgc->base + IRQ_RISING);
    writel(falling, mgc->base + IRQ_FALLING);
    
    spin_unlock_irqrestore(&mgc->lock, flags);
    
    return 0;
}

static struct irq_chip my_gpio_irq_chip = {
    .name = "my-gpio-irq",
    .irq_mask = my_gpio_irq_mask,
    .irq_unmask = my_gpio_irq_unmask,
    .irq_set_type = my_gpio_irq_set_type,
    .flags = IRQCHIP_IMMUTABLE,
    GPIOCHIP_IRQ_RESOURCE_HELPERS,
};

// In probe function, setup IRQ
static int my_gpio_probe(struct platform_device *pdev)
{
    struct gpio_irq_chip *girq;
    
    // ... previous setup ...
    
    girq = &mgc->chip.irq;
    gpio_irq_chip_set_chip(girq, &my_gpio_irq_chip);
    girq->parent_handler = my_gpio_irq_handler;
    girq->num_parents = 1;
    girq->parents = devm_kcalloc(&pdev->dev, 1,
                                 sizeof(*girq->parents),
                                 GFP_KERNEL);
    if (!girq->parents)
        return -ENOMEM;
    
    girq->parents[0] = platform_get_irq(pdev, 0);
    girq->default_type = IRQ_TYPE_NONE;
    girq->handler = handle_edge_irq;
    
    return devm_gpiochip_add_data(&pdev->dev, &mgc->chip, mgc);
}
```

## Common GPIO Patterns

### Toggling GPIO

```c
int val = gpiod_get_value(desc);
gpiod_set_value(desc, !val);
```

### Bit-banging Protocol

```c
void spi_bitbang_write(struct gpio_desc *clk, struct gpio_desc *mosi,
                       u8 data)
{
    int i;
    
    for (i = 7; i >= 0; i--) {
        gpiod_set_value(clk, 0);
        gpiod_set_value(mosi, (data >> i) & 1);
        udelay(1);
        gpiod_set_value(clk, 1);
        udelay(1);
    }
}
```

### Using GPIO as Interrupt

```c
static irqreturn_t my_gpio_irq_handler(int irq, void *dev_id)
{
    struct my_device *priv = dev_id;
    
    // Read GPIO value
    int val = gpiod_get_value(priv->irq_gpio);
    dev_info(priv->dev, "GPIO interrupt! Value: %d\n", val);
    
    return IRQ_HANDLED;
}

static int my_probe(struct platform_device *pdev)
{
    struct gpio_desc *irq_gpio;
    int irq, ret;
    
    irq_gpio = devm_gpiod_get(&pdev->dev, "interrupt", GPIOD_IN);
    if (IS_ERR(irq_gpio))
        return PTR_ERR(irq_gpio);
    
    irq = gpiod_to_irq(irq_gpio);
    if (irq < 0)
        return irq;
    
    ret = devm_request_irq(&pdev->dev, irq, my_gpio_irq_handler,
                           IRQF_TRIGGER_RISING | IRQF_TRIGGER_FALLING,
                           "my-gpio-irq", priv);
    return ret;
}
```

## Debugging GPIO

### sysfs Interface (Legacy)

```bash
# List all GPIO chips
ls /sys/class/gpio/

# Export GPIO for userspace access
echo 23 > /sys/class/gpio/export

# Set direction
echo out > /sys/class/gpio/gpio23/direction
echo in > /sys/class/gpio/gpio23/direction

# Set/get value
echo 1 > /sys/class/gpio/gpio23/value
cat /sys/class/gpio/gpio23/value

# Unexport
echo 23 > /sys/class/gpio/unexport
```

### debugfs Interface

```bash
# View all GPIO lines and their state
cat /sys/kernel/debug/gpio

# Example output:
# gpiochip0: GPIOs 0-31, parent: platform/48310000.gpio, omap_gpio:
#  gpio-5   (reset              ) out hi
#  gpio-12  (enable             ) out lo
#  gpio-7   (                   ) in  hi IRQ
```

### libgpiod Tools (Modern Approach)

```bash
# List GPIO chips
gpiodetect

# Get chip info
gpioinfo gpiochip0

# Set GPIO value
gpioset gpiochip0 5=1

# Get GPIO value
gpioget gpiochip0 5

# Monitor GPIO events
gpiomon --rising-edge --falling-edge gpiochip0 7
```

## Character Device Layer (gpiolib-cdev.c)

### Architecture

The character device layer provides userspace access via `/dev/gpiochipX`:

```
Userspace (libgpiod)
    ↓ open("/dev/gpiochip0")
    ↓ ioctl(GPIO_V2_GET_LINE_IOCTL, ...)
gpiolib-cdev.c
    ├─ gpio_chrdev_open()
    ├─ gpio_ioctl()              // Main ioctl handler
    ├─ lineinfo_get()            // Get line info
    ├─ lineevent_create()        // Event monitoring
    └─ linehandle_create()       // Line request
    ↓
gpiolib.c (core framework)
    ↓
Hardware driver (gpio-omap.c, etc.)
```

### Key Structures

```c
// File operations for /dev/gpiochipX
static const struct file_operations gpio_fileops = {
    .owner = THIS_MODULE,
    .open = gpio_chrdev_open,
    .release = gpio_chrdev_release,
    .unlocked_ioctl = gpio_ioctl,
    .compat_ioctl = gpio_ioctl_compat,
    .read = lineinfo_watch_read,
    .poll = lineinfo_watch_poll,
};
```

### IOCTL Commands

```c
// GPIO v2 API (current)
#define GPIO_V2_GET_LINEINFO_IOCTL       // Get line information
#define GPIO_V2_GET_LINE_IOCTL           // Request lines
#define GPIO_V2_LINE_SET_CONFIG_IOCTL    // Configure lines
#define GPIO_V2_LINE_GET_VALUES_IOCTL    // Read values
#define GPIO_V2_LINE_SET_VALUES_IOCTL    // Write values
```

## Userspace GPIO Access - Simple Methods

### Overview: Three Approaches

| Method | Complexity | Code Required | Use Case |
|--------|-----------|---------------|----------|
| **sysfs** | ⭐ Simplest | Shell commands only | Quick test, shell scripts |
| **Command line tools** | ⭐⭐ Simple | Command invocation | Automation, scripting |
| **libgpiod API** | ⭐⭐⭐⭐ Complex | C/Python code | Application integration |

**Recommendation for driver development**: Use **sysfs** or **command line tools** for testing - focus on kernel code!

---

## Method 1: Sysfs Interface (Simplest) ⭐

**Zero code required** - pure shell commands:

```bash
# Step 1: Export GPIO (request the line)
echo 60 > /sys/class/gpio/export

# Step 2: Set direction
echo in > /sys/class/gpio/gpio60/direction
# or: echo out > /sys/class/gpio/gpio60/direction

# Step 3: Read value
cat /sys/class/gpio/gpio60/value
# Output: 0 or 1

# Step 4: Write value (if output)
echo 1 > /sys/class/gpio/gpio60/value
echo 0 > /sys/class/gpio/gpio60/value

# Step 5: Cleanup when done
echo 60 > /sys/class/gpio/unexport
```

### Shell Script Example

```bash
#!/bin/bash
# Blink LED on GPIO 60

GPIO=60

# Export
echo $GPIO > /sys/class/gpio/export
sleep 0.1  # Wait for kernel to create sysfs files

# Set as output
echo out > /sys/class/gpio/gpio$GPIO/direction

# Blink 10 times
for i in {1..10}; do
    echo 1 > /sys/class/gpio/gpio$GPIO/value
    sleep 0.5
    echo 0 > /sys/class/gpio/gpio$GPIO/value
    sleep 0.5
done

# Cleanup
echo $GPIO > /sys/class/gpio/unexport
```

### Python Example

```python
#!/usr/bin/env python3
import time

GPIO = 60

# Export
with open('/sys/class/gpio/export', 'w') as f:
    f.write(str(GPIO))
time.sleep(0.1)

# Set direction
with open(f'/sys/class/gpio/gpio{GPIO}/direction', 'w') as f:
    f.write('out')

# Blink
for _ in range(10):
    with open(f'/sys/class/gpio/gpio{GPIO}/value', 'w') as f:
        f.write('1')
    time.sleep(0.5)
    with open(f'/sys/class/gpio/gpio{GPIO}/value', 'w') as f:
        f.write('0')
    time.sleep(0.5)

# Cleanup
with open('/sys/class/gpio/unexport', 'w') as f:
    f.write(str(GPIO))
```

### Advanced Sysfs Features

```bash
# Active-low logic (invert)
echo 1 > /sys/class/gpio/gpio60/active_low
# Now: writing 1 drives pin LOW, writing 0 drives HIGH

# Read current configuration
cat /sys/class/gpio/gpio60/direction    # in/out
cat /sys/class/gpio/gpio60/active_low   # 0/1
cat /sys/class/gpio/gpio60/value        # 0/1

# Set output with direction command
echo low > /sys/class/gpio/gpio60/direction   # Set output, drive low
echo high > /sys/class/gpio/gpio60/direction  # Set output, drive high

# Edge detection for interrupts
echo rising > /sys/class/gpio/gpio60/edge
echo falling > /sys/class/gpio/gpio60/edge
echo both > /sys/class/gpio/gpio60/edge
echo none > /sys/class/gpio/gpio60/edge

# Monitor with inotify (blocking)
while inotifywait -e modify /sys/class/gpio/gpio60/value 2>/dev/null; do
    VALUE=$(cat /sys/class/gpio/gpio60/value)
    echo "GPIO changed to: $VALUE"
done
```

**⚠️ Note**: Sysfs GPIO interface deprecated since kernel 4.8, but still available and works well for testing/prototyping.

---

## Method 2: Command Line Tools ⭐⭐

Part of `libgpiod` package, but used as standalone tools:

```bash
# Install
sudo apt install gpiod

# Read GPIO value
gpioget gpiochip0 60
# Output: 0 or 1

# Set GPIO value
gpioset gpiochip0 60=1
gpioset gpiochip0 60=0

# Set and hold (doesn't exit immediately)
gpioset --mode=wait gpiochip0 60=1
# Press Ctrl+C to release

# Set with timeout
gpioset --mode=time --sec=5 gpiochip0 60=1
# Holds for 5 seconds, then releases

# Monitor GPIO events
gpiomon gpiochip0 60
# event: RISING EDGE offset: 60 timestamp: [1234567890.123456789]
# event: FALLING EDGE offset: 60 timestamp: [1234567890.987654321]

# Monitor specific edges
gpiomon --rising-edge gpiochip0 60
gpiomon --falling-edge gpiochip0 60

# Get GPIO line info
gpioinfo gpiochip0 | grep "line  60"
# line  60:      unnamed       unused   input  active-high

# List all GPIO chips
gpiodetect
# gpiochip0 [gpio-0-31] (32 lines)
# gpiochip1 [gpio-32-63] (32 lines)
```

### Shell Script with gpioset

```bash
#!/bin/bash
# Blink LED - Much simpler than sysfs!

for i in {1..10}; do
    gpioset gpiochip0 60=1
    sleep 0.5
    gpioset gpiochip0 60=0
    sleep 0.5
done
```

### Button Monitoring with gpiomon

```bash
#!/bin/bash
# Monitor button presses

echo "Press button (Ctrl+C to quit)..."
gpiomon --rising-edge --falling-edge gpiochip0 48 | while read event; do
    echo "Button event: $event"
    # Parse and react to event
done
```

### Multiple GPIOs

```bash
# Read multiple GPIOs at once
gpioget gpiochip0 60 61 62
# 0 1 1

# Set multiple GPIOs
gpioset gpiochip0 60=1 61=0 62=1
```

---

## Method 3: libgpiod API (For App Integration)

### When to Use

Use libgpiod C/Python API when:
- ⚠️ Need to integrate GPIO into C/C++/Python application
- ⚠️ Need event notification within app logic
- ⚠️ Need to control many GPIOs with complex timing
- ⚠️ Building production-grade application

For **kernel driver testing**: Use sysfs or command line tools instead!

### C API

**Installation:**
```bash
sudo apt install libgpiod-dev gpiod
```

**Basic read/write:**
```c
#include <gpiod.h>
#include <stdio.h>

int main(void)
{
    struct gpiod_chip *chip;
    struct gpiod_line *line;
    int value, ret;
    
    // Open GPIO chip
    chip = gpiod_chip_open("/dev/gpiochip0");
    if (!chip) {
        perror("gpiod_chip_open");
        return 1;
    }
    
    // Get line (GPIO offset 5)
    line = gpiod_chip_get_line(chip, 5);
    if (!line) {
        perror("gpiod_chip_get_line");
        gpiod_chip_close(chip);
        return 1;
    }
    
    // Request as input
    ret = gpiod_line_request_input(line, "myapp");
    if (ret < 0) {
        perror("gpiod_line_request_input");
        gpiod_chip_close(chip);
        return 1;
    }
    
    // Read value
    value = gpiod_line_get_value(line);
    printf("GPIO value: %d\n", value);
    
    // Cleanup
    gpiod_line_release(line);
    gpiod_chip_close(chip);
    
    return 0;
}
```

**Compile:**
```bash
gcc -o read_gpio read_gpio.c -lgpiod
```

**Output control:**
```c
struct gpiod_chip *chip;
struct gpiod_line *line;

chip = gpiod_chip_open("/dev/gpiochip0");
line = gpiod_chip_get_line(chip, 5);

// Request as output with initial value
gpiod_line_request_output(line, "myapp", 1);  // Start high

// Toggle
gpiod_line_set_value(line, 0);
sleep(1);
gpiod_line_set_value(line, 1);

gpiod_line_release(line);
gpiod_chip_close(chip);
```

**Event monitoring:**
```c
struct gpiod_line_event event;
struct timespec timeout = { 1, 0 };  // 1 second

gpiod_line_request_rising_edge_events(line, "myapp");

while (1) {
    ret = gpiod_line_event_wait(line, &timeout);
    if (ret > 0) {
        gpiod_line_event_read(line, &event);
        printf("Event: type=%d at %lld.%09ld\n",
               event.event_type,
               event.ts.tv_sec,
               event.ts.tv_nsec);
    }
}
```

### Python API

**Installation:**
```bash
pip install gpiod
```

**Basic usage:**
```python
import gpiod
import time

# Open chip
chip = gpiod.Chip('gpiochip0')

# Get line
line = chip.get_line(5)

# Request as input
line.request(consumer='myapp', type=gpiod.LINE_REQ_DIR_IN)
value = line.get_value()
print(f"GPIO value: {value}")
line.release()

# Request as output
line.request(consumer='myapp', type=gpiod.LINE_REQ_DIR_OUT, default_vals=[1])

# Toggle
for i in range(5):
    line.set_value(1)
    time.sleep(0.5)
    line.set_value(0)
    time.sleep(0.5)

line.release()
chip.close()
```

## GPIO Line Ownership & Direction Management

### Exclusive Ownership Model

**Critical concept**: GPIO lines in the character device interface use **exclusive ownership** - only ONE consumer can request a line at any time.

```
┌─────────────────────────────────────────────────────┐
│ GPIO Line State Machine                             │
├─────────────────────────────────────────────────────┤
│                                                     │
│  [FREE] ──request()──> [CLAIMED] ──release()──> [FREE]
│                            │                        │
│                            │                        │
│                         request()                   │
│                            │                        │
│                            ↓                        │
│                        -EBUSY                       │
└─────────────────────────────────────────────────────┘
```

### Kernel Implementation

**gpiolib-cdev.c - Request validation:**

```c
// When requesting a line via ioctl(GPIO_V2_GET_LINE_IOCTL)
static int linereq_create(struct gpio_device *gdev, void __user *ip)
{
    // ... setup code ...
    
    for (i = 0; i < ulr.num_lines; i++) {
        desc = gpio_device_get_desc(gdev, offset);
        
        // CHECK: Is line already requested?
        if (test_bit(FLAG_REQUESTED, &desc->flags)) {
            ret = -EBUSY;  // Line claimed by kernel driver or another process
            goto out_free_linereq;
        }
        
        // ... proceed with request ...
    }
}
```

### Who Can Own a GPIO Line?

| Owner Type | Example | Detection |
|-----------|---------|-----------|
| **Kernel driver** | LED driver, gpio-keys | `cat /sys/kernel/debug/gpio` |
| **libgpiod process** | `gpioset`, your app | `lsof /dev/gpiochipX` |
| **sysfs export** | `echo N > /sys/class/gpio/export` | `ls /sys/class/gpio/gpioN/` |

**Example - Line already in use:**

```bash
# Terminal 1: Hold line 5 high
$ gpioset gpiochip0 5=1
# (keeps running, holds exclusive lock)

# Terminal 2: Try to read same line
$ gpioget gpiochip0 5
gpioget: error waiting for events on line 5 of gpiochip0: Device or resource busy

# Kernel log
$ dmesg | tail
[  123.456] gpio gpiochip0: line 5 is busy
```

### Reading GPIO Lines - Direction Considerations

#### ❌ Problem: Original libgpiod Example

The minimal examples often show:

```c
// This ASSUMES line is FREE and UNUSED!
gpiod_line_settings_set_direction(settings, GPIOD_LINE_DIRECTION_INPUT);
request = gpiod_chip_request_lines(chip, req_cfg, line_cfg);
```

**Issue**: If line is already OUTPUT (e.g., controlling motor, LED), this:
1. **Will FAIL** if line already claimed → `-EBUSY`
2. **Could disrupt** hardware if it succeeded (changing OUTPUT→INPUT)

#### ✅ Solution 1: Request with Correct Direction

If you **own** the line and know it's OUTPUT, you can still read:

```c
#include <gpiod.h>

// Request as OUTPUT
gpiod_line_settings_set_direction(settings, GPIOD_LINE_DIRECTION_OUTPUT);
gpiod_line_settings_set_output_value(settings, GPIOD_LINE_VALUE_ACTIVE);

request = gpiod_chip_request_lines(chip, req_cfg, line_cfg);

// Read current OUTPUT value (reads back from hardware register)
value = gpiod_line_request_get_value(request, offset);
printf("Output line driving: %d\n", value);
```

**Hardware behavior**: GPIO controllers can read DATAOUT register even in OUTPUT mode.

#### ✅ Solution 2: Use AS_IS Direction (libgpiod v2+)

Don't change current direction:

```c
// Request line keeping its current direction
gpiod_line_settings_set_direction(settings, GPIOD_LINE_DIRECTION_AS_IS);

request = gpiod_chip_request_lines(chip, req_cfg, line_cfg);

// Works for both INPUT and OUTPUT lines
value = gpiod_line_request_get_value(request, offset);
```

**Limitation**: Line must still be **FREE** (not claimed by anyone).

#### ✅ Solution 3: Monitor via sysfs/debugfs

If line is **already claimed** by kernel driver:

```bash
# Method 1: debugfs (requires root)
$ sudo cat /sys/kernel/debug/gpio
gpiochip0: GPIOs 0-31, parent: platform/44e07000.gpio:
 gpio-5  ( motor-enable         ) out hi    # Already claimed, value=high
 gpio-12 ( sysfs                ) in  lo    # Exported via sysfs

# Method 2: sysfs (deprecated in kernel 4.8+, but still works)
# Note: Can't export if already claimed by driver
$ echo 5 > /sys/class/gpio/export       # FAILS if claimed
bash: echo: write error: Device or resource busy

# But can read if already exported
$ cat /sys/class/gpio/gpio5/direction   # out
$ cat /sys/class/gpio/gpio5/value       # 1
```

### Complete Example: Safe GPIO Read

```c
// SPDX-License-Identifier: GPL-2.0-or-later
#include <errno.h>
#include <gpiod.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

/* 
 * Request line preserving current direction
 * Returns NULL if line is busy or doesn't exist
 */
static struct gpiod_line_request *
request_line_safe(const char *chip_path, unsigned int offset,
                  const char *consumer, enum gpiod_line_direction direction)
{
	struct gpiod_request_config *req_cfg = NULL;
	struct gpiod_line_request *request = NULL;
	struct gpiod_line_settings *settings;
	struct gpiod_line_config *line_cfg;
	struct gpiod_chip *chip;
	int ret;

	chip = gpiod_chip_open(chip_path);
	if (!chip) {
		fprintf(stderr, "Failed to open %s: %s\n",
		        chip_path, strerror(errno));
		return NULL;
	}

	settings = gpiod_line_settings_new();
	if (!settings)
		goto close_chip;

	gpiod_line_settings_set_direction(settings, direction);

	line_cfg = gpiod_line_config_new();
	if (!line_cfg)
		goto free_settings;

	ret = gpiod_line_config_add_line_settings(line_cfg, &offset, 1,
	                                          settings);
	if (ret)
		goto free_line_config;

	if (consumer) {
		req_cfg = gpiod_request_config_new();
		if (!req_cfg)
			goto free_line_config;
		gpiod_request_config_set_consumer(req_cfg, consumer);
	}

	request = gpiod_chip_request_lines(chip, req_cfg, line_cfg);
	
	if (!request) {
		if (errno == EBUSY) {
			fprintf(stderr,
			        "Line %u is already in use (kernel driver or another process)\n",
			        offset);
		} else {
			fprintf(stderr, "Failed to request line %u: %s\n",
			        offset, strerror(errno));
		}
	}

	gpiod_request_config_free(req_cfg);
free_line_config:
	gpiod_line_config_free(line_cfg);
free_settings:
	gpiod_line_settings_free(settings);
close_chip:
	gpiod_chip_close(chip);

	return request;
}

int main(int argc, char *argv[])
{
	const char *chip_path = "/dev/gpiochip0";
	unsigned int line_offset;
	struct gpiod_line_request *request;
	enum gpiod_line_value value;

	if (argc < 2) {
		fprintf(stderr, "Usage: %s <line_offset>\n", argv[0]);
		return EXIT_FAILURE;
	}

	line_offset = atoi(argv[1]);

	/* Try to request with AS_IS (doesn't change direction) */
	request = request_line_safe(chip_path, line_offset, "safe-read",
	                             GPIOD_LINE_DIRECTION_AS_IS);
	if (!request)
		return EXIT_FAILURE;

	value = gpiod_line_request_get_value(request, line_offset);
	
	if (value == GPIOD_LINE_VALUE_ACTIVE)
		printf("GPIO %u = ACTIVE (HIGH)\n", line_offset);
	else if (value == GPIOD_LINE_VALUE_INACTIVE)
		printf("GPIO %u = INACTIVE (LOW)\n", line_offset);
	else
		fprintf(stderr, "Error reading value: %s\n", strerror(errno));

	gpiod_line_request_release(request);
	return EXIT_SUCCESS;
}
```

**Compile and test:**
```bash
gcc -o safe_read safe_read.c -lgpiod
./safe_read 5

# If line 5 already in use:
# "Line 5 is already in use (kernel driver or another process)"

# If line 5 is free:
# "GPIO 5 = ACTIVE (HIGH)"
```

### GPIO Direction Flags in Kernel

```c
// include/linux/gpio/driver.h

/* Line flags */
#define FLAG_REQUESTED    0  /* Line is requested */
#define FLAG_IS_OUT       1  /* Line is output */
#define FLAG_EXPORT       2  /* Line exported via sysfs */
#define FLAG_USED_AS_IRQ  3  /* Line used as interrupt */
// ...

// Check direction
if (test_bit(FLAG_IS_OUT, &desc->flags))
    // Line is OUTPUT
else
    // Line is INPUT
```

### Common Scenarios

#### Scenario 1: Monitor LED Line

```c
// LED controlled by kernel driver
// /sys/class/leds/beaglebone:green:usr0 → gpio-53

// Try to read gpio-53
request = request_line_safe("/dev/gpiochip1", 21, "led-monitor",
                            GPIOD_LINE_DIRECTION_AS_IS);
// FAILS: -EBUSY (LED driver owns this line)

// Alternative: Read LED state via sysfs
FILE *f = fopen("/sys/class/leds/beaglebone:green:usr0/brightness", "r");
fscanf(f, "%d", &brightness);
```

#### Scenario 2: Toggle Between Apps

```c
// App 1: Request and hold
request = request_line_safe(..., GPIOD_LINE_DIRECTION_OUTPUT);
gpiod_line_request_set_value(request, offset, 1);
// Keep request alive (don't release)

// App 2: Try to request same line
request2 = request_line_safe(...);  // FAILS: -EBUSY

// Solution: App 1 must release first
gpiod_line_request_release(request);  // Now App 2 can request
```

#### Scenario 3: Read Your Own OUTPUT

```c
// Request as OUTPUT
request = request_line_safe(..., GPIOD_LINE_DIRECTION_OUTPUT);
gpiod_line_request_set_value(request, offset, 1);

// Later: Read back what we're driving
value = gpiod_line_request_get_value(request, offset);
// SUCCESS: Returns 1 (reads DATAOUT register)

// Useful for: verification, diagnostics, state machines
```

### Debugging Ownership Issues

```bash
# 1. Check what uses each GPIO
sudo cat /sys/kernel/debug/gpio

# 2. See open file descriptors on gpio devices
sudo lsof /dev/gpiochip*

# 3. Trace GPIO requests (requires kernel debug)
sudo trace-cmd record -e gpio:gpio_request -e gpio:gpio_free
# ... trigger your app ...
sudo trace-cmd report

# 4. Kernel messages
dmesg | grep -i gpio
```

### Key Takeaways

1. **One owner at a time** - GPIO lines use exclusive ownership
2. **Request fails if busy** - Line claimed by driver/process → `-EBUSY`
3. **Can read OUTPUT lines** - Request with `OUTPUT` or `AS_IS` direction
4. **Direction matters** - Choose based on hardware usage, not arbitrary
5. **Use AS_IS when unsure** - Preserves current configuration
6. **Monitor kernel lines via sysfs** - Can't request, but can observe

---

## Recommended Testing Workflow

### For Kernel Driver Development

When developing GPIO drivers, **minimize userspace complexity** - focus on kernel code:

```bash
# ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
# STEP 1: Build and load driver
# ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- modules
sudo insmod drivers/gpio/gpio-mydriver.ko

# ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
# STEP 2: Quick test with SYSFS (zero code!)
# ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
echo 60 > /sys/class/gpio/export
cat /sys/class/gpio/gpio60/value      # Read
echo 1 > /sys/class/gpio/gpio60/value # Write
echo 60 > /sys/class/gpio/unexport

# Alternative: Command line tools
gpioget gpiochip0 60
gpioset gpiochip0 60=1

# ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
# STEP 3: Debug kernel driver
# ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
dmesg | tail -20
sudo cat /sys/kernel/debug/gpio

# ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
# STEP 4: Iterate (no userspace app needed!)
# ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
sudo rmmod gpio-mydriver
# Edit driver code...
make modules
sudo insmod drivers/gpio/gpio-mydriver.ko
# Test again with sysfs/gpioget
```

### Quick Reference Table

| Task | Sysfs Method | Command Line Method | libgpiod C Code |
|------|--------------|---------------------|-----------------|
| **Read input** | `cat /sys/class/gpio/gpio60/value` | `gpioget gpiochip0 60` | ~50 lines |
| **Write output** | `echo 1 > /sys/class/gpio/gpio60/value` | `gpioset gpiochip0 60=1` | ~50 lines |
| **Monitor events** | `inotifywait` + edge | `gpiomon gpiochip0 60` | ~80 lines |
| **Blink LED** | Shell loop | Shell loop | ~60 lines |
| **Setup time** | 2 seconds | 1 second | 5 minutes |

**Winner for driver testing**: **Sysfs** or **Command line tools**

### Decision Tree

```
Need to test GPIO driver?
    │
    ├─ Quick test/debug
    │   └─> Use: sysfs (echo/cat commands)
    │
    ├─ Automation script
    │   └─> Use: gpioget/gpioset/gpiomon
    │
    ├─ Integrate into C app
    │   └─> Use: libgpiod C API
    │
    └─ Integrate into Python app
        └─> Use: libgpiod Python or sysfs file I/O
```

### Common Testing Scenarios

#### Scenario 1: Test New GPIO Driver

```bash
# After loading your driver module
sudo insmod gpio-mydriver.ko

# Find your GPIO chip
ls /dev/gpiochip*
gpiodetect

# Test basic functionality
echo 60 > /sys/class/gpio/export
echo out > /sys/class/gpio/gpio60/direction
echo 1 > /sys/class/gpio/gpio60/value

# Check kernel driver logs
dmesg | grep gpio

# Verify in debugfs
sudo cat /sys/kernel/debug/gpio | grep gpio60
# gpio-60  ( sysfs              ) out hi

# Cleanup
echo 60 > /sys/class/gpio/unexport
sudo rmmod gpio-mydriver
```

#### Scenario 2: Test Interrupt Functionality

```bash
# Setup edge detection
echo 48 > /sys/class/gpio/export
echo in > /sys/class/gpio/gpio48/direction
echo rising > /sys/class/gpio/gpio48/edge

# Terminal 1: Monitor
gpiomon --rising-edge gpiochip0 48

# Terminal 2: Trigger (if you have another output GPIO)
gpioset gpiochip0 60=0  # Low
gpioset gpiochip0 60=1  # High -> Triggers rising edge

# Check kernel interrupt stats
cat /proc/interrupts | grep gpio
```

#### Scenario 3: Validate Device Tree Binding

```bash
# After applying device tree overlay
sudo dtoverlay my-gpio-overlay.dtbo

# Check device tree
ls /proc/device-tree/ocp/gpio@4804c000/
cat /proc/device-tree/ocp/gpio@4804c000/status  # Should be "okay"

# Verify GPIO chip registered
dmesg | grep "gpio-"
# gpio-omap 4804c000.gpio: GPIO device initialized

# Test exported GPIO
gpioget gpiochip1 5
```

### Debugging Checklist

When GPIO doesn't work:

```bash
# ✓ 1. Is driver loaded?
lsmod | grep gpio
dmesg | grep -i gpio

# ✓ 2. Is chip registered?
ls /dev/gpiochip*
gpiodetect

# ✓ 3. Is line available?
sudo cat /sys/kernel/debug/gpio
# Check if line already claimed

# ✓ 4. Can export via sysfs?
echo N > /sys/class/gpio/export
# If fails: check dmesg for reason

# ✓ 5. Direction set correctly?
cat /sys/class/gpio/gpioN/direction

# ✓ 6. Hardware connected?
# Use multimeter to check voltage

# ✓ 7. Clock enabled? (for OMAP/AM33xx)
cat /sys/kernel/debug/clk/clk_summary | grep gpio
```

---

## Clock Management with GPIO

### OMAP/AM33xx with ti,sysc Parent

**Device tree:**
```dts
target-module@4c000 {
    compatible = "ti,sysc-omap2", "ti,sysc";
    clocks = <&l4ls_clkctrl 0x74 0>,    // fck (functional clock)
             <&l4ls_clkctrl 0x74 18>;   // dbclk (debounce clock)
    clock-names = "fck", "dbclk";
    
    gpio1: gpio@0 {
        compatible = "ti,omap4-gpio";
        reg = <0x0 0x1000>;
        gpio-controller;
        #gpio-cells = <2>;
    };
};
```

**Driver approach (child of ti,sysc):**
```c
// No explicit clock handling - parent manages fck
static int gpio_probe(struct platform_device *pdev)
{
    struct gpio_bank *bank;
    int ret;
    
    // Enable runtime PM - parent ti,sysc will enable fck
    pm_runtime_enable(&pdev->dev);
    ret = pm_runtime_get_sync(&pdev->dev);
    if (ret < 0) {
        pm_runtime_put_noidle(&pdev->dev);
        pm_runtime_disable(&pdev->dev);
        return ret;
    }
    
    // Now registers are accessible - fck is ON
    u32 revision = readl(bank->base + GPIO_REVISION);
    
    // Optional: Get debounce clock if needed
    bank->dbck = devm_clk_get(&pdev->dev, "dbclk");
    if (!IS_ERR(bank->dbck))
        clk_prepare(bank->dbck);  // Prepare but don't enable
    
    // Register GPIO chip...
    return 0;
}
```

### Standalone GPIO (Explicit Clock)

**Device tree:**
```dts
gpio_standalone: gpio@4804c000 {
    compatible = "my-gpio";
    reg = <0x4804c000 0x1000>;
    
    // Must specify clocks explicitly
    clocks = <&l4ls_clkctrl 0x74 0>;
    clock-names = "fck";
    
    gpio-controller;
    #gpio-cells = <2>;
};
```

**Driver approach:**
```c
#include <linux/clk.h>

static int gpio_standalone_probe(struct platform_device *pdev)
{
    struct clk *fck;
    int ret;
    
    // MUST get and enable clock explicitly
    fck = devm_clk_get(&pdev->dev, "fck");
    if (IS_ERR(fck)) {
        dev_err(&pdev->dev, "Failed to get fck\n");
        return PTR_ERR(fck);
    }
    
    ret = clk_prepare_enable(fck);
    if (ret) {
        dev_err(&pdev->dev, "Failed to enable fck\n");
        return ret;
    }
    
    // Now can access registers
    u32 val = readl(base + GPIO_DATAIN);
    
    // Register GPIO chip...
    return 0;
}

static void gpio_standalone_remove(struct platform_device *pdev)
{
    struct my_gpio *gpio = platform_get_drvdata(pdev);
    
    clk_disable_unprepare(gpio->fck);
}
```

### Clock Comparison

| Approach | Clock Management | When to Use |
|----------|------------------|-------------|
| **ti,sysc child** | Automatic via pm_runtime | BeagleBone, OMAP platforms |
| **Standalone** | Manual devm_clk_get/enable | Other platforms, portable drivers |

## Flow: Userspace → Hardware

```
Userspace:
    gpioget gpiochip0 5
        ↓ libgpiod
    ioctl(fd, GPIO_V2_GET_LINE_IOCTL)
        ↓ Kernel
gpiolib-cdev.c:
    gpio_ioctl() → gpio_v2_get_line_ioctl()
        ↓
gpiolib.c:
    gpiod_get_value(desc)
        ↓ chip->get()
gpio-omap.c / your-driver.c:
    my_gpio_get(chip, offset)
        ↓ readl(base + GPIO_DATAIN)
Hardware:
    Returns GPIO pin state
```

## GPIO Numbering and Base Allocation

### Why GPIO Numbers Start from 512 (Not 0)

Modern kernels use **auto-allocation** for GPIO base numbers, starting from `ARCH_NR_GPIOS` and allocating downward.

```c
// include/asm-generic/gpio.h
#ifndef ARCH_NR_GPIOS
#define ARCH_NR_GPIOS  512  // Default maximum
#endif
```

### Automatic Base Assignment

When GPIO chip driver sets `chip->base = -1`, kernel automatically assigns base:

```c
// drivers/gpio/gpiolib.c
int gpiochip_add(struct gpio_chip *chip)
{
    int base = chip->base;
    
    if (base < 0) {  // Auto-allocation requested
        base = gpiochip_find_base(chip->ngpio);
        chip->base = base;
    }
    ...
}

static int gpiochip_find_base(int ngpio)
{
    struct gpio_chip *chip;
    int base = ARCH_NR_GPIOS - ngpio;  // Start from 512
    
    // Find free space, allocate from high to low
    list_for_each_entry_reverse(chip, &gpio_chips, list) {
        if (chip->base + chip->ngpio <= base)
            break;
        else
            base = chip->base - ngpio;  // Try lower range
    }
    
    return base;
}
```

### Example: BeagleBone GPIO Numbering

```bash
$ ls /sys/class/gpio/
gpiochip512  gpiochip544  gpiochip576  unexport  export

# Each chip has 32 GPIOs:
# gpiochip512: GPIOs 512-543 (GPIO Bank 0)
# gpiochip544: GPIOs 544-575 (GPIO Bank 1)
# gpiochip576: GPIOs 576-607 (GPIO Bank 2)
```

**Mapping hardware names to Linux numbers:**

| Hardware Name | Linux GPIO Range | Calculation |
|---------------|------------------|-------------|
| GPIO0_31      | 512 + 31 = **543** | Bank 0, pin 31 |
| GPIO1_0       | 544 + 0  = **544** | Bank 1, pin 0 |
| GPIO1_31      | 544 + 31 = **575** | Bank 1, pin 31 |
| GPIO2_5       | 576 + 5  = **581** | Bank 2, pin 5 |

### Finding GPIO Base at Runtime

```bash
# Method 1: sysfs
cat /sys/class/gpio/gpiochip*/base
cat /sys/class/gpio/gpiochip*/ngpio
cat /sys/class/gpio/gpiochip*/label

# Method 2: debugfs (most detailed)
sudo cat /sys/kernel/debug/gpio
# Output:
# gpiochip0: GPIOs 512-543, parent: platform/44e07000.gpio, gpio-0-31:
# gpiochip1: GPIOs 544-575, parent: platform/4804c000.gpio, gpio-32-63:
```

### Why Not Hardcode GPIO Numbers

```c
// ❌ WRONG - Will break on different kernels/platforms
#define RESET_GPIO 31  // Assumes base=0, but base could be 512!
ret = gpio_request(RESET_GPIO, "reset");  // Fails with -EPROBE_DEFER

// ✅ CORRECT - Use device tree and descriptors
struct gpio_desc *reset_gpio;
reset_gpio = devm_gpiod_get(&pdev->dev, "reset", GPIOD_OUT_HIGH);
```

### Legacy Number Conversion (If Necessary)

If you MUST use legacy integer API:

```c
// Calculate Linux GPIO number from hardware bank/pin
#define GPIO_BANK_BASE(bank)  (512 + (bank * 32))
#define GPIO_TO_LINUX(bank, pin)  (GPIO_BANK_BASE(bank) + (pin))

// Example: GPIO1_31 on BeagleBone
int gpio_num = GPIO_TO_LINUX(1, 31);  // = 575
ret = gpio_request(gpio_num, "my-gpio");
```

**But better approach - use device tree:**

```dts
// In device tree
my_device {
    compatible = "vendor,my-device";
    gpios = <&gpio1 31 GPIO_ACTIVE_HIGH>;  // Bank 1, pin 31
};
```

```c
// In driver - no number needed!
desc = devm_gpiod_get(&pdev->dev, "gpio", GPIOD_OUT_HIGH);
```

## Best Practices

✅ **DO:**
- Use descriptor-based API (`gpiod_*`) in new code
- Use `devm_gpiod_get*()` for automatic cleanup
- Check return values with `IS_ERR()` and `PTR_ERR()`
- Use meaningful names in device tree (`reset-gpios`, `enable-gpios`)
- Use `_cansleep` variants when calling from contexts that can sleep
- Protect hardware access with spinlocks in GPIO chip drivers
- **Never hardcode GPIO numbers** - use device tree or GPIO lookup tables

❌ **DON'T:**
- Use legacy integer GPIO API (`gpio_request()`, `gpio_set_value()`)
- Hardcode GPIO numbers (base can change between kernels)
- Assume GPIO numbering starts at 0
- Call non-sleeping GPIO functions from atomic context if chip can sleep
- Access GPIO hardware registers without locking
- Forget to handle active-low polarity (framework handles it)
- Mix descriptor and integer APIs

## Common Errors

```c
// ERROR: Using wrong error check
if (!desc)  // WRONG
if (desc < 0)  // WRONG
if (IS_ERR(desc))  // CORRECT

// ERROR: Not checking for optional GPIO
desc = devm_gpiod_get_optional(&pdev->dev, "enable", GPIOD_OUT_HIGH);
if (IS_ERR(desc))
    return PTR_ERR(desc);
// If GPIO is not present, desc will be NULL (not error)
if (desc)
    gpiod_set_value(desc, 1);  // Only use if present
```
