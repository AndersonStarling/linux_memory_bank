# IIO (Industrial I/O) Subsystem

## IIO Architecture Overview

IIO subsystem cung cấp framework chuẩn cho devices như ADC, DAC, accelerometers, gyroscopes và sensors khác. BeagleBone có 7-channel 12-bit ADC onboard.

```
┌──────────────────────────────────────────┐
│   Userspace (sysfs/char device)         │  <- /sys/bus/iio/devices/iio:deviceX
├──────────────────────────────────────────┤
│   IIO Core                                │  <- Channel management, buffering
├──────────────────────────────────────────┤
│   IIO Device Driver (ADC, Sensor, etc.) │  <- Hardware-specific implementation
└──────────────────────────────────────────┘
```

### Key Concepts
- **IIO Device**: Sensor/converter device
- **IIO Channel**: Individual measurement channel (voltage, current, temperature)
- **IIO Buffer**: Continuous data capture buffer
- **IIO Trigger**: Event that initiates data capture
- **Raw/Processed Values**: Raw ADC counts vs converted real-world values

## Essential Headers

```c
#include <linux/iio/iio.h>
#include <linux/iio/sysfs.h>
#include <linux/iio/buffer.h>
#include <linux/iio/trigger.h>
#include <linux/iio/triggered_buffer.h>
```

## IIO Channel Types

```c
// Common channel types (include/uapi/linux/iio/types.h)
enum iio_chan_type {
    IIO_VOLTAGE,        // Voltage measurement
    IIO_CURRENT,        // Current measurement
    IIO_POWER,          // Power measurement
    IIO_ACCEL,          // Accelerometer
    IIO_ANGL_VEL,       // Gyroscope (angular velocity)
    IIO_MAGN,           // Magnetometer
    IIO_LIGHT,          // Light sensor
    IIO_INTENSITY,      // Light intensity
    IIO_PROXIMITY,      // Proximity sensor
    IIO_TEMP,           // Temperature
    IIO_INCLI,          // Inclinometer
    IIO_ROT,            // Rotation
    IIO_ANGL,           // Angle
    IIO_TIMESTAMP,      // Timestamp
    IIO_CAPACITANCE,    // Capacitance
    IIO_PRESSURE,       // Pressure
    IIO_HUMIDITYRELATIVE, // Humidity
    // ... more types
};
```

## IIO Device Driver Implementation

### Basic ADC Driver

```c
#include <linux/iio/iio.h>
#include <linux/module.h>
#include <linux/platform_device.h>

#define MY_ADC_CHANNELS 8
#define MY_ADC_RESOLUTION 12

struct my_adc {
    void __iomem *base;
    struct clk *clk;
    struct mutex lock;
};

// Define channels
#define MY_ADC_CHANNEL(idx) {                           \
    .type = IIO_VOLTAGE,                                 \
    .indexed = 1,                                        \
    .channel = idx,                                      \
    .info_mask_separate = BIT(IIO_CHAN_INFO_RAW),       \
    .info_mask_shared_by_type = BIT(IIO_CHAN_INFO_SCALE), \
    .datasheet_name = "AIN"#idx,                        \
    .scan_index = idx,                                   \
    .scan_type = {                                       \
        .sign = 'u',                                     \
        .realbits = MY_ADC_RESOLUTION,                  \
        .storagebits = 16,                              \
        .shift = 0,                                      \
        .endianness = IIO_CPU,                          \
    },                                                   \
}

static const struct iio_chan_spec my_adc_channels[] = {
    MY_ADC_CHANNEL(0),
    MY_ADC_CHANNEL(1),
    MY_ADC_CHANNEL(2),
    MY_ADC_CHANNEL(3),
    MY_ADC_CHANNEL(4),
    MY_ADC_CHANNEL(5),
    MY_ADC_CHANNEL(6),
    MY_ADC_CHANNEL(7),
    IIO_CHAN_SOFT_TIMESTAMP(8),  // Timestamp channel for buffered mode
};

// Read raw ADC value
static int my_adc_read_raw(struct iio_dev *indio_dev,
                           struct iio_chan_spec const *chan,
                           int *val, int *val2, long mask)
{
    struct my_adc *adc = iio_priv(indio_dev);
    u32 raw;
    
    switch (mask) {
    case IIO_CHAN_INFO_RAW:
        mutex_lock(&adc->lock);
        
        // Start conversion
        writel(BIT(chan->channel), adc->base + CTRL_REG);
        
        // Wait for conversion (busy wait or use interrupt)
        while (!(readl(adc->base + STATUS_REG) & CONVERSION_DONE))
            cpu_relax();
        
        // Read result
        raw = readl(adc->base + DATA_REG + (chan->channel * 4));
        raw &= (1 << MY_ADC_RESOLUTION) - 1;  // Mask to 12 bits
        
        mutex_unlock(&adc->lock);
        
        *val = raw;
        return IIO_VAL_INT;
    
    case IIO_CHAN_INFO_SCALE:
        // Convert raw to millivolts
        // Scale = (Vref / 2^resolution) in mV
        *val = 1800;  // Vref = 1.8V
        *val2 = MY_ADC_RESOLUTION;
        return IIO_VAL_FRACTIONAL_LOG2;
    
    default:
        return -EINVAL;
    }
}

static const struct iio_info my_adc_info = {
    .read_raw = my_adc_read_raw,
};

static int my_adc_probe(struct platform_device *pdev)
{
    struct iio_dev *indio_dev;
    struct my_adc *adc;
    struct resource *res;
    int ret;
    
    // Allocate IIO device with private data
    indio_dev = devm_iio_device_alloc(&pdev->dev, sizeof(*adc));
    if (!indio_dev)
        return -ENOMEM;
    
    adc = iio_priv(indio_dev);
    
    // Map registers
    res = platform_get_resource(pdev, IORESOURCE_MEM, 0);
    adc->base = devm_ioremap_resource(&pdev->dev, res);
    if (IS_ERR(adc->base))
        return PTR_ERR(adc->base);
    
    // Get and enable clock
    adc->clk = devm_clk_get(&pdev->dev, "fck");
    if (IS_ERR(adc->clk))
        return PTR_ERR(adc->clk);
    
    ret = clk_prepare_enable(adc->clk);
    if (ret)
        return ret;
    
    mutex_init(&adc->lock);
    
    // Setup IIO device
    indio_dev->name = dev_name(&pdev->dev);
    indio_dev->dev.parent = &pdev->dev;
    indio_dev->info = &my_adc_info;
    indio_dev->modes = INDIO_DIRECT_MODE;
    indio_dev->channels = my_adc_channels;
    indio_dev->num_channels = ARRAY_SIZE(my_adc_channels);
    
    platform_set_drvdata(pdev, indio_dev);
    
    // Register IIO device
    ret = devm_iio_device_register(&pdev->dev, indio_dev);
    if (ret) {
        dev_err(&pdev->dev, "Failed to register IIO device\n");
        goto err_clk;
    }
    
    dev_info(&pdev->dev, "ADC registered with %d channels\n",
             indio_dev->num_channels);
    
    return 0;

err_clk:
    clk_disable_unprepare(adc->clk);
    return ret;
}

static int my_adc_remove(struct platform_device *pdev)
{
    struct iio_dev *indio_dev = platform_get_drvdata(pdev);
    struct my_adc *adc = iio_priv(indio_dev);
    
    clk_disable_unprepare(adc->clk);
    mutex_destroy(&adc->lock);
    
    return 0;
}

static const struct of_device_id my_adc_of_match[] = {
    { .compatible = "vendor,my-adc", },
    { }
};
MODULE_DEVICE_TABLE(of, my_adc_of_match);

static struct platform_driver my_adc_driver = {
    .probe = my_adc_probe,
    .remove = my_adc_remove,
    .driver = {
        .name = "my-adc",
        .of_match_table = my_adc_of_match,
    },
};
module_platform_driver(my_adc_driver);

MODULE_LICENSE("GPL");
MODULE_DESCRIPTION("My ADC IIO driver");
```

## Userspace Access

### sysfs Interface

```bash
# Find IIO devices
ls /sys/bus/iio/devices/

# Read ADC channel
cd /sys/bus/iio/devices/iio:device0

# Read raw value
cat in_voltage0_raw
# Output: 2048

# Read scale (to convert to real voltage)
cat in_voltage0_scale
# Output: 0.000439453125

# Calculate voltage: raw * scale
# 2048 * 0.000439453125 = 0.9V

# Read all channels
for ch in in_voltage*_raw; do
    echo "$ch: $(cat $ch)"
done

# Read device name
cat name
```

### IIO Buffered Mode

```c
// Enable buffered data capture
static int setup_buffer(struct iio_dev *indio_dev)
{
    int ret;
    
    ret = devm_iio_triggered_buffer_setup(&indio_dev->dev.parent,
                                           indio_dev,
                                           &iio_pollfunc_store_time,
                                           &my_adc_trigger_handler,
                                           NULL);
    if (ret) {
        dev_err(&indio_dev->dev, "Failed to setup buffer\n");
        return ret;
    }
    
    return 0;
}

// Trigger handler - called when data is ready
static irqreturn_t my_adc_trigger_handler(int irq, void *p)
{
    struct iio_poll_func *pf = p;
    struct iio_dev *indio_dev = pf->indio_dev;
    struct my_adc *adc = iio_priv(indio_dev);
    u16 data[MY_ADC_CHANNELS + sizeof(s64)/sizeof(u16)];
    int i, j = 0;
    
    // Read all enabled channels
    for_each_set_bit(i, indio_dev->active_scan_mask, MY_ADC_CHANNELS) {
        data[j++] = readl(adc->base + DATA_REG + (i * 4));
    }
    
    // Add timestamp
    iio_push_to_buffers_with_timestamp(indio_dev, data, pf->timestamp);
    
    iio_trigger_notify_done(indio_dev->trig);
    
    return IRQ_HANDLED;
}
```

### Reading Buffer from Userspace

```c
// C example to read IIO buffer
#include <stdio.h>
#include <fcntl.h>
#include <unistd.h>
#include <sys/ioctl.h>
#include <linux/iio/types.h>

int main(void)
{
    int fd;
    struct iio_event_data event;
    
    // Enable channels
    system("echo 1 > /sys/bus/iio/devices/iio:device0/scan_elements/in_voltage0_en");
    system("echo 1 > /sys/bus/iio/devices/iio:device0/scan_elements/in_voltage1_en");
    
    // Set buffer size
    system("echo 128 > /sys/bus/iio/devices/iio:device0/buffer/length");
    
    // Enable buffer
    system("echo 1 > /sys/bus/iio/devices/iio:device0/buffer/enable");
    
    // Open device
    fd = open("/dev/iio:device0", O_RDONLY);
    if (fd < 0) {
        perror("open");
        return 1;
    }
    
    // Read samples
    uint16_t data[2];
    while (1) {
        read(fd, data, sizeof(data));
        printf("CH0: %u, CH1: %u\n", data[0], data[1]);
    }
    
    close(fd);
    return 0;
}
```

## Device Tree Bindings

```dts
adc: adc@44e0d000 {
    compatible = "ti,am3359-adc";
    reg = <0x44e0d000 0x1000>;
    interrupts = <16>;
    clocks = <&adc_tsc_fck>;
    clock-names = "fck";
    #io-channel-cells = <1>;
    
    // ADC reference voltage
    vref-supply = <&vdd_adc>;
};

// Consumer using ADC channel
touchscreen@0 {
    compatible = "ti,am3359-tsc";
    
    // Use ADC channel 0
    io-channels = <&adc 0>;
    io-channel-names = "pressure";
};
```

## In-Kernel Consumer API

```c
// Driver consuming IIO channels
#include <linux/iio/consumer.h>

struct my_consumer {
    struct iio_channel *adc_ch;
};

static int my_consumer_probe(struct platform_device *pdev)
{
    struct my_consumer *cons;
    int ret, val;
    
    cons = devm_kzalloc(&pdev->dev, sizeof(*cons), GFP_KERNEL);
    if (!cons)
        return -ENOMEM;
    
    // Get IIO channel from device tree
    cons->adc_ch = devm_iio_channel_get(&pdev->dev, "pressure");
    if (IS_ERR(cons->adc_ch))
        return PTR_ERR(cons->adc_ch);
    
    // Read raw value
    ret = iio_read_channel_raw(cons->adc_ch, &val);
    if (ret < 0)
        return ret;
    
    dev_info(&pdev->dev, "ADC raw value: %d\n", val);
    
    // Read processed value (with scale applied)
    ret = iio_read_channel_processed(cons->adc_ch, &val);
    if (ret < 0)
        return ret;
    
    dev_info(&pdev->dev, "ADC processed value: %d mV\n", val);
    
    return 0;
}
```

## Common IIO Sensors

### Accelerometer Example

```c
static const struct iio_chan_spec accel_channels[] = {
    {
        .type = IIO_ACCEL,
        .modified = 1,
        .channel2 = IIO_MOD_X,  // X-axis
        .info_mask_separate = BIT(IIO_CHAN_INFO_RAW),
        .info_mask_shared_by_type = BIT(IIO_CHAN_INFO_SCALE),
    },
    {
        .type = IIO_ACCEL,
        .modified = 1,
        .channel2 = IIO_MOD_Y,  // Y-axis
        .info_mask_separate = BIT(IIO_CHAN_INFO_RAW),
        .info_mask_shared_by_type = BIT(IIO_CHAN_INFO_SCALE),
    },
    {
        .type = IIO_ACCEL,
        .modified = 1,
        .channel2 = IIO_MOD_Z,  // Z-axis
        .info_mask_separate = BIT(IIO_CHAN_INFO_RAW),
        .info_mask_shared_by_type = BIT(IIO_CHAN_INFO_SCALE),
    },
};
```

## Best Practices

✅ **DO:**
- Use `devm_iio_device_alloc()` for cleanup
- Implement proper locking in read_raw
- Provide scale for unit conversion
- Use buffered mode for high-speed sampling
- Document channel mappings

❌ **DON'T:**
- Block in read_raw callback
- Return raw values without scale info
- Forget to handle concurrent access
- Mix up channel indices
- Ignore buffer overruns

## Debugging

```bash
# List IIO devices
iio_info

# Read all channels
iio_attr -C iio:device0

# Monitor continuous data
cat /dev/iio:device0

# Kernel messages
dmesg | grep -i iio
```
