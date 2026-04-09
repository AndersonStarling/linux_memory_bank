# Common Linux Kernel Debugging Patterns

## Pattern 1: NULL Pointer Dereference in Driver Registration

### Complete Crash Log
```
[    4.123456] Unable to handle kernel NULL pointer dereference at virtual address 00000000 when read
[    4.123789] [00000000] *pgd=00000000
[    4.124012] Internal error: Oops: 5 [#1] SMP ARM
[    4.124234] Modules linked in: gpio_bb(O+) usb_f_acm u_serial usb_f_mass_storage
[    4.124567] CPU: 0 PID: 156 Comm: insmod Tainted: G           O      6.1.0-ti #1
[    4.124890] Hardware name: Generic AM33XX (Flattened Device Tree)
[    4.125123] PC is at strcmp+0x4/0x34
[    4.125345] LR is at kset_find_obj+0x3c/0x78
[    4.125567] pc : [<c1165844>]    lr : [<c114e4e4>]    psr: 60000013
[    4.125789] sp : c3c1bd48  ip : 00000000  fp : 00000000
[    4.126012] r10: c17a7459  r9 : c3c1a000  r8 : c17a7468
[    4.126234] r7 : c1708bd8  r6 : c17a7400  r5 : c17a7459  r4 : c23fc814
[    4.126456] r3 : 00000000  r2 : c23fc814  r1 : 00000000  r0 : c17a7459
[    4.126678] Flags: nZCv  IRQs on  FIQs on  Mode SVC_32  ISA ARM  Segment none
[    4.126901] Control: 10c5387d  Table: 83c24019  DAC: 00000051
[    4.127123] Register r0 information: non-slab/vmalloc memory
[    4.127345] Register r1 information: NULL pointer
[    4.127567] Register r2 information: slab kmalloc-1k start c23fc800 pointer offset 20 size 1024
[    4.127890] Process insmod (pid: 156, stack limit = 0x(ptrval))
[    4.128123] Stack: (0xc3c1bd48 to 0xc3c1c000)
[    4.128345] bd40:                   c23fc814 c114e4e4 c17a7459 c17a7400 c1708bd8 c17a7468
[    4.128678] bd60: c3c1a000 c17a7459 00000000 c114e710 c17a7468 00000000 00000000 c17a7400
[    4.129012] bd80: c1708bd8 c0cc7d88 c17a7400 c0cc8650 ffffe000 c23fc800 c17a7400 c0cc8a40
[    4.129345] bda0: c0d91c3c 00000000 c17a7400 c17a7468 c0cc8a00 c0cc9068 c17a7468 c23e3a00
[    4.129678] bdc0: 00000000 c0cc79f8 c17a7468 c23e3a00 c0ccc5e8 c0cc6e1c bf00a050 00000000
[    4.130012] bde0: c23e3a00 c0cc70cc 00000001 bf00a050 bf00a050 c0201a60 00000001 c02019a0
[    4.130345] be00: bf00a000 bf00a074 c1708bd8 c0202584 00000000 bf00a050 00000001 c0200a84
[    4.130678] be20: c17a0754 00000000 c3dd3bc0 c02010a0 bf00a050 00000000 c23e6300 c01ff438
[    4.131012] be40: 00007fff c02001c0 00000000 00000000 bf00a000 00000000 00000000 c020032c
[    4.131345] be60: 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000
[    4.131678] be80: 00000000 00000000 00000000 00000000 00000000 c0100244 c3c1a000 00000000
[    4.132012] bea0: 00000000 c01009e8 00000000 c3dd3bc0 b6f66e38 00000000 00000000 c01418e0
[    4.132345] bec0: 00000000 00000003 00000000 004b8000 004b9878 00000000 00000000 00000000
[    4.132678] bee0: 00000000 00000000 b6f66f0c 004b9848 004b9000 00000000 00000000 b6f66e38
[    4.133012] bf00: 00000080 c0100244 c3c1a000 00000000 00000003 c0141b28 c3dd3bc0 00003f88
[    4.133345] bf20: 00000000 00000000 c3c1bf58 00000000 00000000 00000000 00000000 00000000
[    4.133678] bf40: 00000000 00000000 00000000 00000000 00000000 00000000 00000000 004b9000
[    4.134012] bf60: 00000000 00000000 00000000 c01001c4 00000000 004b8170 00000000 004b8000
[    4.134345] bf80: 004b9878 00000000 00000000 00000000 00000000 00000000 b6f66f0c 004b9848
[    4.134678] bfa0: 004b9000 c0100060 004b9848 004b9000 00000003 b6f66e38 00000000 00000000
[    4.135012] bfc0: 004b9848 004b9000 00000000 00000080 004b8170 004b9878 b6f66f0c 00000000
[    4.135345] bfe0: 00000000 bed21e0c 00000000 b6ee5e24 60000010 00000003 00000000 00000000
[    4.135678] [<c1165844>] (strcmp) from [<c114e4e4>] (kset_find_obj+0x3c/0x78)
[    4.136012] [<c114e4e4>] (kset_find_obj) from [<c0cc8650>] (driver_find+0x1c/0x40)
[    4.136345] [<c0cc8650>] (driver_find) from [<c0cc8a40>] (bus_add_driver+0x40/0x1e0)
[    4.136678] [<c0cc8a40>] (bus_add_driver) from [<c0cc9068>] (driver_register+0x68/0x108)
[    4.137012] [<c0cc9068>] (driver_register) from [<c0cc79f8>] (__platform_driver_register+0x28/0x30)
[    4.137456] [<c0cc79f8>] (__platform_driver_register) from [<bf00a074>] (gpio_bb_driver_init+0x74/0x1000 [gpio_bb])
[    4.137890] [<bf00a074>] (gpio_bb_driver_init [gpio_bb]) from [<c0100244>] (do_one_initcall+0x44/0x1e0)
[    4.138345] [<c0100244>] (do_one_initcall) from [<c01418e0>] (do_init_module+0x48/0x1e8)
[    4.138678] [<c01418e0>] (do_init_module) from [<c0141b28>] (load_module+0x1a0/0x1f8)
[    4.139012] [<c0141b28>] (load_module) from [<c01001c4>] (sys_finit_module+0xa4/0xd8)
[    4.139345] [<c01001c4>] (sys_finit_module) from [<c0100060>] (ret_fast_syscall+0x0/0x54)
[    4.139678] Exception stack(0xc3c1bfa8 to 0xc3c1bff0)
[    4.140012] bfa0:                   004b9848 004b9000 00000003 b6f66e38 00000000 00000000
[    4.140345] bfc0: 004b9848 004b9000 00000000 00000080 004b8170 004b9878 b6f66f0c 00000000
[    4.140678] bfe0: 00000000 bed21e0c 00000000 b6ee5e24
[    4.141012] Code: e1a0c001 e3520000 012fff1e e5d02000 (e5d13000)
[    4.141567] ---[ end trace 0000000000000000 ]---
[    4.141890] Kernel panic - not syncing: Fatal exception
```

### Error Signature (Quick Identification)
```
Unable to handle kernel NULL pointer dereference at virtual address 00000000 when read
PC is at strcmp+0x4/0x34
LR is at kset_find_obj+0x3c/0x78
```

### Symptoms
- Crash during `insmod` or module init
- Even with empty probe() function
- Register `r1` or similar argument register shows `00000000` (NULL)
- Call stack shows: `strcmp` ← `kset_find_obj` ← `driver_register` ← `platform_driver_register`

### Root Cause
**Missing `.driver.name` in platform_driver structure**

```c
// ❌ WRONG - Causes NULL pointer crash
static struct platform_driver my_driver = {
    .probe = my_probe,
    .driver = {
        .of_match_table = my_match,
        // Missing .name → NULL pointer!
    },
};
```

### Why This Crashes

When `module_platform_driver()` is called, the following sequence happens:
1. `platform_driver_register()` is called
2. `driver_register()` tries to register driver in sysfs
3. `kset_find_obj()` searches for duplicate driver names
4. `strcmp(driver->name, existing->name)` is called
5. If `driver->name == NULL` → **CRASH in strcmp()**

### Stack Trace Deep Dive

#### Call Chain Analysis
```
[<c1165844>] (strcmp) from [<c114e4e4>] (kset_find_obj+0x3c/0x78)
[<c114e4e4>] (kset_find_obj) from [<c0cc8650>] (driver_find+0x1c/0x40)
[<c0cc8650>] (driver_find) from [<c0cc8a40>] (bus_add_driver+0x40/0x1e0)
[<c0cc8a40>] (bus_add_driver) from [<c0cc9068>] (driver_register+0x68/0x108)
[<c0cc9068>] (driver_register) from [<c0cc79f8>] (__platform_driver_register+0x28/0x30)
[<c0cc79f8>] (__platform_driver_register) from [<bf00a074>] (gpio_bb_driver_init+0x74/0x1000 [gpio_bb])
[<bf00a074>] (gpio_bb_driver_init [gpio_bb]) from [<c0100244>] (do_one_initcall+0x44/0x1e0)
[<c0100244>] (do_one_initcall) from [<c01418e0>] (do_init_module+0x48/0x1e8)
[<c01418e0>] (do_init_module) from [<c0141b28>] (load_module+0x1a0/0x1f8)
[<c01401b28>] (load_module) from [<c01001c4>] (sys_finit_module+0xa4/0xd8)
[<c01001c4>] (sys_finit_module) from [<c0100060>] (ret_fast_syscall+0x0/0x54)
```

#### Bottom-Up Analysis (How We Got Here)

1. **Userspace** → `insmod gpio_bb.ko` (PID 156)
2. **Syscall** → `sys_finit_module()` - system call to load module
3. **Module Loading** → `load_module()` - parse ELF, allocate memory
4. **Module Init** → `do_init_module()` - prepare module initialization
5. **Init Call** → `do_one_initcall()` - call module's __init function
6. **Your Module** → `gpio_bb_driver_init()` - **module_platform_driver()** expands here
7. **Platform Registration** → `__platform_driver_register()` - register platform driver
8. **Driver Core** → `driver_register()` - generic driver registration
9. **Bus Layer** → `bus_add_driver()` - add driver to platform bus
10. **Duplicate Check** → `driver_find()` - check if driver already exists
11. **Kobject Search** → `kset_find_obj()` - search for existing kobject with same name
12. **String Compare** → `strcmp()` - **CRASH HERE!** Compare NULL with existing name

#### Why Crash at strcmp?

**In kset_find_obj():**
```c
struct kobject *kset_find_obj(struct kset *kset, const char *name)
{
    struct kobject *k;
    struct kobject *ret = NULL;
    
    spin_lock(&kset->list_lock);
    
    list_for_each_entry(k, &kset->list, entry) {
        if (kobject_name(k) && !strcmp(kobject_name(k), name))  // ← name is NULL!
            //           ^existing        ^new driver->name (NULL!)
            ret = kobject_get(k);
            break;
        }
    }
    
    spin_unlock(&kset->list_lock);
    return ret;
}
```

The function tries to find if a driver with the same name already exists:
- `kobject_name(k)`: Returns existing driver's name (valid string)
- `name`: Your driver's name ← **NULL!**
- `strcmp(valid_string, NULL)` → **CRASH!**

#### Stack Memory Dump Analysis

```
Stack: (0xc3c1bd48 to 0xc3c1c000)  ← Stack grows downward, 724 bytes used
bd40:                   c23fc814 c114e4e4 c17a7459 c17a7400 c1708bd8 c17a7468
bd60: c3c1a000 c17a7459 00000000 c114e710 c17a7468 00000000 00000000 c17a7400
                         ^^^^^^^^ ← NULL value on stack (the missing name)
```

- Stack base: 0xc3c1c000 (8KB kernel stack top)
- Current SP: 0xc3c1bd48
- Stack used: 0xc3c1c000 - 0xc3c1bd48 = 696 bytes
- Plenty of stack space (no stack overflow)

### Debug Steps

#### 1. Analyze Stack Trace (Most Important!)
```
PC is at strcmp+0x4/0x34          ← Crashed in strcmp
LR is at kset_find_obj+0x3c/0x78  ← Called from kset_find_obj
r1 : 00000000                     ← Argument 2 is NULL!
r0 : c17a7459                     ← Argument 1 is valid
```

**Conclusion:** strcmp(valid_string, NULL) → second argument is NULL → driver->name is NULL.

#### 2. Check Register Values
```
r0 : c17a7459  ← Valid string pointer
r1 : 00000000  ← NULL! This is the problem
```

In strcmp(r0, r1), r1 is NULL → driver->name is NULL.

#### 3. Add Debug Prints
```c
static int __init my_init(void)
{
    pr_alert("=== Module Init ===\n");
    pr_alert("Driver name: %s\n", 
             my_driver.driver.name ? my_driver.driver.name : "NULL");
    return platform_driver_register(&my_driver);
}
```

#### 4. Use addr2line
```bash
addr2line -e vmlinux -f 0xc114e4e4  # kset_find_obj address
addr2line -e vmlinux -f 0xc1165844  # strcmp address
```

#### 5. Enable KASAN (If available)
```bash
# In kernel config
CONFIG_KASAN=y
CONFIG_KASAN_INLINE=y
```

KASAN will catch NULL dereference earlier with clearer message.

#### 6. Check with objdump
```bash
arm-linux-gnueabihf-objdump -t my_module.ko | grep driver
# Look for .driver.name field
```

### Solution
```c
// ✅ CORRECT - Always set .name
static struct platform_driver my_driver = {
    .probe = my_probe,
    .remove = my_remove,
    .driver = {
        .name = "my-driver",              // ❗ REQUIRED
        .of_match_table = my_match,
        .owner = THIS_MODULE,              // Recommended
    },
};
```

### Prevention
Always use this template for platform drivers:
```c
static struct platform_driver my_driver = {
    .probe  = my_probe,
    .remove = my_remove,
    .driver = {
        .name   = "my-driver",             // ❗ ALWAYS SET
        .of_match_table = my_of_match,
        .owner  = THIS_MODULE,
    },
};

module_platform_driver(my_driver);
```

---

## Pattern 2: GPIO Chip Registration NULL Pointer

### Complete Crash Log Example
```
[   12.456789] Unable to handle kernel NULL pointer dereference at virtual address 00000000 when read
[   12.456890] [00000000] *pgd=00000000
[   12.456912] Internal error: Oops: 5 [#1] SMP ARM
[   12.456934] Modules linked in: gpio_bb(O+)
[   12.456956] CPU: 0 PID: 187 Comm: insmod Tainted: G           O      6.1.0 #1
[   12.457012] PC is at strcmp+0x4/0x34
[   12.457034] LR is at gpiochip_add_data_with_key+0x128/0x4a0
[   12.457056] pc : [<c1165844>]    lr : [<c0a24c88>]    psr: 60000013
[   12.457078] sp : c2d3bd68  ip : 00000000  fp : 00000000
[   12.457100] r10: c2345000  r9 : 00000000  r8 : c0a25100
[   12.457122] r7 : c23e4800  r6 : 00000000  r5 : bf00b0c0  r4 : c2345000
[   12.457144] r3 : 00000000  r2 : 00000000  r1 : 00000000  r0 : c1a2d567
[   12.457166] Flags: nZCv  IRQs on  FIQs on  Mode SVC_32  ISA ARM  Segment none
```

### Register Analysis
```
r0 : c1a2d567  ← Valid string (existing GPIO chip label)
r1 : 00000000  ← NULL! (your chip->label is NULL)
r5 : bf00b0c0  ← Your gpio_chip structure pointer
r4 : c2345000  ← GPIO device structure
```

**Problem:** `chip->label == NULL`
- gpiochip_add_data tries to check for duplicate labels
- Calls `strcmp(existing_chip->label, new_chip->label)`
- Your `chip->label` is NULL → crash

### Error Signature
```
Unable to handle kernel NULL pointer dereference
PC is at strcmp+0x...
LR is at gpiochip_add_data_with_key+0x...
r1 : 00000000  ← chip->label is NULL
```

### Symptoms
- Crash when calling `gpiochip_add_data()`
- Probe function runs but crashes during GPIO chip registration

### Root Cause
**Missing required fields in gpio_chip structure**

```c
// ❌ WRONG - Missing critical fields
struct gpio_chip *chip = devm_kzalloc(...);
chip->get = my_get;
chip->set = my_set;
gpiochip_add_data(chip, data);  // CRASH!
```

### Required Fields
```c
// ✅ CORRECT - All required fields
chip->label = "my-gpio";                    // ❗ REQUIRED - NULL causes crash
chip->parent = dev;                          // ❗ REQUIRED
chip->owner = THIS_MODULE;                   // ❗ REQUIRED
chip->base = -1;                             // -1 for dynamic allocation
chip->ngpio = 32;                            // ❗ REQUIRED - Number of GPIO lines

chip->get_direction = my_get_direction;      // Recommended
chip->direction_input = my_direction_input;  // ❗ REQUIRED
chip->direction_output = my_direction_output;// ❗ REQUIRED
chip->get = my_get;                          // ❗ REQUIRED
chip->set = my_set;                          // ❗ REQUIRED
```

### Debug Steps

#### 1. Check if label is NULL
```c
pr_alert("GPIO chip label: %s\n", chip->label ? chip->label : "NULL");
```

#### 2. Verify all required fields
```c
pr_alert("label=%s parent=%p owner=%p ngpio=%d\n",
         chip->label ? chip->label : "NULL",
         chip->parent, chip->owner, chip->ngpio);
```

#### 3. Use devm_gpiochip_add_data
```c
// Use devm_* for automatic cleanup
ret = devm_gpiochip_add_data(dev, chip, data);
```

### Solution
```c
static int my_probe(struct platform_device *pdev)
{
    struct gpio_chip *chip;
    
    chip = devm_kzalloc(&pdev->dev, sizeof(*chip), GFP_KERNEL);
    if (!chip)
        return -ENOMEM;
    
    // Set ALL required fields
    chip->label = "my-gpio";
    chip->parent = &pdev->dev;
    chip->owner = THIS_MODULE;
    chip->base = -1;
    chip->ngpio = 32;
    
    chip->get_direction = my_get_direction;
    chip->direction_input = my_direction_input;
    chip->direction_output = my_direction_output;
    chip->get = my_get;
    chip->set = my_set;
    
    return devm_gpiochip_add_data(&pdev->dev, chip, data);
}
```

---

## Pattern 3: Module Probe Not Called

### Symptoms
- `insmod` succeeds but probe() never runs
- No crash, no error, just nothing happens
- Module shows in `lsmod` but device not registered

### Debug Steps

#### 1. Check if device tree node exists
```bash
# Check compiled DTB
dtc -I dtb -O dts /boot/dtb/am335x-boneblack.dtb | grep compatible
```

#### 2. Verify compatible string matches
```c
// In driver
static const struct of_device_id my_of_match[] = {
    { .compatible = "vendor,mydevice", },  // Must match DT!
    { }
};
```

```dts
// In device tree
mydevice {
    compatible = "vendor,mydevice";  // Must match driver!
    reg = <0x... 0x...>;
};
```

#### 3. Check if device is registered
```bash
ls /sys/bus/platform/devices/
# Look for your device

cat /sys/bus/platform/devices/*/uevent
# Check MODALIAS
```

#### 4. Check driver registration
```bash
ls /sys/bus/platform/drivers/
# Look for your driver

cat /sys/bus/platform/drivers/my-driver/uevent
```

#### 5. Manual bind (Test)
```bash
# Force bind driver to device
echo "my-device" > /sys/bus/platform/drivers/my-driver/bind
```

#### 6. Check dmesg for probe defer
```bash
dmesg | grep -i "defer\|my-driver"
# Look for -EPROBE_DEFER
```

### Common Causes

#### Cause 1: Compatible string mismatch
```c
// Driver has different compatible than DT
.compatible = "gpio-bb"      // In driver
.compatible = "gpio-beagle"  // In DT - MISMATCH!
```

#### Cause 2: Missing MODULE_DEVICE_TABLE
```c
// ❌ WRONG - Forgot MODULE_DEVICE_TABLE
static const struct of_device_id my_match[] = {
    { .compatible = "vendor,device", },
    { }
};
// Missing: MODULE_DEVICE_TABLE(of, my_match);
```

```c
// ✅ CORRECT
static const struct of_device_id my_match[] = {
    { .compatible = "vendor,device", },
    { }
};
MODULE_DEVICE_TABLE(of, my_match);  // ❗ REQUIRED for module autoload
```

#### Cause 3: Device not in device tree
```bash
# Check if device node exists in loaded DTB
cat /proc/device-tree/...
ls /sys/firmware/devicetree/base/
```

#### Cause 4: Device disabled in DT
```dts
mydevice {
    compatible = "vendor,device";
    status = "disabled";  // ❌ Driver won't probe!
};
```

### Solution Template
```c
// Complete driver template
static int my_probe(struct platform_device *pdev)
{
    pr_alert("=== PROBE CALLED ===\n");  // Debug
    // ... probe code ...
    return 0;
}

static const struct of_device_id my_of_match[] = {
    { .compatible = "vendor,mydevice", },
    { }
};
MODULE_DEVICE_TABLE(of, my_of_match);  // ❗ REQUIRED

static struct platform_driver my_driver = {
    .probe = my_probe,
    .driver = {
        .name = "my-driver",           // ❗ REQUIRED
        .of_match_table = my_of_match,
    },
};

module_platform_driver(my_driver);
```

Device tree:
```dts
mydevice: mydevice@48000000 {
    compatible = "vendor,mydevice";  // Match driver
    reg = <0x48000000 0x1000>;
    status = "okay";                  // ❗ Enable device
};
```

---

## Pattern 4: Printk Not Showing Up

### Symptoms
- `printk()` called but nothing in `dmesg`
- Probe runs but no debug output
- Crash happens but debug line not visible

### Root Causes

#### 1. Log level too low
```c
// ❌ Default printk may not show
printk("Debug message\n");  // Might be filtered

// ✅ Use explicit log level
printk(KERN_ALERT "Debug message\n");  // Always shows
pr_alert("Debug message\n");            // Preferred
```

#### 2. Console log level filtering
```bash
# Check current log level
cat /proc/sys/kernel/printk
# Output: 4  4  1  7
#         ^
#         Console log level (only shows priority 0-4)

# Increase to see all messages
dmesg -n 8
# or
echo 8 > /proc/sys/kernel/printk
```

#### 3. Crash before buffer flush
```c
// Crash happens too fast, printk not flushed yet
printk(KERN_INFO "Before crash\n");
some_crash_here();  // Buffer not flushed!
```

### Solutions

#### Use high priority levels
```c
pr_emerg("Emergency\n");   // 0 - Always shows
pr_alert("Alert\n");        // 1 - Always shows  ✅ Use for debug
pr_crit("Critical\n");      // 2
pr_err("Error\n");          // 3
pr_warn("Warning\n");       // 4
pr_notice("Notice\n");      // 5
pr_info("Info\n");          // 6
pr_debug("Debug\n");        // 7 - Needs DEBUG or dynamic debug
```

#### Use dev_* functions
```c
// Better than pr_*, includes device name
dev_alert(&pdev->dev, "Alert message\n");
dev_info(&pdev->dev, "Info message\n");
dev_err(&pdev->dev, "Error message\n");
```

#### Check dmesg after crash
```bash
# After reboot, check previous boot logs
dmesg | grep -i "my-driver"
journalctl -k -b -1  # Previous boot
```

#### Enable dynamic debug
```bash
# Enable all debug for your module
echo 'module my_module +p' > /sys/kernel/debug/dynamic_debug/control

# Enable for specific file
echo 'file my_driver.c +p' > /sys/kernel/debug/dynamic_debug/control
```

---

## General Debugging Toolbox

### Essential Commands
```bash
# View kernel messages
dmesg
dmesg -w              # Watch mode
dmesg -T              # With timestamps
dmesg | tail -100     # Last 100 lines

# Kernel logs with journalctl
journalctl -k         # Kernel messages
journalctl -k -f      # Follow
journalctl -k -b      # Current boot
journalctl -k -b -1   # Previous boot

# Module info
lsmod                 # List loaded modules
modinfo my_module.ko  # Module information
insmod -v my_module.ko # Verbose insert

# Device tree
dtc -I dtb -O dts /boot/am335x-boneblack.dtb
ls /proc/device-tree/
cat /sys/firmware/devicetree/base/.../compatible

# Platform devices/drivers
ls /sys/bus/platform/devices/
ls /sys/bus/platform/drivers/
cat /sys/bus/platform/drivers/my-driver/uevent

# GPIO
cat /sys/kernel/debug/gpio
ls /sys/class/gpio/
```

### Debug Print Template
```c
#define DEBUG_PRINT(fmt, ...) \
    pr_alert("[%s:%d] " fmt "\n", __func__, __LINE__, ##__VA_ARGS__)

// Usage
DEBUG_PRINT("Probe called");
DEBUG_PRINT("Value: %d", value);
```

### Kernel Config for Debugging
```
CONFIG_DEBUG_INFO=y              # Debug symbols
CONFIG_DEBUG_KERNEL=y            # Kernel debugging
CONFIG_KASAN=y                   # Address sanitizer
CONFIG_DEBUG_KMEMLEAK=y          # Memory leak detection
CONFIG_SLUB_DEBUG=y              # SLUB debugging
CONFIG_DEBUG_PAGEALLOC=y         # Page alloc debugging
CONFIG_DYNAMIC_DEBUG=y           # Dynamic debug
CONFIG_KALLSYMS=y                # Symbol table
CONFIG_KALLSYMS_ALL=y            # All symbols
CONFIG_DEBUG_FS=y                # debugfs
```

### Quick Checklist for New Driver

- [ ] ✅ Set `.driver.name` in platform_driver
- [ ] ✅ Set `MODULE_DEVICE_TABLE(of, ...)`
- [ ] ✅ Compatible string matches device tree
- [ ] ✅ Device tree node has `status = "okay"`
- [ ] ✅ GPIO chip has `.label`, `.parent`, `.owner`, `.ngpio`
- [ ] ✅ Implement `.direction_input` and `.direction_output`
- [ ] ✅ Use `devm_*` functions for auto cleanup
- [ ] ✅ Check return values with `IS_ERR()` for pointers
- [ ] ✅ Use `pr_alert()` or `dev_alert()` for debug
- [ ] ✅ Test with empty probe first, add code incrementally

---

---

## ARM Register Reference Guide

### Core Registers (R0-R15)

#### General Purpose Registers
```
R0-R3   : Function arguments and return values
          - R0: 1st argument / return value
          - R1: 2nd argument
          - R2: 3rd argument  
          - R3: 4th argument
          - Additional args passed on stack

R4-R11  : Callee-saved registers
          - Must be preserved across function calls
          - Can be used freely within function
          
R12 (IP): Intra-Procedure scratch register
          - Can be corrupted by function calls
          - Often used by compiler for temporary values
```

#### Special Purpose Registers
```
R13 (SP): Stack Pointer
          - Points to top of current stack
          - Kernel stack: 0xc0000000 - 0xffffffff range
          - User stack: 0x00000000 - 0xbfffffff range
          
R14 (LR): Link Register  
          - Return address when function called
          - Use "BL" instruction: LR = PC + 4
          - "BX LR" returns to caller
          
R15 (PC): Program Counter
          - Address of current instruction + 8 (ARM mode)
          - Address of current instruction + 4 (Thumb mode)
          - Shows WHERE the crash occurred
```

### Program Status Register (PSR)

```
  31  30  29  28  27           24 23            16 15-8  7  6  5  4-0
 +---+---+---+---+---------------+---------------+-----+---+---+---+-----+
 | N | Z | C | V |   Reserved    |    GE[3:0]    | IT  | J | DNM| E | M  |
 +---+---+---+---+---------------+---------------+-----+---+---+---+-----+
  ^   ^   ^   ^                                          ^           ^
  |   |   |   |                                          |           |
  |   |   |   +-- oVerflow                              FIQ       Mode bits
  |   |   +------ Carry                                  disable
  |   +---------- Zero                                  
  +-------------- Negative
```

#### Condition Flags (Bits 28-31)
```
N (Negative)  : Set if result is negative (bit 31 of result = 1)
Z (Zero)      : Set if result is zero
C (Carry)     : Set if carry/borrow from addition/subtraction
V (oVerflow)  : Set if signed overflow occurred
```

#### Mode Bits (Bits 0-4)
```
0x10 (10000) : User mode       (USR) - Normal program execution
0x11 (10001) : FIQ mode        (FIQ) - Fast Interrupt
0x12 (10010) : IRQ mode        (IRQ) - Normal Interrupt  
0x13 (10011) : Supervisor mode (SVC) - Kernel mode, system calls
0x17 (10111) : Abort mode      (ABT) - Memory access violation
0x1B (11011) : Undefined mode  (UND) - Undefined instruction
0x1F (11111) : System mode     (SYS) - Privileged user mode
```

#### Interrupt Flags (Bits 6-7)
```
Bit 7 (I): IRQ disable
           0 = IRQs enabled
           1 = IRQs disabled
           
Bit 6 (F): FIQ disable  
           0 = FIQs enabled
           1 = FIQs disabled
```

### Example PSR Decoding

**PSR: 0x60000013**
```
Binary: 0110 0000 0000 0000 0000 0000 0001 0011

Flags [31-28]: 0110
  N=0 (positive result)
  Z=1 (zero result)
  C=1 (carry set)
  V=0 (no overflow)
  
Interrupts [7-6]: 00
  IRQs enabled
  FIQs enabled
  
Mode [4-0]: 10011 = 0x13
  Supervisor mode (kernel)
```

### Memory Address Ranges

#### Virtual Memory Layout (ARM32)
```
0x00000000 - 0xbfffffff : User space (3GB)
  0x00000000 - 0x00000fff : NULL guard page (unmapped)
  0x00001000 - 0xbfffffff : User accessible
  
0xc0000000 - 0xffffffff : Kernel space (1GB)
  0xc0000000 - 0xc0ffffff : Kernel image (.text, .data, .bss)
  0xc1000000 - 0xefffffff : vmalloc/ioremap area
  0xf0000000 - 0xffffffff : High vectors, fixmap
```

#### Identifying Pointers by Address
```
0x00000000           : NULL pointer ❌
0x00001000 - 0x1000  : User space (if in user mode)
0xc0000000 - 0xc0ff  : Kernel .text section (code)
0xc1000000 - 0xc1ff  : Kernel .data/.bss (global variables)
0xc2000000 - 0xcfff  : vmalloc area (kmalloc, vmalloc)
0xf0000000+          : I/O mapped hardware registers
```

### Common Error Codes

#### Oops Error Codes
```
Bit 0 (Write):    0 = Read access, 1 = Write access
Bit 1 (User):     0 = Kernel mode, 1 = User mode
Bit 2 (Present):  1 = Page not present, 0 = Page protection fault
Bit 3 (Reserved): 1 = Reserved bit violation

Common values:
  0x5 (0101) : Kernel read from unmapped page
  0x7 (0111) : Kernel write to unmapped page  
  0x4 (0100) : Kernel read from protected page
  0x6 (0110) : Kernel write to protected page
```

### Stack Trace Reading Tips

#### Understanding Stack Dump
```
Stack: (0xc3c1bd48 to 0xc3c1c000)
bd40:                   c23fc814 c114e4e4 c17a7459 c17a7400
      ^^^^^^^^^^^^^^^^  ^^^^^^^^ ^^^^^^^^ ^^^^^^^^ ^^^^^^^^
      Current SP+offset 8 bytes  8 bytes  8 bytes  8 bytes
```

- Each line shows 8 words (32 bytes)
- Stack grows downward (high to low addresses)
- Look for:
  - Return addresses (LR values, in kernel text: 0xc0-0xc1 range)
  - Function arguments (first values pushed)
  - Local variables
  - NULL pointers (0x00000000)

#### Finding Return Path
```
Look for addresses in kernel text range:
c0xxxxxx or c1xxxxxx = Likely return addresses
bfxxxxxx             = Module addresses
00000000             = NULL (bug!)
cxxxxxxx (other)     = Data pointers
```

### Quick Register Checklist

When analyzing a crash:

1. **Check PC**: Where did it crash?
   ```
   PC is at strcmp+0x4/0x34  ← In strcmp, 4 bytes from start
   ```

2. **Check LR**: Who called it?
   ```
   LR is at kset_find_obj+0x3c  ← Called from kset_find_obj
   ```

3. **Check function arguments (R0-R3)**:
   ```
   r0 : c17a7459  ← 1st arg - valid?
   r1 : 00000000  ← 2nd arg - NULL! Bug found!
   ```

4. **Check PSR mode bits**:
   ```
   psr: 60000013
        Mode = 0x13 = Supervisor (kernel mode) ✓
   ```

5. **Check stack pointer**:
   ```
   sp : c3c1bd48  ← Kernel stack (0xc... range) ✓
   ```

6. **Look for NULL pointers anywhere**:
   ```
   Any register = 0x00000000? ← Potential bug!
   ```

---

## Pattern 5: GPIO Request Fails with -EPROBE_DEFER in module_init()

### Symptoms
```bash
# Loading module
$ sudo insmod my_module.ko
insmod: ERROR: could not insert module my_module.ko: Unknown symbol in module

# dmesg shows
[  123.456] my_module: gpio_request failed: -517
[  123.457] my_module: module init failed
```

### Error Signature
```
gpio_request() returns -517 (-EPROBE_DEFER)
Module init fails
But: echo 31 > /sys/class/gpio/export WORKS after boot
```

### Root Cause

**GPIO controller not ready when `module_init()` executes**

Timing issue:
```
Boot sequence:
├─ 1. Kernel core init
├─ 2. module_init() runs early  ← GPIO controller may not be loaded yet
│     └─> gpio_request() → -EPROBE_DEFER (517)
│     └─> module_init returns error → module fails to load
├─ 3. GPIO controller driver loads
└─ 4. After boot complete
      └─> sysfs export works (controller ready now)
```

### Why This Happens

```c
// drivers/gpio/gpiolib-legacy.c
int gpio_request(unsigned gpio, const char *label)
{
    struct gpio_desc *desc = gpio_to_desc(gpio);
    
    /* GPIO valid but controller not loaded yet */
    if (!desc && gpio_is_valid(gpio))
        return -EPROBE_DEFER;  // ← Error 517
    
    return gpiod_request(desc, label);
}
```

**Problem**: `module_init()` cannot handle `-EPROBE_DEFER`:
- If probe() returns `-EPROBE_DEFER` → kernel retries later ✓
- If module_init() returns `-EPROBE_DEFER` → module load fails ✗

### Wrong Code Example

```c
#include <linux/module.h>
#include <linux/gpio.h>

static int __init my_init(void)
{
    int ret;
    
    // GPIO controller might not be ready yet!
    ret = gpio_request(31, "my-gpio");
    if (ret) {
        pr_err("gpio_request failed: %d\n", ret);
        return ret;  // ❌ Module load fails with -EPROBE_DEFER
    }
    
    gpio_direction_input(31);
    return 0;
}

module_init(my_init);
```

### Diagnostic Steps

**1. Check if GPIO controller is loaded:**
```bash
# Method 1: Check sysfs
ls /sys/class/gpio/
# If you see gpiochip*, controller is ready
# If empty or just export/unexport, controller not ready

# Method 2: Check debugfs
sudo cat /sys/kernel/debug/gpio
# Should show gpiochip entries

# Method 3: Check modules
lsmod | grep gpio
# Look for gpio controller module (gpio_omap, etc.)
```

**2. Check GPIO numbering:**
```bash
# Modern kernels use base != 0
$ ls /sys/class/gpio/
gpiochip512  gpiochip544  gpiochip576  # Base starts at 512!

# GPIO "31" doesn't exist - should be 543 or 575
```

**3. Check timing:**
```bash
# Try loading module after boot
sudo insmod my_module.ko  # May fail

# Wait and retry
sleep 5
sudo insmod my_module.ko  # May succeed (controller loaded)
```

### Solutions

#### Solution 1: Use Platform Driver (RECOMMENDED)

```c
#include <linux/module.h>
#include <linux/platform_device.h>
#include <linux/gpio/consumer.h>

struct my_data {
    struct gpio_desc *gpio;
};

static int my_probe(struct platform_device *pdev)
{
    struct my_data *priv;
    
    priv = devm_kzalloc(&pdev->dev, sizeof(*priv), GFP_KERNEL);
    if (!priv)
        return -ENOMEM;
    
    /* This handles EPROBE_DEFER automatically */
    priv->gpio = devm_gpiod_get(&pdev->dev, "reset", GPIOD_IN);
    if (IS_ERR(priv->gpio)) {
        if (PTR_ERR(priv->gpio) == -EPROBE_DEFER) {
            dev_info(&pdev->dev, "GPIO not ready, deferring\n");
            return -EPROBE_DEFER;  // ✓ Kernel will retry
        }
        return PTR_ERR(priv->gpio);
    }
    
    platform_set_drvdata(pdev, priv);
    dev_info(&pdev->dev, "Probed successfully\n");
    return 0;
}

static int my_remove(struct platform_device *pdev)
{
    return 0;  // devm_* handles cleanup
}

static struct platform_driver my_driver = {
    .probe = my_probe,
    .remove = my_remove,
    .driver = {
        .name = "my-gpio-driver",
    },
};

static struct platform_device *my_pdev;

static int __init my_init(void)
{
    int ret;
    
    ret = platform_driver_register(&my_driver);
    if (ret)
        return ret;
    
    my_pdev = platform_device_register_simple("my-gpio-driver", 
                                               -1, NULL, 0);
    if (IS_ERR(my_pdev)) {
        platform_driver_unregister(&my_driver);
        return PTR_ERR(my_pdev);
    }
    
    return 0;
}

static void __exit my_exit(void)
{
    platform_device_unregister(my_pdev);
    platform_driver_unregister(&my_driver);
}

module_init(my_init);
module_exit(my_exit);
MODULE_LICENSE("GPL");
```

#### Solution 2: Use late_initcall (Quick workaround)

```c
#include <linux/module.h>
#include <linux/gpio.h>

static int gpio_num;
static int irq_num;

static int __init my_init(void)
{
    int ret;
    
    ret = gpio_request(575, "my-gpio");  // Note: correct number!
    if (ret) {
        pr_err("gpio_request failed: %d\n", ret);
        return ret;
    }
    
    ret = gpio_direction_input(575);
    if (ret) {
        gpio_free(575);
        return ret;
    }
    
    gpio_num = 575;
    pr_info("Module loaded\n");
    return 0;
}

static void __exit my_exit(void)
{
    gpio_free(gpio_num);
}

// ✓ Load later in boot sequence
late_initcall(my_init);  // Instead of module_init()
module_exit(my_exit);
MODULE_LICENSE("GPL");
```

#### Solution 3: Fix GPIO Number (If Using Legacy API)

```c
// Check actual GPIO base on your system
// ls /sys/class/gpio/
// gpiochip512 means base=512

// For GPIO1_31 on BeagleBone:
// Bank 1 base = 544, pin 31
#define CORRECT_GPIO_NUM  (544 + 31)  // = 575

static int __init my_init(void)
{
    int ret = gpio_request(CORRECT_GPIO_NUM, "my-gpio");
    // Better chance of success with correct number
    ...
}
```

### Verification

```bash
# After fixing:
$ sudo insmod my_module.ko
$ dmesg | tail
[  456.789] my-gpio-driver: Probing
[  456.790] my-gpio-driver: GPIO not ready, deferring  # 1st attempt
[  456.791] my-gpio-driver: Probing                     # 2nd attempt (retry)
[  456.792] my-gpio-driver: Probed successfully         # Success!

# Check driver binding
$ ls /sys/bus/platform/drivers/my-gpio-driver/
my-gpio-driver.0  bind  module  uevent  unbind
```

### Key Takeaways

| | module_init() | Platform probe() |
|---|---|---|
| EPROBE_DEFER handling | ✗ Module fails | ✓ Auto retry |
| GPIO timing | ✗ May be too early | ✓ Handles dependencies |
| Cleanup | Manual gpio_free() | ✓ devm_* auto |
| **Recommended** | Only for simple cases | **Always for GPIO** |

### Related Errors

- **-EINVAL**: GPIO number doesn't exist (wrong base/offset)
- **-EBUSY**: GPIO already requested by another driver
- **-ENODEV**: GPIO chip not registered
- **-517 (-EPROBE_DEFER)**: GPIO controller not ready yet ← This pattern

---

## Memory Bank Update Log

- **2026-02-05**: Initial creation with Pattern 1-4
  - NULL pointer in driver registration
  - GPIO chip registration errors
  - Probe not called issues
  - Printk visibility problems
  - Added complete crash logs with register dumps
  - Added detailed ARM register analysis guide
  - Added stack trace deep dive
  - Added instruction-level debugging

- **2026-03-18**: Added Pattern 5
  - GPIO request fails with -EPROBE_DEFER in module_init()
  - GPIO controller timing and auto-base allocation (512+)
  - Platform driver vs module_init comparison
  - Solutions: platform driver, late_initcall, GPIO number fixes
