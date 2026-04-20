# Tools — Memory Bank

This directory contains development tools for the Linux kernel workspace.
Run all commands from the **kernel root directory** (`~/linux_doc/linux/`).

---

## `gen-compile-commands.py` — Generate `compile_commands.json` for clangd

This tool generates a `compile_commands.json` file so that `clangd` can accurately
trace kernel source code **for the correct target platform/board**
(e.g. x86_64, BeagleBone Black, Raspberry Pi, etc.).

> **Why is this tool needed?**
> Linux Kernel 4.x does not support `make compile_commands.json` (only available from 5.x onward).
> This tool wraps `bear` to intercept the build process and produce the equivalent JSON database.
> Additionally, `bear 3.x` defaults to **merging** new entries into an existing file — this tool
> explicitly handles that to prevent cross-architecture contamination.

---

## System Requirements

```bash
# Install all required tools (one-time setup)
sudo apt install -y \
    build-essential gcc flex bison \
    libssl-dev libelf-dev \
    bear clangd
```

For **BeagleBone Black** (ARM cross-compilation), install the cross-compiler:
```bash
sudo apt install -y gcc-arm-linux-gnueabihf
```

---

## Step-by-Step Guide

### Step 1 — List supported boards

```bash
python3 .github/memory-bank/tools/gen-compile-commands.py --list
```

Example output:
```
  Board                Arch       defconfig                    Description
  -------------------- ---------- ---------------------------- ----------------------------------------
  x86_64               x86_64     x86_64_defconfig             x86-64 (native build, no cross-compiler)
  beaglebone-black     arm        omap2plus_defconfig          BeagleBone Black (TI AM335x, ARM Cortex-A8)
                                  CROSS_COMPILE=arm-linux-gnueabihf-
  rpi3                 arm        multi_v7_defconfig           Raspberry Pi 3 (ARM Cortex-A53, 32-bit)
  rpi4                 arm64      defconfig                    Raspberry Pi 4 (ARM Cortex-A72, 64-bit)
  qemu-arm             arm        vexpress_defconfig           QEMU ARM vexpress (Cortex-A9)
  qemu-arm64           arm64      defconfig                    QEMU AArch64
  qemu-riscv64         riscv      defconfig                    QEMU RISC-V 64-bit
```

---

### Step 2 — Preview commands (without executing)

Use `--dry-run` to inspect what the tool will do without actually building:

```bash
python3 .github/memory-bank/tools/gen-compile-commands.py \
    --board beaglebone-black \
    --dry-run
```

---

### Step 3 — Run the build to generate `compile_commands.json`

#### Case A: Trace x86_64 kernel (native host)

```bash
python3 .github/memory-bank/tools/gen-compile-commands.py \
    --board x86_64
```

#### Case B: Trace BeagleBone Black kernel (ARM Cortex-A8)

```bash
python3 .github/memory-bank/tools/gen-compile-commands.py \
    --board beaglebone-black
```

> ⏱ The build process takes approximately **15–30 minutes** depending on CPU speed.

#### Case C: Clean stale build artifacts before rebuilding

```bash
python3 .github/memory-bank/tools/gen-compile-commands.py \
    --board beaglebone-black \
    --clean
```

#### Case D: Keep separate JSON files per board

```bash
# Generate and save to board-specific files
python3 .github/memory-bank/tools/gen-compile-commands.py \
    --board x86_64 \
    --output compile_commands_x86.json

python3 .github/memory-bank/tools/gen-compile-commands.py \
    --board beaglebone-black \
    --output compile_commands_bbb.json

# Switch active clangd database by copying the desired file
cp compile_commands_bbb.json compile_commands.json
```

---

### Step 4 — Verify the output

On success, the tool prints a summary:

```
============================================================
  ✅ SUCCESS!
  Output  : /home/ntai/linux_doc/linux/compile_commands.json
  Entries : 156,700 compilation units
  Size    : 4.2 MB
============================================================

  clangd will now trace arm kernel code correctly.
  Open any .c file → F12 (Go to Definition) to verify.
```

Quick verification:
```bash
ls -lh compile_commands.json
```

---

### Step 5 — Verify clangd is working

1. Open any kernel `.c` file (e.g. `drivers/gpio/gpio-omap.c`)
2. Hover over a function name → A tooltip with the function signature should appear
3. Press **`F12`** → Jump to the exact definition (Go to Definition)
4. Press **`Shift+F12`** → List all call sites (Find All References)

If clangd is working correctly, there should be **no false red errors** on kernel headers
and all navigation should resolve to the correct architecture-specific files.

---

## Options Reference

| Option | Description | Example |
|--------|-------------|---------|
| `--board` / `-b` | Select a board preset | `--board beaglebone-black` |
| `--list` / `-l` | List all supported boards and exit | `--list` |
| `--output` / `-o` | Output file path | `-o compile_commands_bbb.json` |
| `--jobs` / `-j` | Number of parallel build jobs | `-j8` |
| `--clean` | Run `make clean` before building | `--clean` |
| `--no-backup` | Delete old JSON file instead of backing it up | `--no-backup` |
| `--dry-run` | Print commands without executing | `--dry-run` |
| `--compiler` | Choose `gcc` (default) or `clang` | `--compiler clang` |
| `--arch` | Override target CPU architecture | `--arch arm` |
| `--cross-compile` | Override `CROSS_COMPILE` prefix | `--cross-compile arm-linux-gnueabihf-` |
| `--defconfig` | Override defconfig target | `--defconfig omap2plus_defconfig` |
| `--kcflags` | Append extra compiler flags | `--kcflags "-O0"` |

---

## Known Errors & Fixes

| Error | Root Cause | Fix |
|-------|-----------|-----|
| `Command 'bear' not found` | `bear` is not installed | `sudo apt install -y bear` |
| `cc1: error: code model kernel does not support PIC mode` | Ubuntu GCC enables PIC by default | Handled automatically via `-fno-pic` |
| `undefined reference to '__stack_chk_fail'` | GCC 13+ enables stack-protector by default | Handled automatically via `-fno-stack-protector` |
| `arm-linux-gnueabihf-gcc: not found` | ARM cross-compiler not installed | `sudo apt install -y gcc-arm-linux-gnueabihf` |
| `Unknown board: 'xyz'` | Invalid board name | Run `--list` to see valid names |
| `[ERROR] compile_commands.json was not generated` | Build failed mid-way | Review error output above; retry with `--clean` |

---

## How It Works

```
gen-compile-commands.py
│
├── Check dependencies (bear, make, cross-compiler)
├── [--clean] Run: make clean
├── Backup and remove old compile_commands.json     ← Prevents bear 3.x cross-arch merge
├── Run: make <defconfig> ARCH=<arch>               ← Configure kernel for target platform
└── Run: bear --output tmp.json -- make -j<N> ...   ← Build + capture compilation database
    └── Move tmp.json → compile_commands.json
```

> **Why back up the old file?**
> `bear 3.x` merges new entries into any existing `compile_commands.json` rather than
> overwriting it. Without removing the old x86_64 file first, building for BeagleBone Black
> would produce a mixed database with entries from both architectures, causing clangd to
> resolve symbols to the wrong architecture-specific files.

---

## See Also

- [`05-build-debug.md`](../05-build-debug.md) — Build system details, debugging techniques, and known errors
- [`copilot-instructions.md`](../../copilot-instructions.md) — Main workspace development instructions
