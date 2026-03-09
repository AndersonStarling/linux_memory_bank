# I2C Subsystem

## I2C Architecture Overview

I2C (Inter-Integrated Circuit) là bus 2 dây (SDA, SCL) được sử dụng rộng rãi để kết nối các thiết bị tốc độ thấp đến trung bình. Linux kernel cung cấp framework hoàn chỉnh cho I2C với kiến trúc 3 layers:

```
┌─────────────────────────────────────────┐
│   Device Driver (I2C Client)            │  <- i2c_driver, probe/remove
├─────────────────────────────────────────┤
│   I2C Core (i2c-core.c)                  │  <- Bus management, transfer
├─────────────────────────────────────────┤
│   I2C Adapter/Algorithm (Controller)    │  <- Hardware-specific driver
└─────────────────────────────────────────┘
```

### Key Concepts
- **I2C Adapter/Controller**: Hardware I2C bus controller (master/controller)
- **I2C Algorithm**: Transfer implementation (bit-banging, hardware, etc.)
- **I2C Client/Device**: Slave device on the I2C bus (sensor, EEPROM, etc.)
- **I2C Driver**: Software driver for I2C client devices
- **I2C Message**: Data transfer unit containing address, flags, buffer
- **SMBus**: Subset of I2C with specific command semantics

### I2C Frequency Modes
```c
#define I2C_MAX_STANDARD_MODE_FREQ      100000      // 100 kHz
#define I2C_MAX_FAST_MODE_FREQ          400000      // 400 kHz
#define I2C_MAX_FAST_MODE_PLUS_FREQ     1000000     // 1 MHz
#define I2C_MAX_HIGH_SPEED_MODE_FREQ    3400000     // 3.4 MHz
#define I2C_MAX_ULTRA_FAST_MODE_FREQ    5000000     // 5 MHz
```

## Essential Headers

```c
// For I2C client drivers (devices using I2C)
#include <linux/i2c.h>

// For I2C adapter drivers (implementing I2C controller)
#include <linux/i2c.h>
#include <linux/i2c-dev.h>

// For device tree
#include <linux/of.h>
#include <linux/of_device.h>
```

## I2C Client Driver (Device Driver Using I2C)

### I2C Driver Structure

```c
#include <linux/i2c.h>
#include <linux/module.h>

struct mydev_data {
    struct i2c_client *client;
    struct device *dev;
    int some_value;
    struct mutex lock;
};

// Device ID table
static const struct i2c_device_id mydev_id[] = {
    { "mydevice", 0 },
    { "mydevice-v2", 1 },
    { }
};
MODULE_DEVICE_TABLE(i2c, mydev_id);

// Device tree match table
static const struct of_device_id mydev_of_match[] = {
    { .compatible = "vendor,mydevice", },
    { .compatible = "vendor,mydevice-v2", },
    { }
};
MODULE_DEVICE_TABLE(of, mydev_of_match);

// Probe function
static int mydev_probe(struct i2c_client *client)
{
    struct mydev_data *data;
    int ret;
    
    dev_info(&client->dev, "Probing I2C device at address 0x%02x\n",
             client->addr);
    
    // Allocate private data
    data = devm_kzalloc(&client->dev, sizeof(*data), GFP_KERNEL);
    if (!data)
        return -ENOMEM;
    
    data->client = client;
    data->dev = &client->dev;
    mutex_init(&data->lock);
    
    // Store private data
    i2c_set_clientdata(client, data);
    
    // Check if device is present (optional)
    ret = i2c_smbus_read_byte(client);
    if (ret < 0) {
        dev_err(&client->dev, "Failed to communicate with device\n");
        return ret;
    }
    
    // Initialize device
    ret = mydev_init_device(data);
    if (ret)
        return ret;
    
    dev_info(&client->dev, "Device initialized successfully\n");
    
    return 0;
}

// Remove function
static void mydev_remove(struct i2c_client *client)
{
    struct mydev_data *data = i2c_get_clientdata(client);
    
    dev_info(&client->dev, "Removing device\n");
    
    // Cleanup
    mutex_destroy(&data->lock);
}

// I2C driver structure
static struct i2c_driver mydev_driver = {
    .driver = {
        .name = "mydevice",
        .of_match_table = mydev_of_match,
    },
    .probe = mydev_probe,
    .remove = mydev_remove,
    .id_table = mydev_id,
};

module_i2c_driver(mydev_driver);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Your Name");
MODULE_DESCRIPTION("My I2C device driver");
```

### I2C Data Transfer - Raw Messages

```c
// Read from I2C device using i2c_transfer
int mydev_read_reg(struct i2c_client *client, u8 reg, u8 *val)
{
    struct i2c_msg msgs[2];
    int ret;
    
    // Write register address
    msgs[0].addr = client->addr;
    msgs[0].flags = 0;              // Write
    msgs[0].len = 1;
    msgs[0].buf = &reg;
    
    // Read data
    msgs[1].addr = client->addr;
    msgs[1].flags = I2C_M_RD;       // Read
    msgs[1].len = 1;
    msgs[1].buf = val;
    
    ret = i2c_transfer(client->adapter, msgs, 2);
    if (ret != 2) {
        dev_err(&client->dev, "I2C transfer failed: %d\n", ret);
        return ret < 0 ? ret : -EIO;
    }
    
    return 0;
}

// Write to I2C device
int mydev_write_reg(struct i2c_client *client, u8 reg, u8 val)
{
    u8 buf[2] = { reg, val };
    struct i2c_msg msg;
    int ret;
    
    msg.addr = client->addr;
    msg.flags = 0;              // Write
    msg.len = 2;
    msg.buf = buf;
    
    ret = i2c_transfer(client->adapter, &msg, 1);
    if (ret != 1) {
        dev_err(&client->dev, "I2C write failed: %d\n", ret);
        return ret < 0 ? ret : -EIO;
    }
    
    return 0;
}

// Bulk read
int mydev_read_block(struct i2c_client *client, u8 reg, u8 *data, int len)
{
    struct i2c_msg msgs[2];
    int ret;
    
    msgs[0].addr = client->addr;
    msgs[0].flags = 0;
    msgs[0].len = 1;
    msgs[0].buf = &reg;
    
    msgs[1].addr = client->addr;
    msgs[1].flags = I2C_M_RD;
    msgs[1].len = len;
    msgs[1].buf = data;
    
    ret = i2c_transfer(client->adapter, msgs, 2);
    return (ret == 2) ? 0 : -EIO;
}
```

### I2C Helper Functions (Simple API)

```c
// Simple byte transfer
int val;

// Master send (write data to device)
char buf[] = { 0x10, 0x20, 0x30 };
ret = i2c_master_send(client, buf, sizeof(buf));
if (ret != sizeof(buf))
    return -EIO;

// Master receive (read data from device)
char rbuf[10];
ret = i2c_master_recv(client, rbuf, sizeof(rbuf));
if (ret != sizeof(rbuf))
    return -EIO;
```

### SMBus API (Recommended for Simple Devices)

```c
// SMBus byte operations
s32 ret;

// Read byte (no register)
ret = i2c_smbus_read_byte(client);
if (ret < 0)
    return ret;

// Write byte (no register)
ret = i2c_smbus_write_byte(client, 0xAB);

// Read byte from register
ret = i2c_smbus_read_byte_data(client, 0x10);
if (ret < 0)
    return ret;
u8 value = (u8)ret;

// Write byte to register
ret = i2c_smbus_write_byte_data(client, 0x10, 0x42);

// Read word (16-bit) from register
ret = i2c_smbus_read_word_data(client, 0x20);
if (ret < 0)
    return ret;
u16 word = (u16)ret;

// Write word to register
ret = i2c_smbus_write_word_data(client, 0x20, 0x1234);

// Read word with byte swap
ret = i2c_smbus_read_word_swapped(client, 0x20);

// Write word with byte swap
ret = i2c_smbus_write_word_swapped(client, 0x20, 0x1234);

// Read block (up to 32 bytes)
u8 data[32];
ret = i2c_smbus_read_block_data(client, 0x30, data);
if (ret < 0)
    return ret;
// ret contains number of bytes read

// Write block
u8 wdata[] = { 0x01, 0x02, 0x03, 0x04 };
ret = i2c_smbus_write_block_data(client, 0x30, sizeof(wdata), wdata);

// Read I2C block (fixed length)
u8 buffer[16];
ret = i2c_smbus_read_i2c_block_data(client, 0x40, sizeof(buffer), buffer);

// Write I2C block
ret = i2c_smbus_write_i2c_block_data(client, 0x40, sizeof(buffer), buffer);
```

## Device Tree Bindings

### I2C Controller Node

```dts
// I2C controller (adapter)
i2c0: i2c@48070000 {
    compatible = "ti,omap4-i2c";
    reg = <0x48070000 0x100>;
    interrupts = <GIC_SPI 56 IRQ_TYPE_LEVEL_HIGH>;
    #address-cells = <1>;
    #size-cells = <0>;
    ti,hwmods = "i2c1";
    clock-frequency = <400000>;     // 400 kHz (Fast Mode)
};

// Generic I2C controller
&i2c1 {
    status = "okay";
    clock-frequency = <100000>;     // 100 kHz (Standard Mode)
    
    // I2C devices on this bus...
};
```

### I2C Client/Device Node

```dts
&i2c0 {
    status = "okay";
    
    // Simple I2C device
    eeprom@50 {
        compatible = "atmel,24c256";
        reg = <0x50>;               // 7-bit I2C address
        pagesize = <64>;
    };
    
    // I2C device with interrupt
    touchscreen@38 {
        compatible = "edt,edt-ft5406";
        reg = <0x38>;
        interrupt-parent = <&gpio1>;
        interrupts = <7 IRQ_TYPE_EDGE_FALLING>;
        reset-gpios = <&gpio1 5 GPIO_ACTIVE_LOW>;
        touchscreen-size-x = <800>;
        touchscreen-size-y = <480>;
    };
    
    // I2C device with custom properties
    sensor@68 {
        compatible = "vendor,my-sensor";
        reg = <0x68>;
        vdd-supply = <&vdd_3v3>;
        sampling-rate = <100>;
    };
};
```

### Parsing Device Tree Properties

```c
static int mydev_probe(struct i2c_client *client)
{
    struct device *dev = &client->dev;
    u32 value;
    
    // Read u32 property
    if (of_property_read_u32(dev->of_node, "sampling-rate", &value)) {
        dev_warn(dev, "No sampling-rate, using default\n");
        value = 50;
    }
    
    // Read string property
    const char *str;
    if (of_property_read_string(dev->of_node, "label", &str) == 0)
        dev_info(dev, "Device label: %s\n", str);
    
    // Check boolean property
    if (of_property_read_bool(dev->of_node, "enable-feature"))
        /* feature enabled */
    
    // Get GPIO from device tree
    struct gpio_desc *reset_gpio;
    reset_gpio = devm_gpiod_get(dev, "reset", GPIOD_OUT_HIGH);
    if (IS_ERR(reset_gpio))
        return PTR_ERR(reset_gpio);
    
    // Get regulator
    struct regulator *vdd;
    vdd = devm_regulator_get(dev, "vdd");
    if (IS_ERR(vdd))
        return PTR_ERR(vdd);
    
    return 0;
}
```

## I2C Adapter Driver (Controller Implementation)

### I2C Algorithm Structure

```c
struct i2c_algorithm {
    // Main transfer function
    int (*xfer)(struct i2c_adapter *adap, struct i2c_msg *msgs, int num);
    
    // Atomic transfer (for late system operations)
    int (*xfer_atomic)(struct i2c_adapter *adap,
                       struct i2c_msg *msgs, int num);
    
    // SMBus-specific transfer (optional)
    int (*smbus_xfer)(struct i2c_adapter *adap, u16 addr,
                      unsigned short flags, char read_write,
                      u8 command, int size, union i2c_smbus_data *data);
    
    // Return supported functionality flags
    u32 (*functionality)(struct i2c_adapter *adap);
};
```

### Implementing I2C Adapter Driver

```c
#include <linux/i2c.h>
#include <linux/platform_device.h>
#include <linux/clk.h>
#include <linux/io.h>

struct my_i2c_dev {
    void __iomem *base;
    struct clk *clk;
    struct i2c_adapter adapter;
    struct completion cmd_complete;
    int irq;
    u8 *buf;
    size_t buf_len;
};

// Transfer implementation
static int my_i2c_xfer(struct i2c_adapter *adap, struct i2c_msg *msgs, int num)
{
    struct my_i2c_dev *dev = i2c_get_adapdata(adap);
    int i, ret;
    
    for (i = 0; i < num; i++) {
        struct i2c_msg *msg = &msgs[i];
        
        // Setup transfer
        dev->buf = msg->buf;
        dev->buf_len = msg->len;
        
        // Write slave address
        writel(msg->addr, dev->base + SLAVE_ADDR_REG);
        
        // Configure transfer direction
        if (msg->flags & I2C_M_RD) {
            // Read operation
            writel(msg->len, dev->base + CNT_REG);
            writel(CTRL_START | CTRL_READ, dev->base + CTRL_REG);
        } else {
            // Write operation
            writel(msg->len, dev->base + CNT_REG);
            writel(CTRL_START | CTRL_WRITE, dev->base + CTRL_REG);
        }
        
        // Wait for completion
        ret = wait_for_completion_timeout(&dev->cmd_complete,
                                           adap->timeout);
        if (!ret) {
            dev_err(&adap->dev, "Transfer timeout\n");
            return -ETIMEDOUT;
        }
        
        // Check for errors
        u32 status = readl(dev->base + STATUS_REG);
        if (status & STATUS_NACK) {
            dev_err(&adap->dev, "NACK received\n");
            return -ENXIO;
        }
        if (status & STATUS_AL) {
            dev_err(&adap->dev, "Arbitration lost\n");
            return -EAGAIN;
        }
    }
    
    return num;  // Return number of messages transferred
}

// Functionality flags
static u32 my_i2c_func(struct i2c_adapter *adap)
{
    return I2C_FUNC_I2C |
           I2C_FUNC_SMBUS_EMUL |
           I2C_FUNC_SMBUS_READ_BYTE |
           I2C_FUNC_SMBUS_WRITE_BYTE |
           I2C_FUNC_SMBUS_READ_BYTE_DATA |
           I2C_FUNC_SMBUS_WRITE_BYTE_DATA |
           I2C_FUNC_SMBUS_READ_WORD_DATA |
           I2C_FUNC_SMBUS_WRITE_WORD_DATA |
           I2C_FUNC_SMBUS_READ_BLOCK_DATA |
           I2C_FUNC_SMBUS_WRITE_BLOCK_DATA;
}

static const struct i2c_algorithm my_i2c_algo = {
    .xfer = my_i2c_xfer,
    .functionality = my_i2c_func,
};

// IRQ handler
static irqreturn_t my_i2c_isr(int irq, void *dev_id)
{
    struct my_i2c_dev *dev = dev_id;
    u32 status;
    
    status = readl(dev->base + STATUS_REG);
    
    // Handle data ready
    if (status & STATUS_RRDY) {
        // Read data
        *dev->buf++ = readl(dev->base + DATA_REG);
        dev->buf_len--;
    }
    
    if (status & STATUS_XRDY) {
        // Write data
        if (dev->buf_len > 0) {
            writel(*dev->buf++, dev->base + DATA_REG);
            dev->buf_len--;
        }
    }
    
    // Transfer complete
    if (status & STATUS_ARDY) {
        complete(&dev->cmd_complete);
    }
    
    // Clear interrupt
    writel(status, dev->base + STATUS_REG);
    
    return IRQ_HANDLED;
}

// Probe function
static int my_i2c_probe(struct platform_device *pdev)
{
    struct my_i2c_dev *dev;
    struct resource *res;
    int ret;
    
    dev = devm_kzalloc(&pdev->dev, sizeof(*dev), GFP_KERNEL);
    if (!dev)
        return -ENOMEM;
    
    // Map registers
    res = platform_get_resource(pdev, IORESOURCE_MEM, 0);
    dev->base = devm_ioremap_resource(&pdev->dev, res);
    if (IS_ERR(dev->base))
        return PTR_ERR(dev->base);
    
    // Get IRQ
    dev->irq = platform_get_irq(pdev, 0);
    if (dev->irq < 0)
        return dev->irq;
    
    // Get and enable clock
    dev->clk = devm_clk_get(&pdev->dev, "fck");
    if (IS_ERR(dev->clk))
        return PTR_ERR(dev->clk);
    
    ret = clk_prepare_enable(dev->clk);
    if (ret)
        return ret;
    
    // Initialize completion
    init_completion(&dev->cmd_complete);
    
    // Setup I2C adapter
    dev->adapter.owner = THIS_MODULE;
    dev->adapter.class = I2C_CLASS_DEPRECATED;
    dev->adapter.algo = &my_i2c_algo;
    dev->adapter.dev.parent = &pdev->dev;
    dev->adapter.dev.of_node = pdev->dev.of_node;
    dev->adapter.timeout = msecs_to_jiffies(1000);
    dev->adapter.retries = 3;
    snprintf(dev->adapter.name, sizeof(dev->adapter.name),
             "My I2C adapter");
    
    i2c_set_adapdata(&dev->adapter, dev);
    
    // Request IRQ
    ret = devm_request_irq(&pdev->dev, dev->irq, my_i2c_isr,
                           0, dev_name(&pdev->dev), dev);
    if (ret) {
        dev_err(&pdev->dev, "Failed to request IRQ\n");
        goto err_clk;
    }
    
    // Register I2C adapter
    ret = i2c_add_adapter(&dev->adapter);
    if (ret) {
        dev_err(&pdev->dev, "Failed to add I2C adapter\n");
        goto err_clk;
    }
    
    platform_set_drvdata(pdev, dev);
    
    dev_info(&pdev->dev, "I2C adapter registered\n");
    
    return 0;

err_clk:
    clk_disable_unprepare(dev->clk);
    return ret;
}

static void my_i2c_remove(struct platform_device *pdev)
{
    struct my_i2c_dev *dev = platform_get_drvdata(pdev);
    
    i2c_del_adapter(&dev->adapter);
    clk_disable_unprepare(dev->clk);
}

static const struct of_device_id my_i2c_of_match[] = {
    { .compatible = "vendor,my-i2c", },
    { }
};
MODULE_DEVICE_TABLE(of, my_i2c_of_match);

static struct platform_driver my_i2c_driver = {
    .probe = my_i2c_probe,
    .remove = my_i2c_remove,
    .driver = {
        .name = "my-i2c",
        .of_match_table = my_i2c_of_match,
    },
};
module_platform_driver(my_i2c_driver);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Your Name");
MODULE_DESCRIPTION("My I2C adapter driver");
```

## Common I2C Patterns

### Register Map Abstraction (regmap)

```c
#include <linux/regmap.h>

static const struct regmap_config mydev_regmap_config = {
    .reg_bits = 8,
    .val_bits = 8,
    .max_register = 0xFF,
};

static int mydev_probe(struct i2c_client *client)
{
    struct regmap *regmap;
    unsigned int val;
    
    regmap = devm_regmap_init_i2c(client, &mydev_regmap_config);
    if (IS_ERR(regmap))
        return PTR_ERR(regmap);
    
    // Read register
    regmap_read(regmap, 0x10, &val);
    
    // Write register
    regmap_write(regmap, 0x20, 0x42);
    
    // Update bits
    regmap_update_bits(regmap, 0x30, 0x0F, 0x05);
    
    return 0;
}
```

### Multi-byte Read/Write

```c
// Read multiple bytes
int mydev_read_multi(struct i2c_client *client, u8 reg, u8 *data, int len)
{
    int ret;
    struct i2c_msg msgs[] = {
        {
            .addr = client->addr,
            .flags = 0,
            .len = 1,
            .buf = &reg,
        },
        {
            .addr = client->addr,
            .flags = I2C_M_RD,
            .len = len,
            .buf = data,
        },
    };
    
    ret = i2c_transfer(client->adapter, msgs, ARRAY_SIZE(msgs));
    return (ret == ARRAY_SIZE(msgs)) ? 0 : -EIO;
}

// Write multiple bytes
int mydev_write_multi(struct i2c_client *client, u8 reg, const u8 *data, int len)
{
    u8 *buf;
    int ret;
    
    buf = kmalloc(len + 1, GFP_KERNEL);
    if (!buf)
        return -ENOMEM;
    
    buf[0] = reg;
    memcpy(&buf[1], data, len);
    
    ret = i2c_master_send(client, buf, len + 1);
    kfree(buf);
    
    return (ret == len + 1) ? 0 : -EIO;
}
```

### I2C with IRQ

```c
static irqreturn_t mydev_irq_handler(int irq, void *dev_id)
{
    struct mydev_data *data = dev_id;
    u8 status;
    
    // Read interrupt status
    i2c_smbus_read_byte_data(data->client, STATUS_REG);
    
    // Handle interrupt
    if (status & INT_DATA_READY) {
        // Read data
        queue_work(data->workqueue, &data->work);
    }
    
    return IRQ_HANDLED;
}

static int mydev_probe(struct i2c_client *client)
{
    struct mydev_data *data;
    int ret;
    
    // Allocate data
    data = devm_kzalloc(&client->dev, sizeof(*data), GFP_KERNEL);
    if (!data)
        return -ENOMEM;
    
    data->client = client;
    
    // Request IRQ
    if (client->irq) {
        ret = devm_request_threaded_irq(&client->dev, client->irq,
                                         NULL, mydev_irq_handler,
                                         IRQF_TRIGGER_FALLING | IRQF_ONESHOT,
                                         client->name, data);
        if (ret)
            return ret;
    }
    
    return 0;
}
```

## Debugging I2C

### i2c-tools Commands

```bash
# List all I2C buses
i2cdetect -l

# Scan I2C bus 0 for devices
i2cdetect -y 0

# Read byte from device 0x50, register 0x00
i2cget -y 0 0x50 0x00

# Write byte to device 0x50, register 0x10, value 0x42
i2cset -y 0 0x50 0x10 0x42

# Dump all registers from device
i2cdump -y 0 0x50
```

### sysfs Interface

```bash
# List I2C buses
ls /sys/bus/i2c/devices/

# Show I2C bus details
cat /sys/bus/i2c/devices/i2c-0/name

# Manually instantiate I2C device
echo eeprom 0x50 > /sys/bus/i2c/devices/i2c-0/new_device

# Delete device
echo 0x50 > /sys/bus/i2c/devices/i2c-0/delete_device
```

### Kernel Debugging

```c
// Enable dynamic debug
echo 'file drivers/i2c/* +p' > /sys/kernel/debug/dynamic_debug/control

// View I2C transfers
echo 'file i2c-core-base.c +p' > /sys/kernel/debug/dynamic_debug/control
```

## Best Practices

✅ **DO:**
- Use `devm_*` functions for automatic resource cleanup
- Check return values from all I2C operations
- Use SMBus API for simple register-based devices
- Use regmap for complex register maps
- Implement proper error handling (NACK, timeout, arbitration loss)
- Use threaded IRQ handlers for I2C operations in IRQ context
- Set appropriate timeout and retry values
- Use device tree for device configuration

❌ **DON'T:**
- Call I2C transfer functions from hard IRQ context (may sleep)
- Ignore NACK errors (device may not be present)
- Use fixed delays; use completion for synchronization
- Forget to check I2C_FUNC_* capabilities before using SMBus functions
- Access I2C from atomic context without xfer_atomic support
- Mix SMBus and raw I2C transfers unnecessarily

## Common Errors & Solutions

```c
// Error: -ENXIO (No such device or address)
// Cause: Device not responding (NACK)
// Solution: Check device address, power, pull-ups

// Error: -ETIMEDOUT
// Cause: Transfer timeout
// Solution: Increase adapter timeout, check clock stretching

// Error: -EAGAIN
// Cause: Arbitration lost (multi-master bus)
// Solution: Retry transfer, check bus arbitration

// Error: -EREMOTEIO
// Cause: Remote I/O error
// Solution: Check signal integrity, reduce speed

// Error: -EPROTO
// Cause: Protocol error
// Solution: Verify I2C timing, check for noise

// Checking functionality
if (!i2c_check_functionality(client->adapter, I2C_FUNC_SMBUS_BYTE_DATA)) {
    dev_err(&client->dev, "SMBus byte data not supported\n");
    return -ENODEV;
}
```

## I2C Message Flags

```c
// Message flags (used in struct i2c_msg)
#define I2C_M_RD            0x0001  // Read data (from slave to master)
#define I2C_M_TEN           0x0010  // 10-bit address
#define I2C_M_DMA_SAFE      0x0200  // Buffer is DMA safe
#define I2C_M_RECV_LEN      0x0400  // Length in first byte
#define I2C_M_NO_RD_ACK     0x0800  // Don't ACK on read
#define I2C_M_IGNORE_NAK    0x1000  // Ignore NAK from device
#define I2C_M_REV_DIR_ADDR  0x2000  // Reverse direction flag
#define I2C_M_NOSTART       0x4000  // Don't send START condition
#define I2C_M_STOP          0x8000  // Send STOP after this message
```
