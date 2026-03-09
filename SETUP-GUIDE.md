# Linux Kernel Learning Setup Guide

> **Mục đích**: Hướng dẫn tích hợp Memory Bank vào kernel source của bạn để học Linux kernel với AI agent

---

## Tổng Quan

Repository này chứa **memory bank** - tài liệu tổng hợp kiến thức về Linux kernel subsystems. Bạn sẽ:

1. Clone kernel Linux (phiên bản bất kỳ)
2. Clone memory bank này
3. Tích hợp memory bank vào kernel source
4. Sử dụng AI agent để học kernel với context từ memory bank

**Cấu trúc cuối cùng:**
```
your-linux-workspace/
├── .git/                          # Track CHỈ memory bank
├── .gitignore                     # Ignore kernel source
├── .github/                       # Memory bank từ repo này
│   ├── copilot-instructions.md    # AI agent context
│   └── memory-bank/               # Tài liệu subsystems
│       ├── 00-learning-framework.md
│       ├── 07-gpio-subsystem.md
│       └── ...
└── (kernel files - git ignored)
    ├── Makefile
    ├── drivers/
    ├── arch/
    └── ...
```

---

## Bước 1: Clone Kernel Source

Chọn **MỘT** trong các cách sau:

```bash
# Option A: Phiên bản mới nhất
git clone --depth 1 https://git.kernel.org/torvalds/linux.git
cd linux

# Option B: Phiên bản cụ thể (ví dụ: 6.18-rc1)
git clone --depth 1 --branch v6.18-rc1 \
    https://git.kernel.org/torvalds/linux.git linux-6.18-rc1
cd linux-6.18-rc1

# Option C: Tarball (nếu không cần git)
wget https://cdn.kernel.org/pub/linux/kernel/v6.x/linux-6.18.tar.xz
tar xf linux-6.18.tar.xz
cd linux-6.18
```

---

## Bước 2: Clone Memory Bank

```bash
# Đang ở trong thư mục kernel
# Clone memory bank vào thư mục tạm
cd ..
git clone https://github.com/YOUR_USERNAME/linux-kernel-notes.git temp-memory-bank

# Copy thư mục .github vào kernel
cd linux  # hoặc linux-6.18-rc1
cp -r ../temp-memory-bank/.github .

# Xóa thư mục tạm
cd ..
rm -rf temp-memory-bank
```

**Windows PowerShell:**
```powershell
cd ..
git clone https://github.com/YOUR_USERNAME/linux-kernel-notes.git temp-memory-bank

cd linux
Copy-Item -Recurse ..\temp-memory-bank\.github .

cd ..
Remove-Item -Recurse -Force temp-memory-bank
```

---

## Bước 3: Setup Git để Track Chỉ Memory Bank

```bash
cd linux  # hoặc linux-6.18-rc1

# Tạo .gitignore
cat > .gitignore << 'EOF'
# Ignore TẤT CẢ kernel source files
/*

# CHỈ track memory bank
!.gitignore
!.github/
!README.md

# Trong .github, chỉ keep memory-bank
.github/*
!.github/memory-bank/
!.github/copilot-instructions.md
!.github/SETUP-GUIDE.md
EOF

# Windows: sử dụng notepad để tạo .gitignore với nội dung trên
```

---

## Bước 4: Initialize Git Repository

```bash
# Trong thư mục kernel
git init

# Kiểm tra
git status
# Chỉ thấy: .gitignore, .github/

# Commit
git add .gitignore .github/ README.md
git commit -m "Add kernel learning memory bank"

# Link với remote của BẠN (nếu muốn backup)
git remote add origin https://github.com/YOUR_USERNAME/my-kernel-learning.git
git branch -M main
git push -u origin main
```

---

## Bước 5: Mở VS Code và Hỏi Agent

```bash
# Mở workspace
code .
```

**Trong VS Code, thử hỏi agent:**

```
"What kernel version am I working with?"
"List all memory bank files"
"Explain GPIO subsystem using memory-bank/07-gpio-subsystem.md"
```

---

## Cách Sử Dụng Memory Bank

### 1. Khi Nghiên Cứu Subsystem Mới

```
"Explain I2C subsystem for Linux kernel.
Check memory-bank/09-i2c-subsystem.md for existing context.
Focus on userspace i2c-tools → i2c-dev.c → i2c-core.c flow."
```

### 2. Khi Cần Cập Nhật Memory Bank

```
"Add information about I2C SMBUS API to memory-bank/09-i2c-subsystem.md
Include code examples from drivers/i2c/i2c-core-smbus.c"
```

### 3. Khi Debug Code

```
"I'm seeing error in drivers/gpio/gpio-omap.c at line 850.
Check memory-bank/07-gpio-subsystem.md for context about OMAP GPIO clocks."
```

---

## Memory Bank Structure

```
.github/memory-bank/
├── 00-learning-framework.md   # ⭐ Phương pháp nghiên cứu kernel
│
├── Core Patterns (01-06)
│   ├── 01-core-patterns.md        # Module templates, headers
│   ├── 02-memory-management.md    # kmalloc, GFP flags
│   ├── 03-locking-concurrency.md  # Mutexes, spinlocks, RCU
│   ├── 04-device-drivers.md       # Platform drivers
│   ├── 05-build-debug.md          # Kbuild, debugging
│   └── 06-subsystems.md           # Integration patterns
│
└── I/O Subsystems (07-16)
    ├── 07-gpio-subsystem.md       # GPIO framework
    ├── 08-pwm-subsystem.md        # PWM framework
    ├── 09-i2c-subsystem.md        # I2C framework
    ├── 10-spi-subsystem.md        # SPI framework
    ├── 11-pinctrl-subsystem.md    # Pin multiplexing
    ├── 12-clock-subsystem.md      # Clock framework
    ├── 13-regulator-subsystem.md  # Voltage/current regulation
    ├── 14-dma-subsystem.md        # DMA engine
    ├── 15-iio-subsystem.md        # Industrial I/O
    └── 16-irq-subsystem.md        # Interrupt handling
```

---

## Tips & Best Practices

### ✅ DO

- **Luôn mở file memory bank trước khi hỏi về subsystem đó**
- **Mention memory bank file trong câu hỏi**: "Check memory-bank/07-gpio-subsystem.md"
- **Cập nhật memory bank khi học được điều mới**
- **Commit memory bank changes thường xuyên**

### ❌ DON'T

- **Đừng commit kernel source** - chỉ commit `.github/`
- **Đừng hỏi chung chung** - reference memory bank để có context
- **Đừng skip 00-learning-framework.md** - đọc methodology trước

---

## Troubleshooting

### Git tracking kernel source files

```bash
# Fix: Remove kernel files from git
git rm -r --cached .
git add .gitignore .github/
git commit -m "Fix: Only track memory bank"
```

### Agent không thấy memory bank

1. Mở file `.github/memory-bank/07-gpio-subsystem.md`
2. Mention explicitly: "Check memory-bank/07-gpio-subsystem.md"
3. Verify `.github/copilot-instructions.md` exists

---

## Quick Start (Tóm Tắt)

```bash
# 1. Clone kernel
git clone --depth 1 https://git.kernel.org/torvalds/linux.git
cd linux

# 2. Clone memory bank vào thư mục tạm
cd ..
git clone https://github.com/YOUR_USERNAME/linux-kernel-notes.git temp-mb
cd linux
cp -r ../temp-mb/.github .
rm -rf ../temp-mb

# 3. Setup .gitignore (chỉ track .github/)
cat > .gitignore << 'EOF'
/*
!.gitignore
!.github/
EOF

# 4. Initialize git
git init
git add .gitignore .github/
git commit -m "Add kernel memory bank"

# 5. Open VS Code
code .
```

---

## Resources

- **Kernel source**: https://git.kernel.org/torvalds/linux.git
- **Memory bank methodology**: `.github/memory-bank/00-learning-framework.md`
- **Agent instructions**: `.github/copilot-instructions.md`

---

**Last Updated**: February 9, 2026
