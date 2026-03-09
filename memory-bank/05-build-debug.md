# Build System & Debugging

## Build Commands

### Basic Build
```bash
# Show all available targets
make help

# Default build (vmlinux + modules)
make

# Build only kernel image
make vmlinux

# Build only modules
make modules

# Install modules
sudo make modules_install

# Install kernel (architecture dependent)
sudo make install
```

### Configuration
```bash
# Default configuration
make defconfig

# Architecture-specific default
make ARCH=arm multi_v7_defconfig
make ARCH=arm64 defconfig

# Interactive configuration
make menuconfig          # Text-based UI (ncurses)
make xconfig            # Qt-based UI
make gconfig            # GTK-based UI
make nconfig            # Better text UI

# Show new config options
make listnewconfig

# Update old config
make oldconfig
make olddefconfig       # Use defaults for new options

# Configuration scripts
scripts/config --enable CONFIG_MY_DRIVER
scripts/config --disable CONFIG_MY_DRIVER
scripts/config --module CONFIG_MY_DRIVER
scripts/config --set-str CONFIG_VERSION "1.0"
scripts/config --set-val CONFIG_COUNT 100

# Compare configs
scripts/diffconfig .config.old .config
```

### Cross-Compilation (ARM/BeagleBone)
```bash
# Set architecture and cross-compiler
export ARCH=arm
export CROSS_COMPILE=arm-linux-gnueabihf-

# Or inline
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- zImage
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- dtbs
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- modules

# Build targets for ARM
make ARCH=arm zImage          # Compressed kernel
make ARCH=arm uImage          # U-Boot image
make ARCH=arm dtbs            # Device tree blobs
```

### Building Specific Components
```bash
# Build specific directory
make drivers/net/
make fs/ext4/

# Build single file
make drivers/net/mydriver.o
make drivers/net/mydriver.ko

# Build external module
make M=drivers/staging/mydriver
make M=/path/to/external/module

# Clean specific module
make M=drivers/staging/mydriver clean
```

### Build Options
```bash
# Enable extra warnings
make W=1                # Enable extra gcc warnings
make W=2                # More warnings
make W=3                # Even more warnings

# Run static analysis
make C=1                # Run sparse on re-compiled files
make C=2                # Run sparse on all files
make CHECK=sparse       # Specify checker

# Verbose build
make V=1                # Show full command lines
make V=2                # Show reason for rebuild

# Parallel build
make -j$(nproc)         # Use all CPU cores
make -j8                # Use 8 parallel jobs

# Redirect build output
make O=/path/to/output  # Build in different directory
```

### Cleaning
```bash
# Clean most generated files
make clean

# Remove all generated files + config
make mrproper

# Remove editor backup files, patches, etc.
make distclean

# Clean specific module
make M=drivers/mydriver clean
```

## Code Quality Tools

### checkpatch.pl (MANDATORY)
```bash
# Check single file
scripts/checkpatch.pl --file drivers/mydriver/myfile.c

# Strict checking
scripts/checkpatch.pl --strict --file myfile.c

# Check patch file
scripts/checkpatch.pl mypatch.patch

# Check git commit
scripts/checkpatch.pl -g HEAD
scripts/checkpatch.pl -g HEAD~5..HEAD

# Show only specific error types
scripts/checkpatch.pl --types=SPACING,BRACES myfile.c

# Ignore specific warnings
scripts/checkpatch.pl --ignore=LINE_LENGTH myfile.c

# Output in different formats
scripts/checkpatch.pl --terse myfile.c
scripts/checkpatch.pl --emacs myfile.c
```

### Sparse (Static Analysis)
```bash
# Build with sparse
make C=1 CHECK=sparse

# Specific warnings
make C=2 CF="-Wsparse-all"
make C=2 CF="-Wbitwise"

# Check endianness issues
make C=2 CF="-Wbitwise -Wcontext"
```

### Coccinelle (Semantic Patches)
```bash
# Run all semantic patches
make coccicheck

# Run specific script
make coccicheck COCCI=scripts/coccinelle/api/alloc_cast.cocci

# Different modes
make coccicheck MODE=report    # Report problems
make coccicheck MODE=patch     # Generate patches
make coccicheck MODE=org       # Org-mode output
```

## Debugging Techniques

### Kernel Logging
```c
// Priority levels (highest to lowest)
pr_emerg("System is unusable\n");
pr_alert("Action must be taken immediately\n");
pr_crit("Critical conditions\n");
pr_err("Error conditions\n");
pr_warn("Warning conditions\n");
pr_notice("Normal but significant\n");
pr_info("Informational\n");
pr_debug("Debug-level messages\n");

// Dynamic debug (enable at runtime)
pr_debug("Value: %d\n", val);

// Enable dynamic debug via kernel command line
dyndbg="file drivers/mydriver/* +p"

// Enable at runtime via debugfs
echo 'file drivers/mydriver/myfile.c +p' > /sys/kernel/debug/dynamic_debug/control
echo 'func my_function +p' > /sys/kernel/debug/dynamic_debug/control

// Rate limiting
pr_info_ratelimited("Frequent message\n");
printk_ratelimited(KERN_INFO "Message\n");

// Once only
pr_info_once("This prints only once\n");
```

### Reading Kernel Logs
```bash
# View kernel ring buffer
dmesg

# Follow new messages
dmesg -w
dmesg --follow

# Clear buffer
dmesg -c

# Show with timestamps
dmesg -T

# Filter by facility/level
dmesg -l err,warn
dmesg -f kern

# Using journalctl
journalctl -k              # Kernel messages
journalctl -k -f           # Follow
journalctl -k -b           # Current boot
journalctl -k -b -1        # Previous boot
```

### Stack Traces
```c
// Print stack trace
dump_stack();

// WARN with stack trace
WARN(condition, "Error occurred\n");
WARN_ON(condition);
WARN_ON_ONCE(condition);

// BUG (use sparingly - kills kernel!)
BUG_ON(critical_condition);
```

### Memory Debugging
```bash
# Enable in kernel config
CONFIG_DEBUG_KMEMLEAK=y          # Memory leak detection
CONFIG_KASAN=y                   # Address sanitizer
CONFIG_SLUB_DEBUG=y              # SLUB debugging
CONFIG_DEBUG_PAGEALLOC=y         # Page allocation debugging
CONFIG_DEBUG_OBJECTS=y           # Object debugging

# Check for memory leaks (at runtime)
echo scan > /sys/kernel/debug/kmemleak
cat /sys/kernel/debug/kmemleak

# Clear kmemleak results
echo clear > /sys/kernel/debug/kmemleak
```

### Debugging Tools
```bash
# List loaded modules
lsmod

# Module info
modinfo mymodule.ko

# Load module with debugging
insmod mymodule.ko debug=1

# Module parameters at runtime
cat /sys/module/mymodule/parameters/debug
echo 1 > /sys/module/mymodule/parameters/debug

# Check for oops/panic
cat /proc/sys/kernel/panic
cat /proc/sys/kernel/panic_on_oops

# System request keys (Magic SysRq)
echo 1 > /proc/sys/kernel/sysrq
echo t > /proc/sysrq-trigger    # Dump tasks
echo m > /proc/sysrq-trigger    # Memory info
echo w > /proc/sysrq-trigger    # Blocked tasks
```

### GDB with Kernel
```bash
# Build with debug info
scripts/config --enable CONFIG_DEBUG_INFO
scripts/config --enable CONFIG_DEBUG_INFO_DWARF4
scripts/config --enable CONFIG_GDB_SCRIPTS
make

# Start GDB
gdb vmlinux

# Connect to QEMU
(gdb) target remote :1234

# Load kernel module symbols
(gdb) lx-symbols

# Common commands
(gdb) bt              # Backtrace
(gdb) list            # Show source
(gdb) break function  # Set breakpoint
(gdb) continue        # Continue execution
```

### ftrace
```bash
# Enable function tracing
echo function > /sys/kernel/debug/tracing/current_tracer

# Trace specific function
echo my_function > /sys/kernel/debug/tracing/set_ftrace_filter

# View trace
cat /sys/kernel/debug/tracing/trace

# Function graph tracing
echo function_graph > /sys/kernel/debug/tracing/current_tracer

# Events tracing
echo 1 > /sys/kernel/debug/tracing/events/irq/enable
```

## Testing

### Module Testing
```bash
# Load module
insmod mymodule.ko

# Load with parameters
insmod mymodule.ko debug=1 param=value

# Remove module
rmmod mymodule

# Force remove (dangerous!)
rmmod -f mymodule

# Check module is loaded
lsmod | grep mymodule
```

### KUnit Testing
```bash
# Run all tests
./tools/testing/kunit/kunit.py run

# Run specific test
./tools/testing/kunit/kunit.py run --kunitconfig=drivers/mydriver/

# Build without running
./tools/testing/kunit/kunit.py build

# Config for KUnit
CONFIG_KUNIT=y
CONFIG_KUNIT_TEST=y
```

### Selftests
```bash
# Build selftests
make -C tools/testing/selftests

# Run specific test suite
make -C tools/testing/selftests TARGETS=net run_tests
make -C tools/testing/selftests TARGETS=bpf run_tests

# Install selftests
make -C tools/testing/selftests install INSTALL_PATH=/tmp/tests
```
