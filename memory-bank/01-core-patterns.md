# Core Kernel Patterns

## Module Development Template
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
MODULE_VERSION("1.0");
```

## Common Header Files
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
#include <linux/delay.h>         // msleep, usleep_range
#include <linux/workqueue.h>     // Work queues
#include <linux/kthread.h>       // Kernel threads
```

## Error Codes (ALWAYS USE THESE)
```c
return 0;           // Success
return -ENOMEM;     // Out of memory
return -EINVAL;     // Invalid argument
return -EFAULT;     // Bad address (copy_to/from_user failed)
return -ENODEV;     // No such device
return -EBUSY;      // Device/resource busy
return -EAGAIN;     // Try again (temporary failure)
return -ENOSPC;     // No space left
return -ENOENT;     // No such file or directory
return -EIO;        // I/O error
return -ENOTSUPP;   // Operation not supported
return -EPERM;      // Permission denied
return -EACCES;     // Access denied
return -ETIMEDOUT;  // Timeout occurred
```

## Critical Macros
```c
// Logging (use pr_* instead of printk when possible)
pr_debug("Debug: %s\n", msg);      // Only with DEBUG or dynamic debug
pr_info("Info: %s\n", msg);        // Informational
pr_warn("Warning: %s\n", msg);     // Warning
pr_err("Error: %s\n", msg);        // Error
pr_crit("Critical: %s\n", msg);    // Critical

// Device logging (preferred in drivers)
dev_dbg(&pdev->dev, "Debug message\n");
dev_info(&pdev->dev, "Info message\n");
dev_warn(&pdev->dev, "Warning message\n");
dev_err(&pdev->dev, "Error message\n");

// Branch prediction
if (likely(condition))    // Condition is usually true
if (unlikely(condition))  // Condition is rarely true

// Container access
struct my_struct *ptr = container_of(member_ptr, struct my_struct, member);

// Array size
int size = ARRAY_SIZE(my_array);

// Bit operations
unsigned long flags = BIT(3);              // 0b1000
unsigned long mask = GENMASK(7, 4);        // 0b11110000

// Min/Max
int result = min(a, b);
int result = max(a, b);
int result = clamp(val, min, max);

// Error pointers
if (IS_ERR(ptr))
    return PTR_ERR(ptr);
return ERR_PTR(-ENOMEM);
```

## Copy To/From User Space
```c
// ALWAYS use these for user space data transfer
// NEVER directly dereference user space pointers!

// Copy from user space to kernel
if (copy_from_user(&kbuf, ubuf, size))
    return -EFAULT;

// Copy to user space from kernel
if (copy_to_user(ubuf, &kbuf, size))
    return -EFAULT;

// Get simple value from user space
if (get_user(kval, uptr))
    return -EFAULT;

// Put simple value to user space
if (put_user(kval, uptr))
    return -EFAULT;
```

## String Operations
```c
// Kernel string functions (safer than libc equivalents)
strncpy(dest, src, size);           // Copy string
strlcpy(dest, src, size);           // Copy with guaranteed null termination
snprintf(buf, size, "fmt %d", val); // Formatted print
kasprintf(GFP_KERNEL, "fmt %d", val); // Allocate and format
strcmp(s1, s2);                     // Compare strings
strncmp(s1, s2, n);                 // Compare n characters
strlen(s);                          // String length
```
