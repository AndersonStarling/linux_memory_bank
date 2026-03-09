# SPI Subsystem

## SPI Architecture Overview

SPI (Serial Peripheral Interface) là bus đồng bộ 4-dây được sử dụng rộng rãi để kết nối microcontrollers với sensors, memory và peripherals. Linux kernel cung cấp framework hoàn chỉnh cho SPI với kiến trúc 3 layers:

```
┌─────────────────────────────────────────┐
│   Protocol Driver (SPI Device)          │  <- spi_driver, spi_message
├─────────────────────────────────────────┤
│   SPI Core (spi.c)                       │  <- Message queue, transfer
├─────────────────────────────────────────┤
│   Controller Driver (SPI Master)        │  <- Hardware-specific driver
└─────────────────────────────────────────┘
```

### Key Concepts
- **SPI Controller/Master**: Hardware SPI controller (manages the bus)
- **SPI Device/Slave**: Peripheral device on SPI bus (sensor, ADC, display, etc.)
- **SPI Transfer**: Single data exchange operation (tx_buf, rx_buf, len)
- **SPI Message**: Atomic sequence of transfers (queued for execution)
- **4-Wire Interface**: SCK (clock), MOSI (master out), MISO (master in), CS (chip select)
- **Modes**: 4 clock modes combining CPOL (polarity) and CPHA (phase)

### SPI Signal Lines
```
SCK  (Serial Clock)       - Clock signal from master
MOSI (Master Out Slave In) - Data from master to slave
MISO (Master In Slave Out) - Data from slave to master
CS/SS (Chip Select)       - Device selection (active low)
```

### SPI Clock Modes
```c
// Mode = CPOL | CPHA
Mode 0: CPOL=0, CPHA=0  // Clock idle low, sample on rising edge
Mode 1: CPOL=0, CPHA=1  // Clock idle low, sample on falling edge
Mode 2: CPOL=1, CPHA=0  // Clock idle high, sample on falling edge
Mode 3: CPOL=1, CPHA=1  // Clock idle high, sample on rising edge
```

## Essential Headers

```c
// For SPI protocol drivers (using SPI)
#include <linux/spi/spi.h>

// For SPI controller drivers
#include <linux/spi/spi.h>

// For device tree
#include <linux/of.h>
#include <linux/of_device.h>
```

## SPI Protocol Driver (Device Using SPI)

### SPI Driver Structure

```c
#include <linux/spi/spi.h>
#include <linux/module.h>

struct mydev_data {
    struct spi_device *spi;
    struct mutex lock;
    u8 tx_buf[32];
    u8 rx_buf[32];
};

// Device ID table
static const struct spi_device_id mydev_spi_id[] = {
    { "mydevice", 0 },
    { }
};
MODULE_DEVICE_TABLE(spi, mydev_spi_id);

// Device tree match table
static const struct of_device_id mydev_of_match[] = {
    { .compatible = "vendor,mydevice", },
    { }
};
MODULE_DEVICE_TABLE(of, mydev_of_match);

// Probe function
static int mydev_probe(struct spi_device *spi)
{
    struct mydev_data *data;
    int ret;
    
    dev_info(&spi->dev, "Probing SPI device (CS: %d, Mode: %d)\n",
             spi_get_chipselect(spi, 0), spi->mode);
    
    // Verify SPI mode
    spi->mode = SPI_MODE_0;
    spi->bits_per_word = 8;
    spi->max_speed_hz = 1000000;  // 1 MHz
    
    ret = spi_setup(spi);
    if (ret < 0) {
        dev_err(&spi->dev, "SPI setup failed\n");
        return ret;
    }
    
    // Allocate private data
    data = devm_kzalloc(&spi->dev, sizeof(*data), GFP_KERNEL);
    if (!data)
        return -ENOMEM;
    
    data->spi = spi;
    mutex_init(&data->lock);
    
    spi_set_drvdata(spi, data);
    
    dev_info(&spi->dev, "Device initialized successfully\n");
    
    return 0;
}

// Remove function
static void mydev_remove(struct spi_device *spi)
{
    struct mydev_data *data = spi_get_drvdata(spi);
    
    dev_info(&spi->dev, "Removing device\n");
    mutex_destroy(&data->lock);
}

// SPI driver structure
static struct spi_driver mydev_driver = {
    .driver = {
        .name = "mydevice",
        .of_match_table = mydev_of_match,
    },
    .id_table = mydev_spi_id,
    .probe = mydev_probe,
    .remove = mydev_remove,
};

module_spi_driver(mydev_driver);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Your Name");
MODULE_DESCRIPTION("My SPI device driver");
```

### SPI Data Transfer - Simple API

```c
// Simple write
int mydev_write(struct spi_device *spi, const u8 *buf, size_t len)
{
    return spi_write(spi, buf, len);
}

// Simple read
int mydev_read(struct spi_device *spi, u8 *buf, size_t len)
{
    return spi_read(spi, buf, len);
}

// Write then read (common pattern)
int mydev_write_then_read(struct spi_device *spi,
                          const u8 *txbuf, unsigned n_tx,
                          u8 *rxbuf, unsigned n_rx)
{
    return spi_write_then_read(spi, txbuf, n_tx, rxbuf, n_rx);
}

// Example: Read register
int mydev_read_reg(struct spi_device *spi, u8 reg, u8 *val)
{
    u8 tx_buf[1] = { reg | 0x80 };  // Set read bit
    u8 rx_buf[1];
    int ret;
    
    ret = spi_write_then_read(spi, tx_buf, 1, rx_buf, 1);
    if (ret == 0)
        *val = rx_buf[0];
    
    return ret;
}

// Write register
int mydev_write_reg(struct spi_device *spi, u8 reg, u8 val)
{
    u8 tx_buf[2] = { reg & 0x7F, val };  // Clear read bit
    
    return spi_write(spi, tx_buf, sizeof(tx_buf));
}
```

### SPI Transfer - Advanced API with spi_message

```c
// Full-duplex transfer with custom message
int mydev_transfer(struct spi_device *spi, 
                   const u8 *tx_buf, u8 *rx_buf, size_t len)
{
    struct spi_transfer t = {
        .tx_buf = tx_buf,
        .rx_buf = rx_buf,
        .len = len,
        .speed_hz = 1000000,      // Override speed for this transfer
        .bits_per_word = 8,
    };
    struct spi_message m;
    int ret;
    
    spi_message_init(&m);
    spi_message_add_tail(&t, &m);
    
    ret = spi_sync(spi, &m);
    if (ret)
        dev_err(&spi->dev, "SPI transfer failed: %d\n", ret);
    
    return ret;
}

// Multiple transfers in one message (atomic)
int mydev_multi_transfer(struct spi_device *spi)
{
    u8 cmd = 0x03;  // Read command
    u8 addr[2] = { 0x00, 0x10 };  // Address
    u8 data[16];
    
    struct spi_transfer xfers[] = {
        {
            .tx_buf = &cmd,
            .len = 1,
        },
        {
            .tx_buf = addr,
            .len = 2,
        },
        {
            .rx_buf = data,
            .len = sizeof(data),
        },
    };
    struct spi_message m;
    
    spi_message_init(&m);
    spi_message_add_tail(&xfers[0], &m);
    spi_message_add_tail(&xfers[1], &m);
    spi_message_add_tail(&xfers[2], &m);
    
    return spi_sync(spi, &m);
}
```

### Asynchronous SPI Transfer

```c
static void mydev_complete(void *arg)
{
    struct mydev_data *data = arg;
    
    dev_info(&data->spi->dev, "Async transfer complete\n");
    complete(&data->done);
}

int mydev_async_transfer(struct mydev_data *data)
{
    struct spi_message m;
    struct spi_transfer t = {
        .tx_buf = data->tx_buf,
        .rx_buf = data->rx_buf,
        .len = 16,
    };
    
    init_completion(&data->done);
    
    spi_message_init(&m);
    spi_message_add_tail(&t, &m);
    
    m.complete = mydev_complete;
    m.context = data;
    
    // Submit async (non-blocking)
    spi_async(data->spi, &m);
    
    // Wait for completion (or do other work)
    wait_for_completion(&data->done);
    
    return m.status;
}
```

## Device Tree Bindings

### SPI Controller Node

```dts
// SPI controller
spi0: spi@48030000 {
    compatible = "ti,omap4-mcspi";
    reg = <0x48030000 0x200>;
    interrupts = <GIC_SPI 65 IRQ_TYPE_LEVEL_HIGH>;
    #address-cells = <1>;
    #size-cells = <0>;
    ti,hwmods = "mcspi1";
    ti,spi-num-cs = <4>;
    dmas = <&edma 42 0>, <&edma 43 0>;
    dma-names = "tx0", "rx0";
};

// Generic SPI controller
&spi1 {
    status = "okay";
    pinctrl-names = "default";
    pinctrl-0 = <&spi1_pins>;
    
    // SPI devices on this controller...
};
```

### SPI Device Node

```dts
&spi0 {
    status = "okay";
    
    // SPI device with basic config
    spidev@0 {
        compatible = "vendor,mydevice";
        reg = <0>;                      // Chip select 0
        spi-max-frequency = <1000000>;  // 1 MHz
        spi-cpha;                       // Clock phase = 1
        spi-cpol;                       // Clock polarity = 1
        // Mode = (cpol << 1) | cpha = Mode 3
    };
    
    // SPI device with additional properties
    adc@1 {
        compatible = "ti,adc128s022";
        reg = <1>;                      // Chip select 1
        spi-max-frequency = <500000>;   // 500 kHz
        vref-supply = <&vdd_3v3>;
    };
    
    // SPI display
    display@2 {
        compatible = "ilitek,ili9341";
        reg = <2>;
        spi-max-frequency = <10000000>;  // 10 MHz
        dc-gpios = <&gpio1 10 GPIO_ACTIVE_HIGH>;
        reset-gpios = <&gpio1 11 GPIO_ACTIVE_LOW>;
        rotation = <90>;
    };
};
```

### Parsing Device Tree

```c
static int mydev_probe(struct spi_device *spi)
{
    struct device *dev = &spi->dev;
    u32 value;
    
    // SPI parameters already parsed by core
    dev_info(dev, "Max speed: %u Hz\n", spi->max_speed_hz);
    dev_info(dev, "Mode: %d\n", spi->mode);
    dev_info(dev, "Bits per word: %d\n", spi->bits_per_word);
    
    // Read custom properties
    if (of_property_read_u32(dev->of_node, "rotation", &value) == 0)
        dev_info(dev, "Rotation: %u\n", value);
    
    // Get GPIO
    struct gpio_desc *dc_gpio;
    dc_gpio = devm_gpiod_get(dev, "dc", GPIOD_OUT_LOW);
    if (IS_ERR(dc_gpio))
        return PTR_ERR(dc_gpio);
    
    return 0;
}
```

## SPI Controller Driver Implementation

### SPI Controller Operations

```c
struct spi_controller_ops {
    // Main transfer function
    int (*transfer_one_message)(struct spi_controller *ctlr,
                                struct spi_message *mesg);
    
    // Or implement per-transfer
    int (*transfer_one)(struct spi_controller *ctlr,
                       struct spi_device *spi,
                       struct spi_transfer *transfer);
    
    // Setup/cleanup
    int (*setup)(struct spi_device *spi);
    void (*cleanup)(struct spi_device *spi);
    
    // Chipselect control
    void (*set_cs)(struct spi_device *spi, bool enable);
    
    // Prepare/unprepare hardware
    int (*prepare_transfer_hardware)(struct spi_controller *ctlr);
    int (*unprepare_transfer_hardware)(struct spi_controller *ctlr);
};
```

### Basic SPI Controller Driver

```c
#include <linux/spi/spi.h>
#include <linux/platform_device.h>
#include <linux/clk.h>
#include <linux/io.h>

struct my_spi_controller {
    void __iomem *base;
    struct clk *clk;
    struct completion done;
    const u8 *tx_buf;
    u8 *rx_buf;
    unsigned int len;
};

static void my_spi_set_cs(struct spi_device *spi, bool enable)
{
    struct spi_controller *ctlr = spi->controller;
    struct my_spi_controller *msc = spi_controller_get_devdata(ctlr);
    u32 val;
    
    val = readl(msc->base + CTRL_REG);
    if (enable)
        val |= CS_ACTIVE;
    else
        val &= ~CS_ACTIVE;
    writel(val, msc->base + CTRL_REG);
}

static int my_spi_transfer_one(struct spi_controller *ctlr,
                               struct spi_device *spi,
                               struct spi_transfer *t)
{
    struct my_spi_controller *msc = spi_controller_get_devdata(ctlr);
    u32 ctrl;
    
    // Setup transfer parameters
    msc->tx_buf = t->tx_buf;
    msc->rx_buf = t->rx_buf;
    msc->len = t->len;
    
    // Configure word size
    ctrl = readl(msc->base + CTRL_REG);
    ctrl &= ~WORD_SIZE_MASK;
    ctrl |= WORD_SIZE(t->bits_per_word);
    
    // Configure clock
    u32 div = DIV_ROUND_UP(msc->clk_rate, t->speed_hz);
    ctrl &= ~CLK_DIV_MASK;
    ctrl |= CLK_DIV(div);
    
    writel(ctrl, msc->base + CTRL_REG);
    
    // Start transfer
    writel(START_TRANSFER, msc->base + CMD_REG);
    
    // Wait for completion
    wait_for_completion(&msc->done);
    
    return 0;
}

static irqreturn_t my_spi_irq(int irq, void *dev_id)
{
    struct spi_controller *ctlr = dev_id;
    struct my_spi_controller *msc = spi_controller_get_devdata(ctlr);
    u32 status;
    
    status = readl(msc->base + STATUS_REG);
    
    // Handle TX/RX
    if (status & TX_READY && msc->tx_buf && msc->len) {
        writel(*msc->tx_buf++, msc->base + TX_DATA);
        msc->len--;
    }
    
    if (status & RX_READY && msc->rx_buf) {
        *msc->rx_buf++ = readl(msc->base + RX_DATA);
    }
    
    // Transfer complete
    if (status & TRANSFER_DONE) {
        complete(&msc->done);
    }
    
    // Clear interrupts
    writel(status, msc->base + STATUS_REG);
    
    return IRQ_HANDLED;
}

static int my_spi_probe(struct platform_device *pdev)
{
    struct spi_controller *ctlr;
    struct my_spi_controller *msc;
    struct resource *res;
    int ret, irq;
    
    // Allocate SPI controller
    ctlr = devm_spi_alloc_master(&pdev->dev, sizeof(*msc));
    if (!ctlr)
        return -ENOMEM;
    
    msc = spi_controller_get_devdata(ctlr);
    
    // Map registers
    res = platform_get_resource(pdev, IORESOURCE_MEM, 0);
    msc->base = devm_ioremap_resource(&pdev->dev, res);
    if (IS_ERR(msc->base))
        return PTR_ERR(msc->base);
    
    // Get clock
    msc->clk = devm_clk_get(&pdev->dev, "spi");
    if (IS_ERR(msc->clk))
        return PTR_ERR(msc->clk);
    
    ret = clk_prepare_enable(msc->clk);
    if (ret)
        return ret;
    
    // Get IRQ
    irq = platform_get_irq(pdev, 0);
    if (irq < 0) {
        ret = irq;
        goto err_clk;
    }
    
    ret = devm_request_irq(&pdev->dev, irq, my_spi_irq,
                           0, dev_name(&pdev->dev), ctlr);
    if (ret)
        goto err_clk;
    
    init_completion(&msc->done);
    
    // Setup controller
    ctlr->mode_bits = SPI_CPOL | SPI_CPHA | SPI_CS_HIGH | SPI_LSB_FIRST;
    ctlr->bits_per_word_mask = SPI_BPW_RANGE_MASK(4, 32);
    ctlr->num_chipselect = 4;
    ctlr->bus_num = pdev->id;
    ctlr->dev.of_node = pdev->dev.of_node;
    
    ctlr->transfer_one = my_spi_transfer_one;
    ctlr->set_cs = my_spi_set_cs;
    
    // Register controller
    ret = devm_spi_register_controller(&pdev->dev, ctlr);
    if (ret) {
        dev_err(&pdev->dev, "Failed to register SPI controller\n");
        goto err_clk;
    }
    
    platform_set_drvdata(pdev, ctlr);
    
    dev_info(&pdev->dev, "SPI controller registered\n");
    
    return 0;

err_clk:
    clk_disable_unprepare(msc->clk);
    return ret;
}

static void my_spi_remove(struct platform_device *pdev)
{
    struct spi_controller *ctlr = platform_get_drvdata(pdev);
    struct my_spi_controller *msc = spi_controller_get_devdata(ctlr);
    
    clk_disable_unprepare(msc->clk);
}

static const struct of_device_id my_spi_of_match[] = {
    { .compatible = "vendor,my-spi", },
    { }
};
MODULE_DEVICE_TABLE(of, my_spi_of_match);

static struct platform_driver my_spi_driver = {
    .probe = my_spi_probe,
    .remove = my_spi_remove,
    .driver = {
        .name = "my-spi",
        .of_match_table = my_spi_of_match,
    },
};
module_platform_driver(my_spi_driver);
```

## Common SPI Patterns

### Reading Sensor Data

```c
// Example: Read temperature from ADC
int read_temperature(struct spi_device *spi)
{
    u8 tx_buf[2] = { 0x01, 0x80 };  // Start conversion, channel 0
    u8 rx_buf[2];
    int ret, value;
    
    ret = spi_write_then_read(spi, tx_buf, 2, rx_buf, 2);
    if (ret < 0)
        return ret;
    
    // Convert 12-bit ADC value
    value = ((rx_buf[0] & 0x0F) << 8) | rx_buf[1];
    
    return value;
}
```

### SPI Flash Operations

```c
// Read from SPI flash
int spi_flash_read(struct spi_device *spi, u32 addr, u8 *buf, size_t len)
{
    u8 cmd[4] = {
        0x03,                    // Read command
        (addr >> 16) & 0xFF,     // Address MSB
        (addr >> 8) & 0xFF,
        addr & 0xFF,             // Address LSB
    };
    
    return spi_write_then_read(spi, cmd, 4, buf, len);
}

// Write to SPI flash
int spi_flash_write(struct spi_device *spi, u32 addr, const u8 *buf, size_t len)
{
    u8 cmd[4 + 256];  // Command + address + data (max 256 bytes)
    
    if (len > 256)
        return -EINVAL;
    
    cmd[0] = 0x02;  // Page program command
    cmd[1] = (addr >> 16) & 0xFF;
    cmd[2] = (addr >> 8) & 0xFF;
    cmd[3] = addr & 0xFF;
    memcpy(&cmd[4], buf, len);
    
    return spi_write(spi, cmd, 4 + len);
}
```

### DMA Transfer

```c
int mydev_dma_transfer(struct spi_device *spi, void *tx, void *rx, size_t len)
{
    struct spi_transfer t = {
        .tx_buf = tx,
        .rx_buf = rx,
        .len = len,
        .tx_dma = 0,  // Will be filled by SPI core
        .rx_dma = 0,
    };
    struct spi_message m;
    
    spi_message_init(&m);
    spi_message_add_tail(&t, &m);
    
    return spi_sync(spi, &m);
}
```

## SPI Mode Configuration

```c
// Configure SPI mode in probe
static int mydev_probe(struct spi_device *spi)
{
    // Mode 0: CPOL=0, CPHA=0
    spi->mode = SPI_MODE_0;
    
    // Or specific flags
    spi->mode = SPI_CPOL | SPI_CPHA;  // Mode 3
    
    // Additional flags
    spi->mode |= SPI_CS_HIGH;         // Active high CS
    spi->mode |= SPI_LSB_FIRST;       // LSB first
    spi->mode |= SPI_3WIRE;           // 3-wire mode
    spi->mode |= SPI_NO_CS;           // No CS signal
    
    spi->bits_per_word = 8;
    spi->max_speed_hz = 1000000;
    
    return spi_setup(spi);
}
```

## Debugging SPI

### sysfs Interface

```bash
# List SPI controllers
ls /sys/class/spi_master/

# List SPI devices
ls /sys/bus/spi/devices/

# View device details
cat /sys/bus/spi/devices/spi0.0/modalias
cat /sys/bus/spi/devices/spi0.0/statistics/transfers
```

### Kernel Debugging

```c
// Enable dynamic debug
echo 'file drivers/spi/* +p' > /sys/kernel/debug/dynamic_debug/control

// Trace SPI transfers
echo 'file spi.c +p' > /sys/kernel/debug/dynamic_debug/control
```

### Using spidev for Testing

```bash
# Install spi-tools
apt-get install spi-tools

# Test SPI device
spi-config -d /dev/spidev0.0 -q
spi-pipe -d /dev/spidev0.0 -s 1000000 < data.bin > result.bin
```

## Best Practices

✅ **DO:**
- Use `devm_*` functions for automatic cleanup
- Call `spi_setup()` after changing SPI parameters
- Use `spi_sync()` for synchronous transfers
- Use `spi_async()` for non-blocking transfers
- Check return values from all SPI operations
- Use appropriate SPI mode for your device
- Set correct `max_speed_hz` from device datasheet

❌ **DON'T:**
- Call SPI transfer functions from hard IRQ context (may sleep)
- Modify buffers during active transfer
- Share buffers between concurrent transfers
- Use stack buffers for DMA transfers (use kmalloc)
- Forget to initialize `spi_message` before use
- Mix sync and async operations on same message

## Common Errors & Solutions

```c
// Error: -EINVAL
// Cause: Invalid speed_hz or bits_per_word
// Solution: Check supported values in controller driver

// Error: -EOPNOTSUPP
// Cause: SPI mode not supported
// Solution: Check controller mode_bits capability

// Error: -EBUSY
// Cause: Controller busy with another transfer
// Solution: Retry or use queue properly

// Error: -EAGAIN
// Cause: Transfer temporarily unavailable
// Solution: Retry the operation
```

## SPI Transfer Flags

```c
// Transfer flags (in struct spi_transfer)
.cs_change = 1;        // Toggle CS after this transfer
.delay.value = 100;    // Delay after transfer (microseconds)
.delay.unit = SPI_DELAY_UNIT_USECS;

// Bits per transfer
.tx_nbits = SPI_NBITS_SINGLE;  // 1-bit (standard)
.tx_nbits = SPI_NBITS_DUAL;    // 2-bit (dual SPI)
.tx_nbits = SPI_NBITS_QUAD;    // 4-bit (quad SPI)
```
