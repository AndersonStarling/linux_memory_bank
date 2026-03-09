# IRQ (Interrupt) Subsystem

## IRQ Architecture Overview

Interrupt subsystem quản lý tất cả interrupts trong hệ thống, từ hardware IRQs đến software IRQs, cho phép devices phản ứng ngay lập tức với events.

```
┌──────────────────────────────────────────┐
│   Device Driver                          │  <- request_irq(), handler
├──────────────────────────────────────────┤
│   IRQ Core (kernel/irq/)                 │  <- IRQ management, flow control
├──────────────────────────────────────────┤
│   IRQ Controller Driver (irqchip)       │  <- Hardware interrupt controller
└──────────────────────────────────────────┘
```

### Key Concepts
- **IRQ Number**: Virtual interrupt number
- **Hardware IRQ (hwirq)**: Physical interrupt line
- **IRQ Handler**: Function called when interrupt occurs
- **IRQ Flags**: Trigger type, sharing, threading
- **Threaded IRQ**: Handler runs in thread context (can sleep)
- **Top Half**: Quick IRQ handler (atomic context)
- **Bottom Half**: Deferred work (softirq, tasklet, workqueue)

## Essential Headers

```c
#include <linux/interrupt.h>
#include <linux/irq.h>
```

## IRQ Request and Handling

### Basic IRQ Handler

```c
#include <linux/interrupt.h>

static irqreturn_t mydev_irq_handler(int irq, void *dev_id)
{
    struct mydev_data *data = dev_id;
    u32 status;
    
    // Read interrupt status register
    status = readl(data->base + INT_STATUS_REG);
    
    // Check if this is our interrupt
    if (!(status & MY_IRQ_MASK))
        return IRQ_NONE;  // Not our interrupt
    
    // Clear interrupt
    writel(status & MY_IRQ_MASK, data->base + INT_CLEAR_REG);
    
    // Handle interrupt
    if (status & DATA_READY_BIT) {
        // Read data
        data->rx_data = readl(data->base + DATA_REG);
        wake_up_interruptible(&data->wait_queue);
    }
    
    if (status & ERROR_BIT) {
        dev_err(data->dev, "Hardware error\n");
        data->error_count++;
    }
    
    return IRQ_HANDLED;
}

static int mydev_probe(struct platform_device *pdev)
{
    struct mydev_data *data;
    int irq, ret;
    
    // Get IRQ number from platform resources
    irq = platform_get_irq(pdev, 0);
    if (irq < 0) {
        dev_err(&pdev->dev, "No IRQ resource\n");
        return irq;
    }
    
    // Request IRQ
    ret = devm_request_irq(&pdev->dev, irq, mydev_irq_handler,
                           IRQF_TRIGGER_HIGH, dev_name(&pdev->dev), data);
    if (ret) {
        dev_err(&pdev->dev, "Failed to request IRQ %d: %d\n", irq, ret);
        return ret;
    }
    
    dev_info(&pdev->dev, "IRQ %d registered\n", irq);
    
    return 0;
}
```

### IRQ Flags

```c
// Trigger types
IRQF_TRIGGER_NONE          // No specific trigger type
IRQF_TRIGGER_RISING        // Rising edge triggered
IRQF_TRIGGER_FALLING       // Falling edge triggered
IRQF_TRIGGER_HIGH          // High level triggered
IRQF_TRIGGER_LOW           // Low level triggered

// Sharing and threading
IRQF_SHARED                // Shared IRQ line
IRQF_ONESHOT               // Keep IRQ disabled until threaded handler finishes
IRQF_NO_SUSPEND            // Don't disable during suspend
IRQF_NO_THREAD             // Don't thread this IRQ
IRQF_EARLY_RESUME          // Resume IRQ early

// Common combinations
IRQF_TRIGGER_RISING | IRQF_ONESHOT  // Threaded IRQ, rising edge
IRQF_SHARED | IRQF_TRIGGER_HIGH     // Shared, level-triggered
```

### Threaded IRQ Handler

```c
// Fast handler (runs in hard IRQ context)
static irqreturn_t mydev_irq_quick(int irq, void *dev_id)
{
    struct mydev_data *data = dev_id;
    
    // Quick check and acknowledge
    if (!is_my_interrupt(data))
        return IRQ_NONE;
    
    // Clear interrupt at hardware level
    clear_hw_interrupt(data);
    
    // Wake up threaded handler
    return IRQ_WAKE_THREAD;
}

// Threaded handler (runs in process context, can sleep)
static irqreturn_t mydev_irq_thread(int irq, void *dev_id)
{
    struct mydev_data *data = dev_id;
    
    // Can perform lengthy operations here
    // Can call functions that sleep
    // Can use mutex
    
    mutex_lock(&data->lock);
    
    // Read data from device (may involve I2C/SPI)
    read_sensor_data(data);
    
    // Process data
    process_data(data);
    
    mutex_unlock(&data->lock);
    
    return IRQ_HANDLED;
}

static int mydev_probe(struct platform_device *pdev)
{
    int ret;
    
    // Request threaded IRQ
    ret = devm_request_threaded_irq(&pdev->dev, irq,
                                     mydev_irq_quick,
                                     mydev_irq_thread,
                                     IRQF_TRIGGER_RISING | IRQF_ONESHOT,
                                     dev_name(&pdev->dev),
                                     data);
    return ret;
}
```

## Deferred Work (Bottom Halves)

### Softirq (Very Fast, Atomic)

```c
// Predefined softirqs (kernel/softirq.c)
// Cannot create new softirqs in drivers!

// Common softirqs:
NET_TX_SOFTIRQ          // Network transmit
NET_RX_SOFTIRQ          // Network receive
BLOCK_SOFTIRQ           // Block devices
IRQ_POLL_SOFTIRQ        // IRQ polling
TASKLET_SOFTIRQ         // Tasklets
SCHED_SOFTIRQ           // Scheduler
HRTIMER_SOFTIRQ         // High-resolution timers
RCU_SOFTIRQ             // RCU
```

### Tasklet (Atomic, Built on Softirq)

```c
#include <linux/interrupt.h>

struct mydev_data {
    struct tasklet_struct tasklet;
    void __iomem *base;
};

static void mydev_tasklet_fn(unsigned long data)
{
    struct mydev_data *mydata = (struct mydev_data *)data;
    
    // Process interrupt in tasklet context
    // Still atomic, cannot sleep
    // But not time-critical
    
    process_deferred_work(mydata);
}

static irqreturn_t mydev_irq_handler(int irq, void *dev_id)
{
    struct mydev_data *data = dev_id;
    
    // Quick handling
    acknowledge_interrupt(data);
    
    // Schedule tasklet for deferred work
    tasklet_schedule(&data->tasklet);
    
    return IRQ_HANDLED;
}

static int mydev_probe(struct platform_device *pdev)
{
    struct mydev_data *data;
    
    data = devm_kzalloc(&pdev->dev, sizeof(*data), GFP_KERNEL);
    
    // Initialize tasklet
    tasklet_init(&data->tasklet, mydev_tasklet_fn, (unsigned long)data);
    
    // Request IRQ
    devm_request_irq(&pdev->dev, irq, mydev_irq_handler, 0,
                     dev_name(&pdev->dev), data);
    
    return 0;
}

static int mydev_remove(struct platform_device *pdev)
{
    struct mydev_data *data = platform_get_drvdata(pdev);
    
    // Kill tasklet
    tasklet_kill(&data->tasklet);
    
    return 0;
}
```

### Workqueue (Can Sleep)

```c
#include <linux/workqueue.h>

struct mydev_data {
    struct work_struct work;
    struct delayed_work delayed_work;
    struct workqueue_struct *wq;
};

static void mydev_work_fn(struct work_struct *work)
{
    struct mydev_data *data = container_of(work, struct mydev_data, work);
    
    // Can sleep here
    // Can use mutex
    // Can do I2C/SPI transfers
    
    mutex_lock(&data->lock);
    read_device_slow(data);
    mutex_unlock(&data->lock);
}

static irqreturn_t mydev_irq_handler(int irq, void *dev_id)
{
    struct mydev_data *data = dev_id;
    
    // Quick handling
    acknowledge_interrupt(data);
    
    // Schedule work
    schedule_work(&data->work);
    
    // Or schedule on custom workqueue
    queue_work(data->wq, &data->work);
    
    return IRQ_HANDLED;
}

static int mydev_probe(struct platform_device *pdev)
{
    struct mydev_data *data;
    
    data = devm_kzalloc(&pdev->dev, sizeof(*data), GFP_KERNEL);
    
    // Initialize work
    INIT_WORK(&data->work, mydev_work_fn);
    
    // Create custom workqueue (optional)
    data->wq = create_singlethread_workqueue("mydev_wq");
    if (!data->wq)
        return -ENOMEM;
    
    // Request IRQ
    devm_request_irq(&pdev->dev, irq, mydev_irq_handler, 0,
                     dev_name(&pdev->dev), data);
    
    return 0;
}

static int mydev_remove(struct platform_device *pdev)
{
    struct mydev_data *data = platform_get_drvdata(pdev);
    
    // Cancel work
    cancel_work_sync(&data->work);
    
    // Destroy workqueue
    if (data->wq)
        destroy_workqueue(data->wq);
    
    return 0;
}
```

## IRQ Control Functions

```c
// Disable specific IRQ
disable_irq(irq);               // Waits for handler to complete
disable_irq_nosync(irq);        // Doesn't wait

// Enable specific IRQ
enable_irq(irq);

// Disable all IRQs on current CPU
unsigned long flags;
local_irq_save(flags);
// Critical section
local_irq_restore(flags);

// Just disable (no save/restore)
local_irq_disable();
local_irq_enable();

// Check if IRQs are disabled
if (irqs_disabled())
    pr_warn("IRQs are disabled\n");

// Synchronize with IRQ
synchronize_irq(irq);  // Wait for IRQ handler to complete
```

## Device Tree Bindings

### Interrupt Consumer

```dts
&uart0 {
    interrupts = <72>;              // Simple IRQ number
};

&gpio1 {
    interrupt-parent = <&intc>;     // Interrupt controller
    interrupts = <29>;
    interrupt-controller;            // This device can generate IRQs
    #interrupt-cells = <2>;
};

&spi0 {
    interrupt-parent = <&gpio1>;
    interrupts = <7 IRQ_TYPE_EDGE_FALLING>;
    //           ^  ^
    //           |  +-- IRQ flags (trigger type)
    //           +-- GPIO line number
};

&touchscreen {
    interrupt-parent = <&gpio3>;
    interrupts = <10 IRQ_TYPE_EDGE_RISING>;
};
```

### Interrupt Controller

```dts
intc: interrupt-controller@48200000 {
    compatible = "ti,am33xx-intc";
    interrupt-controller;
    #interrupt-cells = <1>;
    reg = <0x48200000 0x1000>;
};
```

## IRQ Domain (for IRQ Controllers)

```c
#include <linux/irqdomain.h>

static int my_irqchip_probe(struct platform_device *pdev)
{
    struct irq_domain *domain;
    
    // Create IRQ domain
    domain = irq_domain_add_linear(pdev->dev.of_node,
                                    NUM_IRQS,
                                    &my_irq_domain_ops,
                                    priv);
    if (!domain) {
        dev_err(&pdev->dev, "Failed to create IRQ domain\n");
        return -ENOMEM;
    }
    
    return 0;
}
```

## Best Practices

✅ **DO:**
- Keep IRQ handlers short and fast
- Use threaded IRQs for complex processing
- Use `devm_request_irq()` for cleanup
- Check return value of IRQ functions
- Disable IRQ during critical hardware access
- Use appropriate deferred work mechanism

❌ **DON'T:**
- Sleep in hard IRQ handler
- Use mutex in hard IRQ handler
- Perform lengthy operations in IRQ context
- Forget to acknowledge/clear interrupt
- Share IRQ without IRQF_SHARED flag
- Access shared data without locking

## Debugging

```bash
# View IRQ statistics
cat /proc/interrupts

# View IRQ affinity
cat /proc/irq/72/smp_affinity

# View effective affinity
cat /proc/irq/72/effective_affinity

# Set IRQ affinity to CPU 0
echo 1 > /proc/irq/72/smp_affinity

# Kernel messages
dmesg | grep -i irq
```

## Common Patterns

### IRQ with Wait Queue

```c
static irqreturn_t mydev_irq(int irq, void *dev_id)
{
    struct mydev_data *data = dev_id;
    
    data->data_ready = true;
    wake_up_interruptible(&data->wait_queue);
    
    return IRQ_HANDLED;
}

static ssize_t mydev_read(struct file *file, char __user *buf,
                          size_t count, loff_t *ppos)
{
    struct mydev_data *data = file->private_data;
    
    // Wait for interrupt
    wait_event_interruptible(data->wait_queue, data->data_ready);
    data->data_ready = false;
    
    // Read data
    return count;
}
```
