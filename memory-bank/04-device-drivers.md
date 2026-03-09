# Device Driver Development

## Character Device Driver Template

```c
// SPDX-License-Identifier: GPL-2.0
#include <linux/module.h>
#include <linux/fs.h>
#include <linux/cdev.h>
#include <linux/device.h>
#include <linux/uaccess.h>

#define DEVICE_NAME "mydev"
#define CLASS_NAME "myclass"

static dev_t dev_num;
static struct class *dev_class;
static struct cdev my_cdev;

static int dev_open(struct inode *inode, struct file *file)
{
    pr_info("%s: Device opened\n", DEVICE_NAME);
    return 0;
}

static int dev_release(struct inode *inode, struct file *file)
{
    pr_info("%s: Device closed\n", DEVICE_NAME);
    return 0;
}

static ssize_t dev_read(struct file *file, char __user *buf, 
                        size_t len, loff_t *offset)
{
    char kbuf[] = "Hello from kernel\n";
    size_t to_copy = min(len, sizeof(kbuf));
    
    if (*offset >= sizeof(kbuf))
        return 0;
    
    if (copy_to_user(buf, kbuf, to_copy))
        return -EFAULT;
    
    *offset += to_copy;
    return to_copy;
}

static ssize_t dev_write(struct file *file, const char __user *buf,
                         size_t len, loff_t *offset)
{
    char kbuf[256];
    size_t to_copy = min(len, sizeof(kbuf) - 1);
    
    if (copy_from_user(kbuf, buf, to_copy))
        return -EFAULT;
    
    kbuf[to_copy] = '\0';
    pr_info("%s: Received: %s\n", DEVICE_NAME, kbuf);
    return to_copy;
}

static struct file_operations fops = {
    .owner = THIS_MODULE,
    .open = dev_open,
    .release = dev_release,
    .read = dev_read,
    .write = dev_write,
};

static int __init mydev_init(void)
{
    // Allocate device number
    if (alloc_chrdev_region(&dev_num, 0, 1, DEVICE_NAME) < 0) {
        pr_err("Failed to allocate device number\n");
        return -1;
    }
    
    // Initialize cdev
    cdev_init(&my_cdev, &fops);
    my_cdev.owner = THIS_MODULE;
    
    // Add cdev
    if (cdev_add(&my_cdev, dev_num, 1) < 0) {
        pr_err("Failed to add cdev\n");
        goto err_cdev;
    }
    
    // Create device class
    dev_class = class_create(CLASS_NAME);
    if (IS_ERR(dev_class)) {
        pr_err("Failed to create class\n");
        goto err_class;
    }
    
    // Create device
    if (IS_ERR(device_create(dev_class, NULL, dev_num, NULL, DEVICE_NAME))) {
        pr_err("Failed to create device\n");
        goto err_device;
    }
    
    pr_info("%s: Module loaded (Major: %d)\n", DEVICE_NAME, MAJOR(dev_num));
    return 0;

err_device:
    class_destroy(dev_class);
err_class:
    cdev_del(&my_cdev);
err_cdev:
    unregister_chrdev_region(dev_num, 1);
    return -1;
}

static void __exit mydev_exit(void)
{
    device_destroy(dev_class, dev_num);
    class_destroy(dev_class);
    cdev_del(&my_cdev);
    unregister_chrdev_region(dev_num, 1);
    pr_info("%s: Module unloaded\n", DEVICE_NAME);
}

module_init(mydev_init);
module_exit(mydev_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Your Name");
MODULE_DESCRIPTION("Character device driver example");
```

## Platform Device Driver

```c
// SPDX-License-Identifier: GPL-2.0
#include <linux/module.h>
#include <linux/platform_device.h>
#include <linux/of.h>
#include <linux/io.h>

struct mydev_data {
    void __iomem *base;
    int irq;
    struct device *dev;
};

static int mydev_probe(struct platform_device *pdev)
{
    struct mydev_data *data;
    struct resource *res;
    u32 reg_value;
    
    dev_info(&pdev->dev, "Probing device\n");
    
    // Allocate device data
    data = devm_kzalloc(&pdev->dev, sizeof(*data), GFP_KERNEL);
    if (!data)
        return -ENOMEM;
    
    data->dev = &pdev->dev;
    
    // Get memory resource
    res = platform_get_resource(pdev, IORESOURCE_MEM, 0);
    if (!res) {
        dev_err(&pdev->dev, "No memory resource\n");
        return -ENODEV;
    }
    
    // Map memory
    data->base = devm_ioremap_resource(&pdev->dev, res);
    if (IS_ERR(data->base))
        return PTR_ERR(data->base);
    
    // Get IRQ
    data->irq = platform_get_irq(pdev, 0);
    if (data->irq < 0) {
        dev_err(&pdev->dev, "No IRQ resource\n");
        return data->irq;
    }
    
    // Parse device tree properties
    if (of_property_read_u32(pdev->dev.of_node, "reg-value", &reg_value)) {
        dev_warn(&pdev->dev, "No reg-value property, using default\n");
        reg_value = 0;
    }
    
    // Store private data
    platform_set_drvdata(pdev, data);
    
    dev_info(&pdev->dev, "Probe successful (IRQ: %d)\n", data->irq);
    return 0;
}

static int mydev_remove(struct platform_device *pdev)
{
    struct mydev_data *data = platform_get_drvdata(pdev);
    
    dev_info(&pdev->dev, "Removing device\n");
    
    // Cleanup handled by devm_* functions automatically
    return 0;
}

static const struct of_device_id mydev_of_match[] = {
    { .compatible = "vendor,mydevice", },
    { /* sentinel */ }
};
MODULE_DEVICE_TABLE(of, mydev_of_match);

static struct platform_driver mydev_driver = {
    .probe = mydev_probe,
    .remove = mydev_remove,
    .driver = {
        .name = "mydevice",
        .of_match_table = mydev_of_match,
    },
};

module_platform_driver(mydev_driver);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Your Name");
MODULE_DESCRIPTION("Platform device driver example");
```

## Device Tree Parsing

```c
// Get device node
struct device_node *np = pdev->dev.of_node;

// Read u32 property
u32 value;
if (of_property_read_u32(np, "property-name", &value))
    dev_err(&pdev->dev, "Failed to read property\n");

// Read string property
const char *str;
if (of_property_read_string(np, "string-prop", &str))
    dev_err(&pdev->dev, "Failed to read string\n");

// Read array
u32 array[4];
if (of_property_read_u32_array(np, "array-prop", array, 4))
    dev_err(&pdev->dev, "Failed to read array\n");

// Check if property exists
if (of_property_read_bool(np, "enable-feature"))
    /* feature is enabled */

// Get GPIO from device tree
int gpio = of_get_named_gpio(np, "reset-gpios", 0);
if (gpio_is_valid(gpio)) {
    devm_gpio_request_one(&pdev->dev, gpio, 
                          GPIOF_OUT_INIT_HIGH, "reset");
}

// Get regulator
struct regulator *reg = devm_regulator_get(&pdev->dev, "vdd");
if (IS_ERR(reg))
    return PTR_ERR(reg);
```

## Interrupt Handling

```c
#include <linux/interrupt.h>

static irqreturn_t mydev_irq_handler(int irq, void *dev_id)
{
    struct mydev_data *data = dev_id;
    u32 status;
    
    // Read interrupt status
    status = readl(data->base + STATUS_REG);
    
    // Clear interrupt
    writel(status, data->base + STATUS_REG);
    
    // Handle interrupt
    if (status & IRQ_FLAG_RX)
        /* handle receive */
    
    if (status & IRQ_FLAG_TX)
        /* handle transmit */
    
    return IRQ_HANDLED;
}

// Request IRQ (in probe)
ret = devm_request_irq(&pdev->dev, data->irq, mydev_irq_handler,
                       IRQF_TRIGGER_HIGH, dev_name(&pdev->dev), data);
if (ret) {
    dev_err(&pdev->dev, "Failed to request IRQ\n");
    return ret;
}

// Threaded IRQ (for lengthy processing)
static irqreturn_t mydev_irq_quick(int irq, void *dev_id)
{
    // Quick handling, return IRQ_WAKE_THREAD to run threaded part
    return IRQ_WAKE_THREAD;
}

static irqreturn_t mydev_irq_thread(int irq, void *dev_id)
{
    // Lengthy processing (can sleep)
    return IRQ_HANDLED;
}

devm_request_threaded_irq(&pdev->dev, data->irq,
                          mydev_irq_quick, mydev_irq_thread,
                          IRQF_TRIGGER_HIGH, dev_name(&pdev->dev), data);
```

## sysfs Attributes

```c
// Simple attribute
static ssize_t status_show(struct device *dev,
                           struct device_attribute *attr, char *buf)
{
    struct mydev_data *data = dev_get_drvdata(dev);
    return sprintf(buf, "%d\n", data->status);
}

static ssize_t status_store(struct device *dev,
                            struct device_attribute *attr,
                            const char *buf, size_t count)
{
    struct mydev_data *data = dev_get_drvdata(dev);
    int ret;
    
    ret = kstrtoint(buf, 10, &data->status);
    if (ret)
        return ret;
    
    return count;
}
static DEVICE_ATTR_RW(status);

// Register attribute
ret = device_create_file(&pdev->dev, &dev_attr_status);

// Or use attribute group
static struct attribute *mydev_attrs[] = {
    &dev_attr_status.attr,
    NULL,
};
ATTRIBUTE_GROUPS(mydev);

// In platform_driver
.driver = {
    .name = "mydevice",
    .of_match_table = mydev_of_match,
    .groups = mydev_groups,
},
```

## Resource Management (devm_*)

```c
// Use devm_* functions for automatic cleanup

// Memory
data = devm_kzalloc(&pdev->dev, sizeof(*data), GFP_KERNEL);

// I/O memory
base = devm_ioremap_resource(&pdev->dev, res);

// GPIO
devm_gpio_request_one(&pdev->dev, gpio, flags, label);

// IRQ
devm_request_irq(&pdev->dev, irq, handler, flags, name, dev_id);

// Clock
clk = devm_clk_get(&pdev->dev, "core");

// Regulator
reg = devm_regulator_get(&pdev->dev, "vdd");

// PWM
pwm = devm_pwm_get(&pdev->dev, NULL);
```
