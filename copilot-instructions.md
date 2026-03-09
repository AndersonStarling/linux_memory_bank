# Linux Kernel Development Instructions

> **Memory Bank Structure**: This project uses a modular memory bank system for organized knowledge retention:
> 
> **Core Fundamentals:**
> - [01-core-patterns.md](memory-bank/01-core-patterns.md) - Module templates, headers, error codes, macros
> - [02-memory-management.md](memory-bank/02-memory-management.md) - Memory allocation, GFP flags, DMA
> - [03-locking-concurrency.md](memory-bank/03-locking-concurrency.md) - Mutexes, spinlocks, RCU, atomics
> - [04-device-drivers.md](memory-bank/04-device-drivers.md) - Character devices, platform drivers, interrupts
> - [05-build-debug.md](memory-bank/05-build-debug.md) - Build system, debugging, testing
> - [06-subsystems.md](memory-bank/06-subsystems.md) - Kernel subsystems, integration patterns
>
> **I/O & Communication Subsystems:**
> - [07-gpio-subsystem.md](memory-bank/07-gpio-subsystem.md) - GPIO consumer/driver API, device tree, IRQ
> - [08-pwm-subsystem.md](memory-bank/08-pwm-subsystem.md) - PWM consumer/driver API, state management, calculations
> - [09-i2c-subsystem.md](memory-bank/09-i2c-subsystem.md) - I2C client/adapter drivers, SMBus API, regmap integration
> - [10-spi-subsystem.md](memory-bank/10-spi-subsystem.md) - SPI protocol/controller drivers, 4-wire interface, transfer modes
>
> **Hardware Control & Power Management:**
> - [11-pinctrl-subsystem.md](memory-bank/11-pinctrl-subsystem.md) - Pin multiplexing/configuration, runtime state switching
> - [12-clock-subsystem.md](memory-bank/12-clock-subsystem.md) - Clock framework, prepare/enable semantics, rate management
> - [13-regulator-subsystem.md](memory-bank/13-regulator-subsystem.md) - Voltage/current regulation, PMIC integration
> - [14-dma-subsystem.md](memory-bank/14-dma-subsystem.md) - DMA engine, scatter-gather, cyclic transfers
>
> **Sensors & Interrupts:**
> - [15-iio-subsystem.md](memory-bank/15-iio-subsystem.md) - Industrial I/O, ADC/DAC, sensors, buffered acquisition
> - [16-irq-subsystem.md](memory-bank/16-irq-subsystem.md) - Interrupt handling, threaded IRQs, deferred work
>
> **AI Agent Workflow**: When starting work, review relevant memory bank files for context. These contain patterns, examples, and critical knowledge for Linux kernel development.

## Memory Bank - Quick Context Reference

### Critical File Locations
- **Main config**: `.config` (generated), `arch/*/configs/*_defconfig` (defaults)
- **Build outputs**: `vmlinux` (kernel image), `*.ko` (modules), `System.map` (symbols)
- **Headers**: `include/linux/` (core), `include/uapi/` (userspace API), `arch/*/include/`
- **Build system**: `Makefile` (root), `Kbuild`, `scripts/Makefile.*`
- **Code style**: `scripts/checkpatch.pl`, `.clang-format`, `Documentation/process/coding-style.rst`

### Common Header Files & Their Purpose
```c
#include <linux/module.h>        // Module loading/unloading
#include <linux/kernel.h>        // Kernel core functions (printk, etc.)
#include <linux/init.h>          // __init, __exit macros
#include <linux/fs.h>            // Filesystem operations
#include <linux/slab.h>          // Memory allocation (kmalloc, kfree)
#include <linux/uaccess.h>       // copy_to_user, copy_from_user
#include <linux/device.h>        // Device driver model
#include <linux/cdev.h>          // Character devices
#include <linux/mutex.h>         // Mutex locks
#include <linux/spinlock.h>      // Spinlocks
#include <linux/interrupt.h>     // Interrupt handling
#include <linux/platform_device.h> // Platform devices
#include <linux/of.h>            // Device tree
```

### Memory Allocation Patterns (REMEMBER THESE)
```c
// Kernel memory allocation
void *ptr = kmalloc(size, GFP_KERNEL);     // Can sleep, most common
void *ptr = kzalloc(size, GFP_KERNEL);     // Zero-initialized
void *ptr = kmalloc(size, GFP_ATOMIC);     // Cannot sleep (IRQ context)
kfree(ptr);                                 // Free memory

// Page allocation
struct page *page = alloc_pages(GFP_KERNEL, order);
free_pages((unsigned long)page_address(page), order);

// Virtual memory
void *ptr = vmalloc(size);                  // For large allocations
vfree(ptr);
```

### Locking Quick Reference (CRITICAL FOR CONCURRENCY)
```c
// Mutex (can sleep, use in process context)
struct mutex my_mutex;
mutex_init(&my_mutex);
mutex_lock(&my_mutex);
// ... critical section ...
mutex_unlock(&my_mutex);

// Spinlock (cannot sleep, use in interrupt/atomic context)
spinlock_t my_lock;
spin_lock_init(&my_lock);
spin_lock(&my_lock);
// ... critical section ...
spin_unlock(&my_lock);

// Spinlock with IRQ disable
unsigned long flags;
spin_lock_irqsave(&my_lock, flags);
// ... critical section ...
spin_unlock_irqrestore(&my_lock, flags);

// RCU (read-copy-update for read-heavy workloads)
rcu_read_lock();
// ... read critical section ...
rcu_read_unlock();
```

### Error Code Conventions (ALWAYS RETURN THESE)
```c
return 0;           // Success
return -ENOMEM;     // Out of memory
return -EINVAL;     // Invalid argument
return -EFAULT;     // Bad address (copy_to/from_user failed)
return -ENODEV;     // No such device
return -EBUSY;      // Device/resource busy
return -EAGAIN;     // Try again
return -ENOSPC;     // No space left
return -ENOENT;     // No such file or directory
return -EIO;        // I/O error
```

### Module Development Template (MEMORIZE THIS PATTERN)
```c
// SPDX-License-Identifier: GPL-2.0
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/init.h>

static int __init my_module_init(void)
{
    pr_info("Module loaded\n");
    return 0;
}

static void __exit my_module_exit(void)
{
    pr_info("Module unloaded\n");
}

module_init(my_module_init);
module_exit(my_module_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Your Name");
MODULE_DESCRIPTION("Module description");
```

### Device Tree & Platform Devices
```c
// Parsing device tree properties
struct device_node *np = pdev->dev.of_node;
u32 value;
of_property_read_u32(np, "property-name", &value);

// Platform device matching
static const struct of_device_id my_of_match[] = {
    { .compatible = "vendor,device-name", },
    { /* sentinel */ }
};
MODULE_DEVICE_TABLE(of, my_of_match);
```

### Critical Macros to Remember
```c
pr_debug(), pr_info(), pr_warn(), pr_err()  // Kernel logging
likely(), unlikely()                         // Branch prediction hints
container_of(ptr, type, member)             // Get containing structure
ARRAY_SIZE(array)                           // Array element count
BIT(nr), GENMASK(h, l)                      // Bit manipulation
IS_ERR(ptr), PTR_ERR(ptr), ERR_PTR(err)    // Error pointer conversion
```

## Project Architecture

This is the **Linux kernel 6.18-rc1** codebase, a monolithic kernel with modular driver architecture. Key subsystems:

- **Core kernel** ([kernel/](kernel/)): Process management, scheduling ([kernel/sched/](kernel/sched/)), locking ([kernel/locking/](kernel/locking/)), RCU ([kernel/rcu/](kernel/rcu/))
- **Memory management** ([mm/](mm/)): Page allocation, virtual memory, memory reclaim
- **Device drivers** ([drivers/](drivers/)): Hardware abstraction organized by bus type and device class
- **Filesystems** ([fs/](fs/)): VFS layer and specific filesystem implementations
- **Architecture-specific** ([arch/](arch/)): CPU architecture support (arm/, x86/, etc.)
- **Block layer** ([block/](block/)): Storage device I/O and scheduling
- **Network stack** ([net/](net/)): Protocol implementations and network device interface
- **Rust support** ([rust/](rust/)): Rust kernel bindings and safe driver development

## Build System & Configuration

### Kbuild System
- **Kconfig**: Configuration language defining kernel options ([Kconfig](Kconfig), [*/Kconfig](Documentation/kbuild/kconfig.rst))
- **Makefiles**: Recursive build system with [Makefile](Makefile) as root
- **Cross-compilation**: Use `ARCH=` and `CROSS_COMPILE=` variables

### Essential Build Commands
```bash
make help                    # List all targets
make defconfig              # Default configuration
make menuconfig             # Interactive configuration
make modules                # Build loadable modules
make modules_install        # Install modules to system
make clean                  # Remove build artifacts
make mrproper              # Full clean including .config
```

### Configuration Management
- `.config`: Current kernel configuration
- `defconfig`: Architecture-specific default configs in [arch/*/configs/](arch/)
- Environment variables: `KCONFIG_CONFIG`, `KCONFIG_ALLCONFIG`

## Development Workflows

### Code Quality & Style
- **checkpatch.pl**: Mandatory style checker - `scripts/checkpatch.pl <patch>`
- **Sparse**: Static analysis - `make C=1` or `make C=2`
- **Coccinelle**: Semantic patches in [scripts/coccinelle/](scripts/coccinelle/)
- **SPDX headers**: Required license identifiers in all files

### Rust Development
- **Enabled with**: `CONFIG_RUST=y`
- **Bindings**: Auto-generated in [rust/bindings/](rust/)
- **Safe abstractions**: Core kernel APIs wrapped in [rust/kernel/](rust/)
- **Driver examples**: See [samples/rust/](samples/rust/) (if exists)

### Module Development
- **In-tree**: Add to appropriate [drivers/](drivers/) subdirectory
- **Out-of-tree**: Use `make M=/path/to/module` 
- **Module signature**: Required for secure boot environments

### Testing
- **KUnit**: Kernel unit testing framework
- **Selftests**: User-space test suite in [tools/testing/selftests/](tools/testing/selftests/)
- **Kernel CI**: Various automated testing infrastructures

## Kernel-Specific Patterns

### Memory Management
- **GFP flags**: `GFP_KERNEL` (can sleep), `GFP_ATOMIC` (cannot sleep)
- **Reference counting**: `get_*()` / `put_*()` patterns for object lifetime
- **Copy to/from user**: Always use `copy_to_user()` / `copy_from_user()`

### Concurrency & Locking
- **Spinlocks**: For short critical sections, cannot sleep
- **Mutexes**: For longer critical sections, can sleep
- **RCU**: Read-copy-update for read-heavy workloads
- **Per-CPU data**: Use `per_cpu()` macros for scalability

### Error Handling
- **Return codes**: Negative errno values (e.g., `-ENOMEM`, `-EINVAL`)
- **Cleanup patterns**: Use `goto` for error paths and resource cleanup
- **Unlikely errors**: Use `unlikely()` / `likely()` for branch prediction

### Device Driver Model
- **sysfs integration**: Automatic device/driver/class hierarchy
- **Power management**: Suspend/resume callbacks mandatory
- **Device tree**: Hardware description for embedded systems

## Critical Integration Points

### Subsystem Boundaries
- **VFS**: [include/linux/fs.h](include/linux/fs.h) - filesystem interface
- **Block layer**: [include/linux/bio.h](include/linux/bio.h) - storage I/O
- **Network**: [include/linux/netdevice.h](include/linux/netdevice.h) - network interface
- **Driver core**: [include/linux/device.h](include/linux/device.h) - device model

### Cross-Architecture Support
- **Generic code**: Use [asm-generic/](include/asm-generic/) headers when possible
- **Architecture hooks**: Implement in [arch/*/](arch/) directories
- **Endianness**: Use `cpu_to_le32()`, `le32_to_cpu()` family functions

## Development Environment Requirements

**Minimum versions** (see [Documentation/process/changes.rst](Documentation/process/changes.rst)):
- GCC 8.1+ or Clang 15.0+
- GNU Make 4.0+
- Rust 1.78.0+ (if CONFIG_RUST=y)
- Various tools: flex, bison, pahole, etc.

## Common Workflows & Debugging

### Building Specific Components
```bash
# Build only specific directory
make drivers/net/
make M=drivers/staging/mydriver/

# Build with warnings
make W=1                        # Enable extra warnings
make C=1                        # Run sparse on changed files
make C=2                        # Run sparse on all files

# Cross-compile for ARM (BeagleBone)
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- zImage
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- dtbs
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- modules
```

### Configuration Workflows
```bash
# Start from architecture defaults
make ARCH=arm multi_v7_defconfig

# Enable specific config
scripts/config --enable CONFIG_MODULE_NAME
scripts/config --disable CONFIG_MODULE_NAME
scripts/config --set-val CONFIG_NAME value

# Check what changed
scripts/diffconfig .config.old .config

# Find config dependencies
make ARCH=arm menuconfig         # Navigate and use '?' for help
```

### Debugging Techniques
```c
// Dynamic debug (enable at runtime via debugfs)
pr_debug("Debug message\n");

// Kernel print with rate limiting
printk_ratelimited(KERN_INFO "Frequent message\n");

// Dump stack trace
dump_stack();

// Check conditions
WARN_ON(condition);              // Print warning if true
BUG_ON(condition);               // Kernel panic if true (use sparingly!)

// Memory debugging
CONFIG_KASAN=y                   // Kernel Address Sanitizer
CONFIG_DEBUG_KMEMLEAK=y          // Memory leak detector
CONFIG_SLUB_DEBUG=y              // SLUB allocator debugging
```

### Testing & Validation
```bash
# Code style check (ALWAYS RUN THIS)
scripts/checkpatch.pl --strict --file drivers/mydriver/myfile.c
scripts/checkpatch.pl --strict myfile.patch

# Build test for multiple architectures
make ARCH=arm allmodconfig
make ARCH=arm64 defconfig
make ARCH=x86_64 allnoconfig

# Run KUnit tests
./tools/testing/kunit/kunit.py run
./tools/testing/kunit/kunit.py run --kunitconfig=drivers/mydriver/

# Run selftests
make -C tools/testing/selftests
make -C tools/testing/selftests TARGETS=net run_tests
```

### Module Loading & Debugging
```bash
# Load module with parameters
insmod mymodule.ko param=value

# View module info
modinfo mymodule.ko
lsmod | grep mymodule

# Kernel logs
dmesg | tail
dmesg -w                         # Watch mode
journalctl -k -f                 # Follow kernel logs

# Remove module
rmmod mymodule
```

### Common Pitfalls to AVOID
- ❌ Sleeping in atomic context (spinlock, IRQ handler)
- ❌ Forgetting to free allocated memory (use kfree, vfree)
- ❌ Not checking return values (especially copy_to/from_user)
- ❌ Using floating point in kernel space (not allowed!)
- ❌ Direct hardware access without ioremap()
- ❌ Missing SPDX license identifier at file start
- ❌ Not using proper locking for shared data
- ❌ Ignoring checkpatch.pl warnings

## Key Documentation References

- [Documentation/process/](Documentation/process/): Development process and submitting patches
- [Documentation/kbuild/](Documentation/kbuild/): Build system documentation  
- [Documentation/rust/](Documentation/rust/): Rust kernel development (if exists)
- [MAINTAINERS](MAINTAINERS): Subsystem ownership and contact information
- [Documentation/driver-api/](Documentation/driver-api/): Driver development APIs
- [Documentation/core-api/](Documentation/core-api/): Core kernel APIs
- [Documentation/devicetree/bindings/](Documentation/devicetree/bindings/): Device tree bindings