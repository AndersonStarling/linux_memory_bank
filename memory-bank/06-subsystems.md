# Kernel Subsystems Overview

## Directory Structure

### Core Kernel
```
kernel/          - Core kernel functionality
├── sched/       - Process scheduler
├── locking/     - Locking primitives
├── rcu/         - Read-Copy-Update
├── irq/         - IRQ handling
├── time/        - Timekeeping
├── power/       - Power management
├── printk/      - Kernel logging
└── module/      - Module loading/unloading
```

### Memory Management
```
mm/              - Memory management
├── Memory allocation (slab, slub, slob)
├── Virtual memory
├── Page cache
└── Memory reclaim
```

### Drivers
```
drivers/         - Device drivers
├── base/        - Core driver infrastructure
├── char/        - Character devices
├── block/       - Block devices
├── net/         - Network devices
├── gpio/        - GPIO subsystem
├── i2c/         - I2C bus
├── spi/         - SPI bus
├── usb/         - USB subsystem
├── pci/         - PCI subsystem
├── platform/    - Platform devices
└── staging/     - Staging drivers (not mainline quality)
```

### Networking
```
net/             - Network stack
├── core/        - Core networking
├── ipv4/        - IPv4 protocol
├── ipv6/        - IPv6 protocol
├── packet/      - Packet sockets
├── unix/        - Unix domain sockets
└── netlink/     - Netlink sockets
```

### Filesystems
```
fs/              - Filesystems
├── VFS layer (virtual filesystem)
├── ext4/        - ext4 filesystem
├── nfs/         - Network File System
├── proc/        - /proc filesystem
├── sysfs/       - /sys filesystem
└── debugfs/     - /sys/kernel/debug
```

### Architecture-Specific
```
arch/            - Architecture support
├── arm/         - ARM 32-bit
├── arm64/       - ARM 64-bit (AArch64)
├── x86/         - x86/x86_64
├── riscv/       - RISC-V
└── [others]
```

## Key Header Locations

### Core Headers
```
include/linux/           - Core kernel headers
├── kernel.h             - Core kernel functions
├── module.h             - Module support
├── init.h               - Initialization
├── slab.h               - Memory allocation
├── gfp.h                - Memory allocation flags
├── mm.h                 - Memory management
├── sched.h              - Scheduler
└── interrupt.h          - Interrupts
```

### Driver Headers
```
include/linux/
├── device.h             - Device model
├── platform_device.h    - Platform devices
├── i2c.h                - I2C
├── spi/spi.h            - SPI
├── usb.h                - USB
├── pci.h                - PCI
└── of.h                 - Device tree
```

### Userspace API Headers
```
include/uapi/            - Userspace API
└── linux/               - Kernel-to-user headers
```

### Architecture Headers
```
arch/*/include/          - Architecture-specific
└── asm/                 - Assembly headers
```

## Critical Subsystem Entry Points

### VFS (Virtual Filesystem)
```c
// include/linux/fs.h
struct file_operations {
    struct module *owner;
    loff_t (*llseek) (struct file *, loff_t, int);
    ssize_t (*read) (struct file *, char __user *, size_t, loff_t *);
    ssize_t (*write) (struct file *, const char __user *, size_t, loff_t *);
    int (*open) (struct inode *, struct file *);
    int (*release) (struct inode *, struct file *);
    // ... many more
};
```

### Block Layer
```c
// include/linux/blkdev.h
// include/linux/bio.h

// BIO - Basic I/O unit
struct bio {
    struct block_device *bi_bdev;
    sector_t bi_sector;
    // ...
};
```

### Network Device
```c
// include/linux/netdevice.h
struct net_device_ops {
    int (*ndo_open)(struct net_device *dev);
    int (*ndo_stop)(struct net_device *dev);
    netdev_tx_t (*ndo_start_xmit)(struct sk_buff *skb,
                                   struct net_device *dev);
    // ... many more
};
```

### Device Driver Model
```c
// include/linux/device.h
struct device {
    struct device *parent;
    struct device_driver *driver;
    void *driver_data;
    // ...
};

struct device_driver {
    const char *name;
    struct bus_type *bus;
    int (*probe) (struct device *dev);
    int (*remove) (struct device *dev);
    // ...
};
```

## Common Subsystem Patterns

### Power Management
```c
#include <linux/pm.h>

static int mydev_suspend(struct device *dev)
{
    /* Save state, power down */
    return 0;
}

static int mydev_resume(struct device *dev)
{
    /* Restore state, power up */
    return 0;
}

static SIMPLE_DEV_PM_OPS(mydev_pm_ops, mydev_suspend, mydev_resume);

static struct platform_driver mydev_driver = {
    .driver = {
        .name = "mydevice",
        .pm = &mydev_pm_ops,
    },
};
```

### Workqueues
```c
#include <linux/workqueue.h>

struct work_struct my_work;

static void my_work_handler(struct work_struct *work)
{
    /* Do work in process context */
}

// Initialize
INIT_WORK(&my_work, my_work_handler);

// Schedule work
schedule_work(&my_work);

// Delayed work
struct delayed_work my_delayed_work;
INIT_DELAYED_WORK(&my_delayed_work, my_work_handler);
schedule_delayed_work(&my_delayed_work, msecs_to_jiffies(1000));

// Cancel work
cancel_work_sync(&my_work);
cancel_delayed_work_sync(&my_delayed_work);
```

### Timers
```c
#include <linux/timer.h>

struct timer_list my_timer;

static void my_timer_callback(struct timer_list *t)
{
    /* Timer expired */
    pr_info("Timer expired\n");
}

// Initialize
timer_setup(&my_timer, my_timer_callback, 0);

// Start timer (expires in 5 seconds)
mod_timer(&my_timer, jiffies + msecs_to_jiffies(5000));

// Delete timer
del_timer_sync(&my_timer);
```

### Kernel Threads
```c
#include <linux/kthread.h>

static int my_thread_fn(void *data)
{
    while (!kthread_should_stop()) {
        /* Do work */
        
        // Sleep
        msleep(1000);
        
        // Or sleep interruptible
        if (kthread_should_stop())
            break;
    }
    return 0;
}

// Create and start thread
struct task_struct *thread;
thread = kthread_run(my_thread_fn, NULL, "my_thread");
if (IS_ERR(thread))
    return PTR_ERR(thread);

// Stop thread
kthread_stop(thread);
```

## Kconfig & Makefile Integration

### Kconfig Entry
```kconfig
# drivers/mydriver/Kconfig
config MY_DRIVER
    tristate "My Device Driver"
    depends on ARCH_ARM || COMPILE_TEST
    select REGMAP_I2C
    help
      This is my device driver for XYZ hardware.
      
      Say Y or M here to enable support.
      
      To compile this driver as a module, choose M here.
```

### Makefile Entry
```makefile
# drivers/mydriver/Makefile
obj-$(CONFIG_MY_DRIVER) += mydriver.o

# Multiple source files
mydriver-y := main.o probe.o irq.o

# Conditional compilation
mydriver-$(CONFIG_MY_FEATURE) += feature.o

# Include paths
ccflags-y += -I$(src)/include
```

### Integration into Parent
```kconfig
# drivers/Kconfig
source "drivers/mydriver/Kconfig"
```

```makefile
# drivers/Makefile
obj-$(CONFIG_MY_DRIVER) += mydriver/
```
