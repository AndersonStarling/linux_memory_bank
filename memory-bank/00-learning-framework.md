# Linux Kernel Learning Framework

> **Nguyên tắc cốt lõi**: Khi nghiên cứu bất kỳ subsystem/tính năng nào trong Linux kernel, luôn luôn tìm hiểu **mối liên hệ giữa userspace và kernel** - làm cách nào user có thể tương tác xuống kernel và hardware.
> 
> **LUẬT BẤT BIẾN**: **TUYỆT ĐỐI KHÔNG ĐƯỢC SỬA SOURCE CODE TRONG KERNEL**. Mục tiêu là nghiên cứu và tracing, không phải sửa lỗi kernel. Mọi vấn đề về build phải được giải quyết bằng compiler flags hoặc đổi compiler tương thích.
>
> **QUY TẮC TRÌNH BÀY**: KHÔNG DÙNG ICON KHI UPDATE TÀI LIỆU.

---

## Framework Phân Tích Chuẩn

Mọi kernel subsystem đều tuân theo kiến trúc phân lớp từ userspace xuống hardware:

```
┌─────────────────────────────────────────────────────┐
│  USERSPACE (User Application)                       │
│  - CLI tools, libraries, applications               │
│  - What user sees and interacts with                │
└────────────────┬────────────────────────────────────┘
                 │ System calls / ioctl / sysfs / device files
┌────────────────▼────────────────────────────────────┐
│  USERSPACE INTERFACE LAYER                          │
│  - Character device (/dev/*)                        │
│  - Sysfs attributes (/sys/class/*)                  │
│  - Procfs (/proc/*)                                 │
│  - ioctl handlers                                   │
└────────────────┬────────────────────────────────────┘
                 │ Kernel API calls
┌────────────────▼────────────────────────────────────┐
│  CORE FRAMEWORK (Subsystem Core)                    │
│  - gpiolib.c, clk-core.c, regulator-core.c, etc.    │
│  - Descriptor/handle management                     │
│  - Common API implementation                        │
│  - Device/resource registration                     │
└────────────────┬────────────────────────────────────┘
                 │ Driver callbacks (ops)
┌────────────────▼────────────────────────────────────┐
│  HARDWARE DRIVER LAYER (Chip-specific)              │
│  - gpio-omap.c, clk-ti.c, etc.                      │
│  - Implements ops callbacks                         │
│  - Direct hardware access                           │
│  - Clock/PM/pinmux management                       │
└────────────────┬────────────────────────────────────┘
                 │ Register read/write
┌────────────────▼────────────────────────────────────┐
│  HARDWARE (Registers, Pins, Peripherals)            │
│  - Memory-mapped registers                          │
│  - Physical GPIO pins, clocks, regulators, etc.     │
└─────────────────────────────────────────────────────┘
```

---

## Research Checklist

Khi nghiên cứu một subsystem mới, trả lời **7 câu hỏi cốt lõi** theo thứ tự:

### 1. **Userspace Interface** - User dùng gì?

**Câu hỏi:**
- User application tương tác với subsystem này như thế nào?
- CLI tools nào có sẵn?
- Libraries nào hỗ trợ?
- Device files hoặc sysfs paths là gì?

**Ví dụ (GPIO):**
- CLI tools: `gpioget`, `gpioset`, `gpiomon`
- Library: `libgpiod` (C/Python)
- Device files: `/dev/gpiochip0`, `/dev/gpiochip1`
- Sysfs (legacy): `/sys/class/gpio/`

**Ví dụ (Clock):**
- CLI tools: `cat /sys/kernel/debug/clk/clk_summary`
- Debugfs: `/sys/kernel/debug/clk/`
- No direct userspace control (kernel-only API)

---

### 2. **Kernel Entry Point** - Vào kernel ở đâu?

**Câu hỏi:**
- System call / ioctl nào được sử dụng?
- File operations (open/read/write/ioctl/mmap) nào được implement?
- Sysfs attributes nào được expose?

**Ví dụ (GPIO):**
- Character device: `gpio_fileops` trong `gpiolib-cdev.c`
- IOCTL: `GPIO_V2_GET_LINE_IOCTL`, `GPIO_V2_LINE_SET_VALUES_IOCTL`
- File ops: `open`, `ioctl`, `read` (event notification), `poll`

**Pattern tìm kiếm:**
```bash
# Tìm file operations
grep -r "struct file_operations" drivers/gpio/
grep -r "\.unlocked_ioctl.*=" drivers/gpio/

# Tìm character device registration
grep -r "cdev_init\|cdev_add" drivers/gpio/
grep -r "device_create\|class_create" drivers/gpio/
```

---

### 3. **Core Framework** - Logic trung tâm ở đâu?

**Câu hỏi:**
- File nào chứa core implementation?
- Structures chính là gì? (`struct gpio_chip`, `struct clk`, ...)
- API registration/deregistration?
- API consumer sử dụng?

**Ví dụ (GPIO):**
- Core file: `drivers/gpio/gpiolib.c`
- Main structure: `struct gpio_chip`, `struct gpio_desc`
- Registration: `gpiochip_add_data()`, `gpiochip_remove()`
- Consumer API: `gpiod_get()`, `gpiod_set_value()`, `gpiod_direction_input()`

**Pattern tìm kiếm:**
```bash
# Tìm core files
ls drivers/gpio/gpiolib*.c
ls drivers/clk/clk*.c

# Tìm registration functions
grep -r "EXPORT_SYMBOL.*add\|EXPORT_SYMBOL.*register" drivers/gpio/gpiolib.c
```

---

### 4. **Hardware Driver** - Implement cụ thể như thế nào?

**Câu hỏi:**
- Structure `ops` callbacks nào cần implement?
- Functions nào là mandatory vs optional?
- Ví dụ driver reference nào tốt?

**Ví dụ (GPIO):**
- Structure: `struct gpio_chip { .get, .set, .direction_input, .direction_output, ... }`
- Mandatory: `.get`, `.set`, `.direction_input`, `.direction_output`
- Optional: `.to_irq`, `.set_config`, `.dbg_show`
- Reference driver: `drivers/gpio/gpio-omap.c` (OMAP/AM33xx)

**Pattern tìm kiếm:**
```bash
# Xem structure definition
grep -A 30 "^struct gpio_chip {" include/linux/gpio/driver.h

# Tìm example drivers
ls drivers/gpio/gpio-*.c | head -5
```

---

### 5. **Device Tree Binding** - DT integration?

**Câu hỏi:**
- Properties bắt buộc là gì?
- Properties optional nào hữu ích?
- Convention naming (e.g., `*-gpios`, `clocks`, `clock-names`)?

**Ví dụ (GPIO):**
```dts
/* Controller node */
gpio1: gpio@4804c000 {
    compatible = "ti,omap4-gpio";
    reg = <0x4804c000 0x1000>;
    gpio-controller;           /* Required */
    #gpio-cells = <2>;         /* Required: <offset flags> */
    interrupt-controller;      /* Optional */
    #interrupt-cells = <2>;    /* If interrupt-controller */
};

/* Consumer node */
my_device {
    reset-gpios = <&gpio1 5 GPIO_ACTIVE_LOW>;  /* Convention: name-gpios */
    enable-gpios = <&gpio1 12 GPIO_ACTIVE_HIGH>;
};
```

**Documentation:**
```bash
# Xem DT bindings
cat Documentation/devicetree/bindings/gpio/gpio-omap.txt
ls Documentation/devicetree/bindings/gpio/
```

---

### 6. **Power/Clock/Pinmux Management** - Dependencies?

**Câu hỏi:**
- Subsystem này cần clock nào? (`fck`, `dbclk`, ...)
- Runtime PM có được sử dụng?
- Pinmux configuration cần thiết?
- Regulator dependencies?

**Ví dụ (GPIO AM33xx):**
- Clock: `fck` (functional clock) - managed by `ti,sysc` parent
- Runtime PM: `pm_runtime_enable()` + `pm_runtime_get_sync()`
- Pinmux: Automatic via `gpio-ranges` property
- No regulator dependency

**Pattern:**
```c
// Check driver probe for dependencies
static int my_probe(struct platform_device *pdev)
{
    // Clock?
    clk = devm_clk_get(&pdev->dev, "fck");
    
    // Runtime PM?
    pm_runtime_enable(&pdev->dev);
    pm_runtime_get_sync(&pdev->dev);
    
    // Pinctrl?
    pinctrl_pm_select_default_state(&pdev->dev);
    
    // Regulator?
    regulator = devm_regulator_get(&pdev->dev, "vdd");
}
```

---

### 7. **Debugging & Verification** - Làm sao verify hoạt động?

**Câu hỏi:**
- Debugfs interfaces nào có sẵn?
- Sysfs attributes để inspect state?
- Tools để test?
- Common errors và cách fix?

**Ví dụ (GPIO):**
- Debugfs: `cat /sys/kernel/debug/gpio`
- Sysfs: `/sys/class/gpio/` (legacy)
- Device files: `ls /dev/gpiochip*`
- Test tools: `gpioinfo gpiochip0`, `gpioget gpiochip0 5`

**Debug commands:**
```bash
# Check driver loaded
lsmod | grep gpio
dmesg | grep -i gpio

# Check device registered
ls /sys/class/gpio/
ls /dev/gpiochip*

# Inspect state
cat /sys/kernel/debug/gpio
gpioinfo

# Enable dynamic debug
echo "file drivers/gpio/* +p" > /sys/kernel/debug/dynamic_debug/control
```

---

## Template Workflow

Khi bắt đầu học một subsystem mới:

### Phase 1: Userspace Understanding (30 minutes)

```bash
# 1. Tìm tools
which gpioget gpioset          # GPIO example
which pwmconfig                # PWM example

# 2. Test basic operations
gpiodetect
gpioinfo gpiochip0
gpioget gpiochip0 5

# 3. Trace system calls
strace gpioget gpiochip0 5 2>&1 | grep -E "open|ioctl"
# → Thấy open("/dev/gpiochip0"), ioctl(GPIO_V2_GET_LINE_IOCTL)

# 4. Check libraries
ldconfig -p | grep gpio
pkg-config --cflags --libs libgpiod
```

**Output:** Hiểu user perspective - tools, files, libraries.

---

### Phase 2: Kernel Entry Point Discovery (1 hour)

```bash
# 1. Tìm character device handler
cd /path/to/linux
grep -r "gpio_fileops\|gpio.*file_operations" drivers/gpio/

# 2. Đọc ioctl handler
vim drivers/gpio/gpiolib-cdev.c
# → Tìm `gpio_ioctl()`, xem cases

# 3. Map userspace → kernel
# gpioget → open(/dev/gpiochip0) → gpio_chrdev_open()
#        → ioctl(GPIO_V2_GET_LINE) → gpio_ioctl() → ...
```

**Output:** Flow diagram từ userspace vào kernel.

---

### Phase 3: Core Framework Study (2 hours)

```bash
# 1. Tìm core files
ls drivers/gpio/gpiolib*.c

# 2. Đọc main structures
vim include/linux/gpio/driver.h
# → Xem `struct gpio_chip`

# 3. Trace một operation
# gpiod_set_value() trong gpiolib.c
#   → gpiod_set_raw_value()
#   → chip->set()  ← callback vào driver
```

**Output:** Hiểu core API và callback mechanism.

---

### Phase 4: Driver Implementation (2-3 hours)

```bash
# 1. Đọc reference driver
vim drivers/gpio/gpio-omap.c

# 2. Xác định pattern
# - Probe: setup gpio_chip, register via gpiochip_add_data()
# - Implement callbacks: .get, .set, .direction_input/output
# - Cleanup: devm_* auto cleanup

# 3. Check hardware access
# - Register offsets
# - Clock management
# - Runtime PM integration
```

**Output:** Template driver implementation.

---

### Phase 5: Testing & Debugging (1 hour)

```bash
# 1. Build & load
make ARCH=arm drivers/gpio/gpio-mydriver.ko
insmod gpio-mydriver.ko

# 2. Verify registration
dmesg | tail
ls /dev/gpiochip*
cat /sys/kernel/debug/gpio

# 3. Test operations
gpioset gpiochip2 5=1
gpioget gpiochip2 5

# 4. Debug issues
echo 8 > /proc/sys/kernel/printk           # Enable debug
echo "file drivers/gpio/* +p" > /sys/kernel/debug/dynamic_debug/control
```

**Output:** Working driver + verification.

---

## Common Patterns Across Subsystems

| Subsystem | Userspace Tools | Device Files | Core File | Main Structure | Registration API |
|-----------|----------------|--------------|-----------|----------------|------------------|
| **GPIO** | gpioget/set/mon | /dev/gpiochipX | gpiolib.c | gpio_chip | gpiochip_add_data() |
| **Clock** | debugfs only | N/A | clk.c | clk_hw | clk_hw_register() |
| **PWM** | pwmconfig | /sys/class/pwm/ | core.c | pwm_chip | pwmchip_add() |
| **I2C** | i2cget/set/detect | /dev/i2c-X | i2c-core-base.c | i2c_adapter | i2c_add_adapter() |
| **SPI** | spidev_test | /dev/spidevX.Y | spi.c | spi_controller | spi_register_controller() |
| **Regulator** | debugfs | /sys/class/regulator/ | core.c | regulator_desc | regulator_register() |
| **IIO** | iio_* tools | /dev/iio:deviceX | industrialio-core.c | iio_dev | iio_device_register() |

---

## Anti-Patterns (Tránh)

❌ **Học driver trước khi hiểu userspace**
- Hậu quả: Không biết driver làm gì, test như thế nào
- ✅ Đúng: Luôn bắt đầu từ user perspective

❌ **Bỏ qua character device layer**
- Hậu quả: Không hiểu userspace → kernel mapping
- ✅ Đúng: Trace từ open/ioctl → core → driver

❌ **Copy-paste code không hiểu flow**
- Hậu quả: Driver work nhưng không maintain được
- ✅ Đúng: Vẽ flow diagram, hiểu từng bước

❌ **Ignore dependencies (clock/PM/pinmux)**
- Hậu quả: Driver crash vì register access khi clock OFF
- ✅ Đúng: Checklist dependencies trong probe

❌ **Không test từng bước**
- Hậu quả: Debug khó khi nhiều lỗi chồng chéo
- ✅ Đúng: Test incremental (probe → register → basic ops → advanced)

---

## Quick Reference: Flow Tracing Commands

```bash
# Trace system calls
strace -e trace=open,ioctl,read,write COMMAND 2>&1 | less

# Find ioctl definitions
grep -r "define.*GPIO.*IOCTL" include/uapi/

# Trace kernel functions
echo 'p:myprobe gpiod_set_value' > /sys/kernel/debug/tracing/kprobe_events
echo 1 > /sys/kernel/debug/tracing/events/kprobes/myprobe/enable
cat /sys/kernel/debug/tracing/trace

# Find structure definition
cscope -d -L -1 gpio_chip         # Tìm definition
cscope -d -L -0 gpiod_set_value   # Tìm references
```

---

## Memory Bank Organization

Mỗi subsystem memory bank file **PHẢI** bao gồm các sections sau (theo thứ tự):

1. Architecture Overview - Diagram phân lớp
2. Userspace Interface - Tools, libraries, device files
3. Character Device / Sysfs Layer - Entry points
4. Core Framework - Main logic, structures, APIs
5. Hardware Driver Layer - Callbacks, implementation pattern
6. Device Tree Integration - Bindings, examples
7. Dependencies - Clock, PM, pinmux requirements
8. Debugging & Verification - Tools, commands, common issues

**Example:**
- [07-gpio-subsystem.md](07-gpio-subsystem.md) - Follows this structure
- [08-pwm-subsystem.md](08-pwm-subsystem.md) - Follows this structure
- All future subsystem docs must follow

---

## Summary

> **Golden Rule:** Hiểu một subsystem = Hiểu flow từ user application → system call/ioctl → kernel framework → hardware driver → register access.

**Workflow:**
1. Start userspace → trace vào kernel
2. Identify entry points → map ioctl handlers
3. Study core framework → understand API
4. Read reference driver → implement pattern
5. Test & debug → verify end-to-end

Áp dụng framework này cho **MỌI** subsystem: GPIO ✅, Clock, Regulator, PWM, I2C, SPI, DMA, IIO, IRQ, etc.
