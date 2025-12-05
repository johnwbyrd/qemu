# ZBC (Zero Board Computer) for QEMU: Implementation Strategy

## Executive Summary

This document describes a strategy for implementing ZBC-style minimal test boards in QEMU, providing standardized test environments for all 14 QEMU-supported CPU architectures. Each ZBC board consists of a CPU, RAM, a memory-mapped semihosting device (using the RIFF-based ZBC protocol), and an MC6847-style text display.

The goal is upstream-quality code suitable for contribution to the QEMU project.

---

## Table of Contents

1. [Background and Motivation](#1-background-and-motivation)
2. [Target Architectures](#2-target-architectures)
3. [Component Overview](#3-component-overview)
4. [ZBC Semihosting Device](#4-zbc-semihosting-device)
5. [MC6847 Text Display Device](#5-mc6847-text-display-device)
6. [ZBC Machine Types](#6-zbc-machine-types)
7. [Memory Layout Strategy](#7-memory-layout-strategy)
8. [Binary Loading](#8-binary-loading)
9. [Interrupt Routing](#9-interrupt-routing)
10. [Console I/O](#10-console-io)
11. [Build System Integration](#11-build-system-integration)
12. [Testing Strategy](#12-testing-strategy)
13. [Documentation](#13-documentation)
14. [Upstream Submission Strategy](#14-upstream-submission-strategy)
15. [File Structure](#15-file-structure)
16. [Implementation Phases](#16-implementation-phases)
17. [Architecture-Specific Notes](#17-architecture-specific-notes)
18. [Open Questions and Risks](#18-open-questions-and-risks)

---

## 1. Background and Motivation

### 1.1 What is ZBC?

ZBC (Zero Board Computer) is a minimal, standardized test environment design that originated in MAME. It provides:

- A CPU with maximum available RAM
- A memory-mapped semihosting device for host I/O
- A simple text display (MC6847 VDG)
- Direct binary loading (no bootloader/firmware required)

The key insight is that most CPU testing and embedded development doesn't need a full system emulation—just a CPU, memory, a way to see output, and a way to interact with the host filesystem.

### 1.2 Why Port to QEMU?

QEMU is the dominant open-source emulator for systems development. While MAME excels at arcade/home computer accuracy, QEMU is the standard for:

- Operating system development
- Firmware development
- Cross-architecture testing
- CI/CD pipelines
- Embedded systems prototyping

Bringing ZBC to QEMU provides a uniform, minimal test harness across all QEMU-supported architectures.

### 1.3 Relationship to Existing QEMU Semihosting

QEMU already has semihosting support for ARM, RISC-V, and some other architectures via trap instructions (e.g., `BKPT`, `SVC`, `EBREAK`). The ZBC semihosting device is:

- **Memory-mapped**: Uses standard load/store, no special instructions
- **Architecture-agnostic**: Same interface on all platforms
- **Self-describing**: RIFF protocol with explicit configuration

ZBC semihosting and existing QEMU semihosting can coexist. Guest software chooses which mechanism to use.

---

## 2. Target Architectures

QEMU supports 14 system emulation targets. ZBC boards will be created for all of them:

| Architecture | QEMU Binary | Address Width | Notes |
|--------------|-------------|---------------|-------|
| ARM (32-bit) | `qemu-system-arm` | 32-bit | Cortex-A/R/M profiles |
| ARM (64-bit) | `qemu-system-aarch64` | 64-bit | AArch64 |
| AVR | `qemu-system-avr` | 16-bit | Harvard architecture |
| ColdFire/m68k | `qemu-system-m68k` | 32-bit | Motorola 68000 family |
| LoongArch | `qemu-system-loongarch64` | 64-bit | Chinese ISA |
| MIPS (32-bit) | `qemu-system-mips[el]` | 32-bit | Big/little endian variants |
| MIPS (64-bit) | `qemu-system-mips64[el]` | 64-bit | Big/little endian variants |
| OpenRISC | `qemu-system-or1k` | 32-bit | Open ISA |
| PowerPC (32-bit) | `qemu-system-ppc` | 32-bit | |
| PowerPC (64-bit) | `qemu-system-ppc64` | 64-bit | |
| RISC-V (32-bit) | `qemu-system-riscv32` | 32-bit | |
| RISC-V (64-bit) | `qemu-system-riscv64` | 64-bit | |
| RX | `qemu-system-rx` | 32-bit | Renesas |
| s390x | `qemu-system-s390x` | 64-bit | IBM mainframe |
| SPARC (32-bit) | `qemu-system-sparc` | 32-bit | |
| SPARC (64-bit) | `qemu-system-sparc64` | 64-bit | |
| x86 (32-bit) | `qemu-system-i386` | 32-bit | |
| x86 (64-bit) | `qemu-system-x86_64` | 64-bit | |
| Xtensa | `qemu-system-xtensa` | 32-bit | Configurable cores |

**Note**: Some architectures have both 32-bit and 64-bit variants. Each variant gets its own ZBC machine type (e.g., `zbc-riscv32`, `zbc-riscv64`).

---

## 3. Component Overview

### 3.1 Core Components

```
┌─────────────────────────────────────────────────────────────┐
│                     ZBC Machine Type                         │
│  (zbc-arm, zbc-riscv64, zbc-i386, etc.)                     │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌──────────┐  ┌─────────────────┐  ┌──────────────────┐   │
│  │   CPU    │  │  ZBC Semihost   │  │  MC6847 Display  │   │
│  │          │  │  Device (32B)   │  │  (512B VRAM)     │   │
│  └────┬─────┘  └────────┬────────┘  └────────┬─────────┘   │
│       │                 │                     │              │
│       │                 │ IRQ (optional)      │              │
│       │◄────────────────┘                     │              │
│       │                                       │              │
│  ┌────┴───────────────────────────────────────┴─────────┐   │
│  │                      RAM                              │   │
│  │         (sized to fill address space)                 │   │
│  └───────────────────────────────────────────────────────┘   │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 3.2 Component Responsibilities

| Component | Responsibility |
|-----------|----------------|
| ZBC Semihost Device | Process RIFF-based semihosting requests (file I/O, console, time, exit) |
| MC6847 Display | Render 32×16 character framebuffer to QEMU display |
| ZBC Machine Type | Instantiate CPU, RAM, devices; calculate memory layout; wire interrupts |

---

## 4. ZBC Semihosting Device

### 4.1 Device Type

- **QOM Type**: `TYPE_ZBC_SEMIHOST` ("zbc-semihost")
- **Parent**: `TYPE_SYS_BUS_DEVICE`
- **Category**: `hw/misc/`

### 4.2 Register Map (32 bytes)

| Offset | Size | Name | Access | Description |
|--------|------|------|--------|-------------|
| 0x00 | 8 | SIGNATURE | R | ASCII "SEMIHOST" |
| 0x08 | 16 | RIFF_PTR | RW | Guest pointer to RIFF buffer |
| 0x18 | 1 | DOORBELL | W | Write triggers request processing |
| 0x19 | 1 | IRQ_STATUS | R | Bit 0: RESPONSE_READY, Bit 1: ERROR |
| 0x1A | 1 | IRQ_ENABLE | RW | Interrupt enable mask |
| 0x1B | 1 | IRQ_ACK | W | Write 1s to clear IRQ_STATUS bits |
| 0x1C | 1 | STATUS | R | Bit 0: RESPONSE_READY, Bit 7: DEVICE_PRESENT |
| 0x1D-0x1F | 3 | RESERVED | - | Reserved |

### 4.3 Device Properties

| Property | Type | Description |
|----------|------|-------------|
| `int-size` | uint8 | Guest C `int` size in bytes (default: derived from CPU) |
| `ptr-size` | uint8 | Guest pointer size in bytes (default: derived from CPU) |
| `endianness` | uint8 | 0=LE, 1=BE, 2=PDP (default: derived from CPU) |

These properties are normally auto-configured by the ZBC machine type based on the CPU, but can be overridden for testing.

### 4.4 RIFF Protocol Processing

When DOORBELL is written:

1. Read RIFF buffer from guest memory at RIFF_PTR address
2. Validate RIFF header (`RIFF....SEMI`)
3. Parse CNFG chunk (if present) to learn guest architecture
4. Parse CALL chunk to extract opcode and parameters
5. Dispatch to appropriate syscall handler
6. Write RETN or ERRO chunk back to guest memory
7. Set IRQ_STATUS.RESPONSE_READY
8. If IRQ_ENABLE.RESPONSE_READY_EN, assert IRQ line

### 4.5 Supported Syscalls

All 23 ARM semihosting syscalls:

| Opcode | Name | Description |
|--------|------|-------------|
| 0x01 | SYS_OPEN | Open file |
| 0x02 | SYS_CLOSE | Close file |
| 0x03 | SYS_WRITEC | Write character to stdout |
| 0x04 | SYS_WRITE0 | Write string to stdout |
| 0x05 | SYS_WRITE | Write buffer to file |
| 0x06 | SYS_READ | Read from file |
| 0x07 | SYS_READC | Read character from stdin |
| 0x08 | SYS_ISERROR | Check if value is error |
| 0x09 | SYS_ISTTY | Check if fd is TTY |
| 0x0A | SYS_SEEK | Seek in file |
| 0x0C | SYS_FLEN | Get file length |
| 0x0D | SYS_TMPNAM | Generate temp filename |
| 0x0E | SYS_REMOVE | Delete file |
| 0x0F | SYS_RENAME | Rename file |
| 0x10 | SYS_CLOCK | Centiseconds since start |
| 0x11 | SYS_TIME | Unix epoch seconds |
| 0x12 | SYS_SYSTEM | Execute command (disabled) |
| 0x13 | SYS_ERRNO | Get last errno |
| 0x15 | SYS_GET_CMDLINE | Get command line |
| 0x16 | SYS_HEAPINFO | Get heap/stack info |
| 0x18 | SYS_EXIT | Exit emulation |
| 0x20 | SYS_EXIT_EXTENDED | Exit with extended info |
| 0x30 | SYS_ELAPSED | 64-bit tick count |
| 0x31 | SYS_TICKFREQ | Tick frequency |

### 4.6 Backend Implementation

The device uses a backend abstraction for actual I/O operations:

```c
typedef struct ZbcSemihostBackend {
    int (*open)(const char *path, int mode);
    int (*close)(int fd);
    ssize_t (*read)(int fd, void *buf, size_t count);
    ssize_t (*write)(int fd, const void *buf, size_t count);
    // ... etc
} ZbcSemihostBackend;
```

Default backend: POSIX operations with optional sandboxing (configurable).

### 4.7 Security Considerations

- `SYS_SYSTEM` is disabled by default (returns EPERM)
- File operations can be sandboxed to a directory (optional property)
- All paths are validated to prevent directory traversal

---

## 5. MC6847 Text Display Device

### 5.1 Device Type

- **QOM Type**: `TYPE_MC6847` ("mc6847")
- **Parent**: `TYPE_SYS_BUS_DEVICE`
- **Category**: `hw/display/`

### 5.2 Simplified Implementation

This is a "good enough" implementation for text display, not a cycle-accurate MC6847:

- 32 columns × 16 rows = 512 bytes VRAM
- Each byte is a character code (ASCII subset)
- No semigraphics modes
- No color attributes (green-on-black or amber-on-black)
- Fixed ~60Hz refresh to QEMU display

### 5.3 Memory Interface

- 512-byte memory region mapped into guest address space
- Guest writes character codes directly to VRAM
- Device reads VRAM and renders to QEMU's display console

### 5.4 Device Properties

| Property | Type | Description |
|----------|------|-------------|
| `cols` | uint8 | Columns (default: 32) |
| `rows` | uint8 | Rows (default: 16) |
| `foreground` | uint32 | Foreground color (default: green) |
| `background` | uint32 | Background color (default: black) |

### 5.5 Rendering

- Create a QEMU `DisplaySurface` of appropriate pixel size
- On each refresh (or VRAM write, if optimizing), render characters to surface
- Use a built-in bitmap font (8×8 or 8×12 pixels per character)
- Register with QEMU's display subsystem via `graphic_console_init()`

### 5.6 Character Set

- Support printable ASCII (0x20-0x7E)
- Uppercase conversion optional (authentic MC6847 behavior)
- Undefined characters render as space or placeholder glyph

---

## 6. ZBC Machine Types

### 6.1 Machine Naming Convention

Each architecture gets a ZBC machine type:

| Architecture | Machine Name | QEMU Invocation |
|--------------|--------------|-----------------|
| ARM 32-bit | `zbc-arm` | `qemu-system-arm -M zbc-arm` |
| ARM 64-bit | `zbc-aarch64` | `qemu-system-aarch64 -M zbc-aarch64` |
| RISC-V 32-bit | `zbc-riscv32` | `qemu-system-riscv32 -M zbc-riscv32` |
| RISC-V 64-bit | `zbc-riscv64` | `qemu-system-riscv64 -M zbc-riscv64` |
| x86 32-bit | `zbc-i386` | `qemu-system-i386 -M zbc-i386` |
| x86 64-bit | `zbc-x86_64` | `qemu-system-x86_64 -M zbc-x86_64` |
| m68k | `zbc-m68k` | `qemu-system-m68k -M zbc-m68k` |
| MIPS 32-bit | `zbc-mips` | `qemu-system-mips -M zbc-mips` |
| MIPS 64-bit | `zbc-mips64` | `qemu-system-mips64 -M zbc-mips64` |
| PowerPC 32-bit | `zbc-ppc` | `qemu-system-ppc -M zbc-ppc` |
| PowerPC 64-bit | `zbc-ppc64` | `qemu-system-ppc64 -M zbc-ppc64` |
| SPARC 32-bit | `zbc-sparc` | `qemu-system-sparc -M zbc-sparc` |
| SPARC 64-bit | `zbc-sparc64` | `qemu-system-sparc64 -M zbc-sparc64` |
| s390x | `zbc-s390x` | `qemu-system-s390x -M zbc-s390x` |
| OpenRISC | `zbc-or1k` | `qemu-system-or1k -M zbc-or1k` |
| Xtensa | `zbc-xtensa` | `qemu-system-xtensa -M zbc-xtensa` |
| LoongArch | `zbc-loongarch64` | `qemu-system-loongarch64 -M zbc-loongarch64` |
| RX | `zbc-rx` | `qemu-system-rx -M zbc-rx` |
| AVR | `zbc-avr` | `qemu-system-avr -M zbc-avr` |

### 6.2 Machine Initialization Flow

Each ZBC machine's `init` function:

1. **Create CPU**: Instantiate default CPU for the architecture
2. **Calculate memory layout**: Use address width to determine:
   - RAM size
   - Semihost device address
   - VRAM address
   - Load address
3. **Create RAM**: Map RAM across available address space
4. **Create semihost device**: Map at calculated address
5. **Create display device**: Map VRAM at calculated address
6. **Wire interrupts**: Connect semihost IRQ to CPU's interrupt controller
7. **Set entry point**: Configure CPU to start at load address

### 6.3 Machine Properties

| Property | Type | Description |
|----------|------|-------------|
| `cpu` | string | CPU model (default: architecture-specific) |
| `ram-size` | size | RAM size (default: calculated from address width) |

### 6.4 Shared Infrastructure

Common code in `hw/zbc/zbc-common.c`:

```c
/* Calculate semihost device address from address width */
uint64_t zbc_semihost_addr(unsigned int addr_bits);

/* Calculate VRAM address from address width */
uint64_t zbc_vram_addr(unsigned int addr_bits);

/* Calculate program load address from address width */
uint64_t zbc_load_addr(unsigned int addr_bits);

/* Common device wiring logic */
void zbc_create_semihost(/* ... */);
void zbc_create_display(/* ... */);
```

---

## 7. Memory Layout Strategy

### 7.1 Address Calculation Formulas

From the ZBC specification, all addresses are derived from CPU address width `n`:

| Region | Formula | 16-bit | 32-bit | 64-bit |
|--------|---------|--------|--------|--------|
| Reserved start | `2^n - 2^(n/2)` | 0xFF00 | 0xFFFF0000 | 0xFFFFFFFF00000000 |
| VRAM (512B) | `reserved - 512` | 0xFE00 | 0xFFFFFE00 | 0xFFFFFFFFFFFFFE00 |
| Semihost (32B) | `reserved - 1536` | 0xFA00 | 0xFFFFFA00 | 0xFFFFFFFFFFFFFA00 |
| Load address | `2^(1 + n/2)` | 0x0200 | 0x00020000 | 0x0000000200000000 |
| RAM | `0` to `semihost - 1` | ~64KB | ~4GB | ~16EB |

### 7.2 Implementation

```c
static inline uint64_t zbc_reserved_start(unsigned int addr_bits)
{
    return (1ULL << addr_bits) - (1ULL << (addr_bits / 2));
}

static inline uint64_t zbc_vram_addr(unsigned int addr_bits)
{
    return zbc_reserved_start(addr_bits) - 512;
}

static inline uint64_t zbc_semihost_addr(unsigned int addr_bits)
{
    return zbc_reserved_start(addr_bits) - 1536;
}

static inline uint64_t zbc_load_addr(unsigned int addr_bits)
{
    return 1ULL << (1 + addr_bits / 2);
}
```

### 7.3 Memory Map Example (32-bit)

```
0x00000000 ┌─────────────────────────────────────┐
           │                                     │
           │              RAM                    │
           │         (~4GB - 64KB)               │
           │                                     │
0xFFFF0000 ├─────────────────────────────────────┤
           │         Reserved Region             │
0xFFFFFA00 ├─────────────────────────────────────┤
           │    ZBC Semihost Device (32 bytes)   │
0xFFFFFA20 ├─────────────────────────────────────┤
           │           (unused gap)              │
0xFFFFFE00 ├─────────────────────────────────────┤
           │      MC6847 VRAM (512 bytes)        │
0xFFFFFFFF └─────────────────────────────────────┘

Load address: 0x00020000
```

---

## 8. Binary Loading

### 8.1 Mechanism

Use QEMU's standard `-kernel` option:

```bash
qemu-system-riscv64 -M zbc-riscv64 -kernel program.bin
```

### 8.2 Implementation

In each ZBC machine's init function:

```c
/* Load kernel image at calculated load address */
uint64_t load_addr = zbc_load_addr(addr_bits);
uint64_t entry = load_addr;

if (machine->kernel_filename) {
    load_image_targphys(machine->kernel_filename, load_addr, ram_size);
    /* Set CPU program counter to load address */
    cpu_set_pc(cpu, entry);
}
```

### 8.3 Entry Point

The loaded binary's first byte is at the load address. The CPU's program counter is set to this address. The binary is responsible for its own initialization.

No bootloader, no firmware, no headers—just raw code.

---

## 9. Interrupt Routing

### 9.1 Strategy

Each ZBC machine wires the semihost device's IRQ output to the platform's standard external interrupt mechanism:

| Architecture | Interrupt Controller | IRQ Routing |
|--------------|---------------------|-------------|
| ARM | GIC (Generic Interrupt Controller) | SPI (Shared Peripheral Interrupt) |
| RISC-V | PLIC (Platform-Level Interrupt Controller) | External interrupt source |
| x86 | IOAPIC / PIC | IRQ line |
| m68k | Direct CPU IRQ | IPL lines |
| PowerPC | OpenPIC or direct | External interrupt |
| MIPS | CP0 or external controller | Interrupt pin |
| s390x | Direct CPU | I/O interrupt |
| SPARC | Direct CPU | Interrupt level |
| Others | Architecture-specific | Standard mechanism |

### 9.2 Implementation Pattern

```c
/* Create semihost device */
dev = qdev_new(TYPE_ZBC_SEMIHOST);
sysbus_realize_and_unref(SYS_BUS_DEVICE(dev), &error_fatal);
sysbus_mmio_map(SYS_BUS_DEVICE(dev), 0, semihost_addr);

/* Wire IRQ to interrupt controller */
sysbus_connect_irq(SYS_BUS_DEVICE(dev), 0, irq_line);
```

### 9.3 Guest Software Considerations

Guest software that wants asynchronous semihosting:

1. Writes `IRQ_ENABLE = 0x01` to enable completion interrupt
2. Sets up interrupt handler for the platform
3. In handler: reads RETN, then writes `IRQ_ACK = 0x01`

Polling mode (default) requires no interrupt handling.

---

## 10. Console I/O

### 10.1 Output Routing

| Syscall | Destination |
|---------|-------------|
| SYS_WRITEC | Host stdout |
| SYS_WRITE0 | Host stdout |
| SYS_WRITE (fd=1) | Host stdout |
| SYS_WRITE (fd=2) | Host stderr |
| SYS_WRITE (fd≥3) | File (if opened) |

### 10.2 Input Routing

| Syscall | Source |
|---------|--------|
| SYS_READC | Host stdin |
| SYS_READ (fd=0) | Host stdin |
| SYS_READ (fd≥3) | File (if opened) |

### 10.3 Implementation

```c
static void semihost_writec(ZbcSemihostState *s, char c)
{
    fputc(c, stdout);
    fflush(stdout);
}

static int semihost_readc(ZbcSemihostState *s)
{
    return fgetc(stdin);
}
```

### 10.4 Not Using QEMU Serial

The semihost console is *not* a QEMU chardev/serial device. It directly uses host stdio. This is simpler and matches expected behavior for embedded development tools.

---

## 11. Build System Integration

### 11.1 Meson Configuration

New files added to QEMU's Meson build:

**`hw/misc/meson.build`** (addition):
```meson
system_ss.add(when: 'CONFIG_ZBC_SEMIHOST', if_true: files('zbc-semihost.c'))
```

**`hw/display/meson.build`** (addition):
```meson
system_ss.add(when: 'CONFIG_MC6847', if_true: files('mc6847.c'))
```

**`hw/zbc/meson.build`** (new):
```meson
system_ss.add(when: 'CONFIG_ZBC', if_true: files('zbc-common.c'))

# Per-architecture machine files
system_ss.add(when: 'CONFIG_ZBC_ARM', if_true: files('zbc-arm.c'))
system_ss.add(when: 'CONFIG_ZBC_RISCV', if_true: files('zbc-riscv.c'))
# ... etc for each architecture
```

### 11.2 Kconfig

**`hw/misc/Kconfig`** (addition):
```
config ZBC_SEMIHOST
    bool
```

**`hw/display/Kconfig`** (addition):
```
config MC6847
    bool
```

**`hw/zbc/Kconfig`** (new):
```
config ZBC
    bool
    select ZBC_SEMIHOST
    select MC6847

config ZBC_ARM
    bool
    select ZBC
    select ARM

config ZBC_RISCV
    bool
    select ZBC
    select RISCV

# ... etc
```

**`hw/Kconfig`** (addition):
```
source zbc/Kconfig
```

### 11.3 Target Default Configs

For each architecture, add ZBC to the default configuration:

**`configs/targets/arm-softmmu.mak`** (addition):
```
CONFIG_ZBC_ARM=y
```

**`configs/targets/riscv64-softmmu.mak`** (addition):
```
CONFIG_ZBC_RISCV=y
```

*(Repeat for all 14 architectures)*

---

## 12. Testing Strategy

### 12.1 Unit Tests

Test the semihost device in isolation:

- Register read/write behavior
- RIFF parsing (valid and invalid inputs)
- Syscall dispatch
- Endianness handling
- Error conditions

Location: `tests/unit/test-zbc-semihost.c`

### 12.2 Functional Tests

Test complete ZBC machines:

- Boot and reach load address
- Semihost write to stdout
- Semihost file operations
- Display output
- Interrupt-driven operation

Location: `tests/functional/test_zbc_*.py`

### 12.3 Cross-Architecture Test Suite

A single test binary (written in C, compiled for each architecture) that exercises:

- All 23 syscalls
- Edge cases
- Error handling

This validates consistent behavior across all 14 ZBC machine types.

### 12.4 CI Integration

Add ZBC tests to QEMU's CI pipeline:

- Build all ZBC-enabled targets
- Run functional tests for each architecture
- Ensure no regressions

---

## 13. Documentation

### 13.1 Main Documentation

**`docs/system/zbc.rst`**:

- Overview and motivation
- Supported architectures
- Usage examples
- Memory layout
- Semihosting protocol reference
- Device properties
- Comparison with existing semihosting

### 13.2 Per-Architecture Notes

**`docs/system/target-<arch>.rst`** (additions):

Each existing architecture documentation page gets a section on the ZBC machine variant.

### 13.3 Man Pages

Update `qemu-system-*.1` man pages to list ZBC machines.

---

## 14. Upstream Submission Strategy

### 14.1 Patch Series Organization

Submit as a series of logical patches:

1. **Patch 1-2**: ZBC semihost device (`hw/misc/zbc-semihost.c`, headers)
2. **Patch 3-4**: MC6847 display device (`hw/display/mc6847.c`, headers)
3. **Patch 5**: ZBC common infrastructure (`hw/zbc/zbc-common.c`)
4. **Patch 6-N**: Individual machine types (one patch per architecture)
5. **Patch N+1**: Documentation
6. **Patch N+2**: Tests

### 14.2 Mailing List

Submit to `qemu-devel@nongnu.org` with appropriate maintainers CC'd:
- Architecture maintainers for each target
- Device maintainers for misc/display

### 14.3 Coding Standards

- Follow QEMU coding style (Linux kernel style)
- Use QEMU's standard macros (OBJECT_DECLARE_*, OBJECT_DEFINE_*, etc.)
- Include Signed-off-by lines
- Write detailed commit messages

### 14.4 Review Process

Expect multiple review rounds. Be prepared to:
- Justify design decisions
- Refactor based on feedback
- Add tests or documentation as requested

---

## 15. File Structure

### 15.1 New Files

```
hw/
├── misc/
│   ├── zbc-semihost.c           # Semihost device implementation
│   └── Kconfig                   # (modified)
├── display/
│   ├── mc6847.c                  # Display device implementation
│   └── Kconfig                   # (modified)
├── zbc/
│   ├── zbc-common.c              # Shared ZBC infrastructure
│   ├── zbc-arm.c                 # ARM ZBC machine
│   ├── zbc-aarch64.c             # AArch64 ZBC machine
│   ├── zbc-riscv.c               # RISC-V ZBC machines (32/64)
│   ├── zbc-i386.c                # x86 ZBC machines (32/64)
│   ├── zbc-m68k.c                # m68k/ColdFire ZBC machine
│   ├── zbc-mips.c                # MIPS ZBC machines
│   ├── zbc-ppc.c                 # PowerPC ZBC machines
│   ├── zbc-sparc.c               # SPARC ZBC machines
│   ├── zbc-s390x.c               # s390x ZBC machine
│   ├── zbc-or1k.c                # OpenRISC ZBC machine
│   ├── zbc-xtensa.c              # Xtensa ZBC machine
│   ├── zbc-loongarch.c           # LoongArch ZBC machine
│   ├── zbc-rx.c                  # RX ZBC machine
│   ├── zbc-avr.c                 # AVR ZBC machine
│   ├── Kconfig                   # ZBC Kconfig
│   └── meson.build               # ZBC meson build
├── Kconfig                       # (modified to include zbc/)
└── meson.build                   # (modified to include zbc/)

include/
├── hw/
│   ├── misc/
│   │   └── zbc-semihost.h        # Semihost device header
│   ├── display/
│   │   └── mc6847.h              # Display device header
│   └── zbc/
│       └── zbc.h                 # ZBC common header

docs/
└── system/
    └── zbc.rst                   # ZBC documentation

tests/
├── unit/
│   └── test-zbc-semihost.c       # Unit tests
└── functional/
    └── test_zbc_*.py             # Functional tests

configs/targets/
├── arm-softmmu.mak               # (modified)
├── aarch64-softmmu.mak           # (modified)
├── riscv32-softmmu.mak           # (modified)
├── riscv64-softmmu.mak           # (modified)
# ... etc for all architectures
```

### 15.2 Lines of Code Estimate

| Component | Estimated LOC |
|-----------|---------------|
| zbc-semihost.c | 800-1200 |
| mc6847.c | 300-500 |
| zbc-common.c | 100-200 |
| Each machine file | 150-300 |
| Headers | 200-300 |
| Documentation | 500-800 |
| Tests | 500-1000 |
| **Total** | ~5000-8000 |

---

## 16. Implementation Phases

### Phase 1: Core Devices

1. Implement `zbc-semihost` device
2. Implement `mc6847` display device
3. Unit tests for both devices

**Deliverable**: Standalone devices that can be manually instantiated

### Phase 2: First Machine Type

4. Implement `zbc-common.c` shared infrastructure
5. Implement one ZBC machine (suggest: RISC-V 64-bit as it's well-supported and clean)
6. Functional tests for that machine

**Deliverable**: Working `qemu-system-riscv64 -M zbc-riscv64 -kernel test.bin`

### Phase 3: Remaining Machines

7. Implement remaining 13+ machine types
8. Architecture-specific adjustments as needed
9. Functional tests for each

**Deliverable**: All ZBC machines working

### Phase 4: Polish

10. Documentation
11. Code review and cleanup
12. CI integration
13. Upstream submission preparation

**Deliverable**: Ready for upstream review

---

## 17. Architecture-Specific Notes

### 17.1 ARM (32-bit and 64-bit)

- Use Cortex-A series CPU (e.g., `cortex-a15` for 32-bit, `cortex-a53` for 64-bit)
- Wire semihost IRQ to GIC SPI
- Existing QEMU ARM semihosting can coexist

### 17.2 RISC-V (32-bit and 64-bit)

- Clean architecture, good starting point
- Wire semihost IRQ to PLIC
- Existing QEMU RISC-V semihosting can coexist

### 17.3 x86 (32-bit and 64-bit)

- Start in real mode or protected mode?
- Recommendation: Start in 32-bit protected mode with flat memory model
- For 64-bit: Start in long mode
- Wire semihost IRQ to IOAPIC or legacy PIC

### 17.4 m68k (ColdFire)

- Classic architecture, straightforward
- Direct CPU interrupt lines

### 17.5 MIPS

- Big-endian and little-endian variants
- CP0 interrupt handling

### 17.6 PowerPC

- 32-bit and 64-bit variants
- OpenPIC or direct interrupt

### 17.7 SPARC

- 32-bit and 64-bit variants
- Register windows and trap handling

### 17.8 s390x

- Unusual I/O model (channel-based)
- May need special consideration for memory-mapped devices
- Consult s390x maintainers

### 17.9 OpenRISC

- Straightforward 32-bit RISC
- Direct interrupt lines

### 17.10 Xtensa

- Configurable cores
- Use a standard configuration

### 17.11 LoongArch

- Newer architecture
- Similar to MIPS in some ways

### 17.12 RX

- Renesas microcontroller
- Straightforward 32-bit

### 17.13 AVR

- 8-bit data, 16-bit address
- Harvard architecture (separate code/data)
- May need special handling for semihost buffer access
- **Potential challenge**: Harvard architecture means code and data spaces are separate

---

## 18. Open Questions and Risks

### 18.1 Open Questions

1. **s390x I/O model**: Does memory-mapped I/O work naturally on s390x, or does its channel-based I/O require a different approach?

2. **AVR Harvard architecture**: How to handle the split code/data address spaces? The semihost buffer is in data space, but does QEMU's AVR emulation handle this correctly?

3. **Xtensa core configuration**: Which Xtensa core configuration should be the default? QEMU supports multiple.

4. **x86 startup mode**: Should the x86 ZBC machine start in real mode (authentic) or protected/long mode (convenient)?

5. **Default RAM sizes**: Should RAM size be the maximum addressable, or something more reasonable (e.g., 128MB for 32-bit)?

### 18.2 Risks

1. **Upstream acceptance**: QEMU maintainers may push back on adding 14+ new machine types. Mitigation: Emphasize utility for testing, CI, embedded development.

2. **Architecture quirks**: Some architectures may have unexpected complications. Mitigation: Start with clean architectures (RISC-V, ARM), tackle harder ones later.

3. **Maintenance burden**: 14 machine types to maintain. Mitigation: Maximize shared code in `zbc-common.c`.

4. **Display complexity**: MC6847 implementation may be more complex than expected. Mitigation: Keep it minimal (text only, no semigraphics).

---

## Appendix A: Reference Implementation

The semihosting protocol specification and reference C implementation are available at:

- Specification: `~/git/semihost/doc/`
- Host implementation: `~/git/semihost/src/zbc_host.c`
- Client implementation: `~/git/semihost/src/zbc_client.c`
- Backend interface: `~/git/semihost/include/zbc_backend.h`

This code can be adapted or used as a reference for the QEMU implementation.

---

## Appendix B: Example Usage

### Basic Usage

```bash
# Compile a simple program for RISC-V 64-bit
riscv64-unknown-elf-gcc -nostdlib -T linker.ld -o program.elf program.c
riscv64-unknown-elf-objcopy -O binary program.elf program.bin

# Run on ZBC
qemu-system-riscv64 -M zbc-riscv64 -kernel program.bin -nographic
```

### With Display

```bash
# Run with graphical display
qemu-system-riscv64 -M zbc-riscv64 -kernel program.bin
```

### Debugging

```bash
# Run with GDB stub
qemu-system-riscv64 -M zbc-riscv64 -kernel program.bin -s -S

# In another terminal
riscv64-unknown-elf-gdb program.elf -ex "target remote :1234"
```

---

*End of document*
