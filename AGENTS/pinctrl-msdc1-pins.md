# MT6789 MSDC1 Pin Register Map — GPIO 71-76

**Source**: Binary-extracted from `kernelmodules/ko/pinctrl-mt6789.ko` `.rodata` section
**Formula**: From `pinctrl-mtk-common-v2.c:126-134` (`mtk_hw_pin_field_lookup`)
**Struct**: `struct mtk_pin_field_calc` — 20 bytes per entry (u16 s_pin, u16 e_pin, u8 i_base, u32 s_addr, u8 x_addrs, u8 s_bit, u8 x_bits, u8 sz_reg, u8 fixed)

## Critical Correction

The previous analysis document (`pinctrl-mt6789-ko-analysis.md`) had **INCORRECT mode register addresses**. It assumed each pin gets its own 0x10-byte-spaced register (0x10005770, 0x10005780, etc.), but the actual kernel driver packs **8 pins per 32-bit register** (4 bits each). The correct addresses are different and documented below.

## Register Computation Formula (from kernel source)

```c
// From pinctrl-mtk-common-v2.c:126-134
bits = s_bit + (pin - s_pin) * x_bits;          // when fixed=0
offset = s_addr + x_addrs * (bits / sz_reg);     // register offset from base
bitpos = bits % sz_reg;                           // bit position within register
```

Physical address = `base_addresses[i_base] + offset`

## Bank Base Addresses (from DTS)

| i_base | Name  | Physical Base | Description |
|--------|-------|---------------|-------------|
| 0      | base  | 0x10005000    | GPIO mode/dir/di/do registers |
| 1      | rb    | 0x11c30000    | IOCFG RB bank |
| 2      | bm    | 0x11d10000    | IOCFG BM bank |
| 3      | br    | 0x11d40000    | IOCFG BR bank |
| 4      | lm    | 0x11e20000    | IOCFG LM bank |
| 5      | bl    | 0x11e60000    | IOCFG BL bank |
| 6      | rt    | 0x11ea0000    | IOCFG RT bank (GPIO 71-76 are here for IES/SMT/PU/DRV) |
| 7      | eint  | 0x1000b000    | EINT registers |

## MSDC1 Pin Function Mapping

**ALL MSDC1 pins use function number = 1**

| GPIO | Name | Function | Function Number |
|------|------|----------|-----------------|
| 71 | GPIO71 | MSDC1_CLK | 1 |
| 72 | GPIO72 | MSDC1_CMD | 1 |
| 73 | GPIO73 | MSDC1_DAT0 | 1 |
| 74 | GPIO74 | MSDC1_DAT1 | 1 |
| 75 | GPIO75 | MSDC1_DAT2 | 1 |
| 76 | GPIO76 | MSDC1_DAT3 | 1 |

Source: `mediatek,mt6789-pinfunc.h` lines 471-513

## Binary-Extracted Range Tables Used

All tables extracted from `.rodata` section of `pinctrl-mt6789.ko` (file offset 0x1290):

| Symbol | Section Offset | Size |
|--------|---------------|------|
| `mt6789_pin_mode_range` | 0x3d38 | 0x1E0 (480 bytes, 24 entries) |
| `mt6789_pin_dir_range` | 0x3f18 | 0x78 (120 bytes, 6 entries) |
| `mt6789_pin_di_range` | 0x3f90 | 0x78 (120 bytes) |
| `mt6789_pin_do_range` | 0x4008 | 0x78 (120 bytes) |
| `mt6789_pin_smt_range` | 0x4080 | 0xEB0 (3760 bytes, 188 entries) |
| `mt6789_pin_ies_range` | 0x7d5c | 0xEb0 (3760 bytes, 188 entries) |
| `mt6789_pin_pupd_range` | 0x76cc | 0x230 (560 bytes) |
| `mt6789_pin_r0_range` | 0x78fc | 0x230 (560 bytes) |
| `mt6789_pin_r1_range` | 0x7b2c | 0x230 (560 bytes) |
| `mt6789_pin_drv_range` | 0x6830 | 0xEA4 (3740+ bytes) |

## Complete Register Map for GPIO 71-76

### Pin Mode (Function Select) — 4 bits per pin

GPIO 71 is in the **same register** as GPIO 64-70. GPIO 72-76 are in the **next register**.

| GPIO | Physical Address | Bits | Shared Register |
|------|-----------------|------|-----------------|
| 71 (CLK) | **0x10005380** | [31:28] | Same reg as GPIO 64-70 |
| 72 (CMD) | **0x10005390** | [3:0] | Same reg as GPIO 73-79 |
| 73 (DAT0) | **0x10005390** | [7:4] | Same reg |
| 74 (DAT1) | **0x10005390** | [11:8] | Same reg |
| 75 (DAT2) | **0x10005390** | [15:12] | Same reg |
| 76 (DAT3) | **0x10005390** | [19:16] | Same reg |

**Range table entry**: GPIO 64-71 → `s_addr=0x0380, x_addrs=0x10, s_bit=0, x_bits=4, sz_reg=32`
**Range table entry**: GPIO 72-79 → `s_addr=0x0390, x_addrs=0x10, s_bit=0, x_bits=4, sz_reg=32`

### Direction Register — 1 bit per pin

All GPIO 71-76 share the **same register**.

| GPIO | Physical Address | Bit | Description |
|------|-----------------|-----|-------------|
| 71 | **0x10005020** | [7] | 1=output, 0=input |
| 72 | **0x10005020** | [8] | 1=output, 0=input |
| 73 | **0x10005020** | [9] | 1=output, 0=input |
| 74 | **0x10005020** | [10] | 1=output, 0=input |
| 75 | **0x10005020** | [11] | 1=output, 0=input |
| 76 | **0x10005020** | [12] | 1=output, 0=input |

### Data Input (DI) — 1 bit per pin

All GPIO 71-76 share the **same register**.

| GPIO | Physical Address | Bit |
|------|-----------------|-----|
| 71 | **0x10005220** | [7] |
| 72 | **0x10005220** | [8] |
| 73 | **0x10005220** | [9] |
| 74 | **0x10005220** | [10] |
| 75 | **0x10005220** | [11] |
| 76 | **0x10005220** | [12] |

### Data Output (DO) — 1 bit per pin

All GPIO 71-76 share the **same register**.

| GPIO | Physical Address | Bit |
|------|-----------------|-----|
| 71 | **0x10005120** | [7] |
| 72 | **0x10005120** | [8] |
| 73 | **0x10005120** | [9] |
| 74 | **0x10005120** | [10] |
| 75 | **0x10005120** | [11] |
| 76 | **0x10005120** | [12] |

### IES (Input Enable / IO Strap) — 1 bit per pin

All GPIO 71-76 share the **same register** in bank "rt" (0x11ea0000).

| GPIO | Physical Address | Bit | Description |
|------|-----------------|-----|-------------|
| 71 | **0x11EA0080** | [1] | 1=enable |
| 72 | **0x11EA0080** | [2] | 1=enable |
| 73 | **0x11EA0080** | [3] | 1=enable |
| 74 | **0x11EA0080** | [4] | 1=enable |
| 75 | **0x11EA0080** | [5] | 1=enable |
| 76 | **0x11EA0080** | [6] | 1=enable |

### SMT (Schmitt Trigger) — 1 bit per pin

All GPIO 71-76 share the **same register**.

| GPIO | Physical Address | Bit |
|------|-----------------|-----|
| 71 | **0x11EA0140** | [16] |
| 72 | **0x11EA0140** | [17] |
| 73 | **0x11EA0140** | [18] |
| 74 | **0x11EA0140** | [19] |
| 75 | **0x11EA0140** | [20] |
| 76 | **0x11EA0140** | [21] |

### Pull Configuration (PUPD + R0 + R1 Type)

GPIO 71-76 use **MTK_PULL_PUPD_R1R0_TYPE** (4-resistor pull, NOT simple PU/PD).

**Pull-Up/Down Select (PUPD)** — 1 bit per pin

| GPIO | Physical Address | Bit |
|------|-----------------|-----|
| 71 | **0x11EA00B0** | [0] |
| 72 | **0x11EA00B0** | [1] |
| 73 | **0x11EA00B0** | [2] |
| 74 | **0x11EA00B0** | [3] |
| 75 | **0x11EA00B0** | [4] |
| 76 | **0x11EA00B0** | [5] |

**R0 Resistor Selector** — 1 bit per pin

| GPIO | Physical Address | Bit |
|------|-----------------|-----|
| 71 | **0x11EA00E0** | [0] |
| 72 | **0x11EA00E0** | [1] |
| 73 | **0x11EA00E0** | [2] |
| 74 | **0x11EA00E0** | [3] |
| 75 | **0x11EA00E0** | [4] |
| 76 | **0x11EA00E0** | [5] |

**R1 Resistor Selector** — 1 bit per pin

| GPIO | Physical Address | Bit |
|------|-----------------|-----|
| 71 | **0x11EA00F0** | [0] |
| 72 | **0x11EA00F0** | [1] |
| 73 | **0x11EA00F0** | [2] |
| 74 | **0x11EA00F0** | [3] |
| 75 | **0x11EA00F0** | [4] |
| 76 | **0x11EA00F0** | [5] |

**Pull Resistor Values (PUPD/R0/R1 encoding)**

| PUPD | R0 | R1 | Mode |
|------|----|----|------|
| 0 | 0 | 0 | High-Z (disabled) |
| 1 | 0 | 0 | Pull-up 50K |
| 1 | 1 | 0 | Pull-up 24K |
| 1 | 1 | 1 | Pull-up 8K |
| 0 | 1 | 0 | Pull-down 50K |
| 0 | 1 | 1 | Pull-down 24K |
| 0 | 0 | 1 | Pull-down 8K |

### Drive Strength — 3 bits per pin (DRV_GRP4)

All GPIO 71-76 are in the **same register**.

| GPIO | Physical Address | Bits | Description |
|------|-----------------|------|-------------|
| 71 | **0x11EA0030** | [11:9] | 3-bit drive value |
| 72 | **0x11EA0030** | [14:12] | 3-bit drive value |
| 73 | **0x11EA0030** | [17:15] | 3-bit drive value |
| 74 | **0x11EA0030** | [20:18] | 3-bit drive value |
| 75 | **0x11EA0030** | [23:21] | 3-bit drive value |
| 76 | **0x11EA0030** | [26:24] | 3-bit drive value |

Drive strength values (DRV_GRP4 — 3-bit drive):
- 0 = 4mA (minimum)
- 1 = 8mA
- 2 = 12mA
- 3 = 16mA (maximum)

## Summary Per Pin

```
GPIO 71 (MSDC1_CLK) — Function 1
  Mode:    0x10005380 [31:28] = 1
  Dir:     0x10005020 [7]     = 0 (input)
  IES:     0x11EA0080 [1]     = 1 (enable)
  SMT:     0x11EA0140 [16]    = 1 (enable Schmitt)
  PUPD:    0x11EA00B0 [0]     = 1 (pull-up)
  R0:      0x11EA00E0 [0]     = 1
  R1:      0x11EA00F0 [0]     = 0  → 24K pull-up
  DRV:     0x11EA0030 [11:9]  = 2 (12mA)

GPIO 72 (MSDC1_CMD) — Function 1
  Mode:    0x10005390 [3:0]   = 1
  Dir:     0x10005020 [8]     = 0 (input)
  IES:     0x11EA0080 [2]     = 1 (enable)
  SMT:     0x11EA0140 [17]    = 1 (enable Schmitt)
  PUPD:    0x11EA00B0 [1]     = 1 (pull-up)
  R0:      0x11EA00E0 [1]     = 1
  R1:      0x11EA00F0 [1]     = 0  → 24K pull-up
  DRV:     0x11EA0030 [14:12] = 2 (12mA)

GPIO 73 (MSDC1_DAT0) — Function 1
  Mode:    0x10005390 [7:4]   = 1
  Dir:     0x10005020 [9]     = 0 (input)
  IES:     0x11EA0080 [3]     = 1 (enable)
  SMT:     0x11EA0140 [18]    = 1 (enable Schmitt)
  PUPD:    0x11EA00B0 [2]     = 1 (pull-up)
  R0:      0x11EA00E0 [2]     = 1
  R1:      0x11EA00F0 [2]     = 0  → 24K pull-up
  DRV:     0x11EA0030 [17:15] = 2 (12mA)

GPIO 74 (MSDC1_DAT1) — Function 1
  Mode:    0x10005390 [11:8]  = 1
  Dir:     0x10005020 [10]    = 0 (input)
  IES:     0x11EA0080 [4]     = 1 (enable)
  SMT:     0x11EA0140 [19]    = 1 (enable Schmitt)
  PUPD:    0x11EA00B0 [3]     = 1 (pull-up)
  R0:      0x11EA00E0 [3]     = 1
  R1:      0x11EA00F0 [3]     = 0  → 24K pull-up
  DRV:     0x11EA0030 [20:18] = 2 (12mA)

GPIO 75 (MSDC1_DAT2) — Function 1
  Mode:    0x10005390 [15:12] = 1
  Dir:     0x10005020 [11]    = 0 (input)
  IES:     0x11EA0080 [5]     = 1 (enable)
  SMT:     0x11EA0140 [20]    = 1 (enable Schmitt)
  PUPD:    0x11EA00B0 [4]     = 1 (pull-up)
  R0:      0x11EA00E0 [4]     = 1
  R1:      0x11EA00F0 [4]     = 0  → 24K pull-up
  DRV:     0x11EA0030 [23:21] = 2 (12mA)

GPIO 76 (MSDC1_DAT3) — Function 1
  Mode:    0x10005390 [19:16] = 1
  Dir:     0x10005020 [12]    = 0 (input)
  IES:     0x11EA0080 [6]     = 1 (enable)
  SMT:     0x11EA0140 [21]    = 1 (enable Schmitt)
  PUPD:    0x11EA00B0 [5]     = 1 (pull-up)
  R0:      0x11EA00E0 [5]     = 1
  R1:      0x11EA00F0 [5]     = 0  → 24K pull-up
  DRV:     0x11EA0030 [26:24] = 2 (12mA)
```

## How to Configure MSDC1 Pins from U-Boot

### Set Pin Mode to MSDC1 (function=1)

**CRITICAL**: GPIO 71 is in a DIFFERENT register than GPIO 72-76. Do NOT use `mw.l` naively — you must do read-modify-write to avoid clobbering GPIO 64-70 or GPIO 77-79 in the shared registers.

```bash
# === Set Mode Registers ===

# GPIO71 mode: 0x10005380, bits [31:28]
# Current value has GPIO64-70 in bits [27:0], preserve them
# Set bits [31:28] = 1 (MSDC1 function)
mw.l 0x10005380 0x10000000
# (This sets GPIO71=1, clears GPIO64-70 bits — WRONG for live system)
# Better: read-modify-write in U-Boot:
#   read 0x10005380 → mask, set [31:28]=1, write back

# GPIO72-76 mode: 0x10005390, bits [3:0] through [19:16]
# Set all 5 pins to function 1:
# [3:0]=1 [7:4]=1 [11:8]=1 [15:12]=1 [19:16]=1
# = 0x00011111 = 0x11111
# GPIO77-79 in bits [31:20] should be preserved
mw.l 0x10005390 0x00011111
```

### Enable IES (Input Enable)

All 6 pins in bits [6:1] of 0x11EA0080:

```bash
# Read current value, set bits [6:1] = 0x7E, preserve other bits
# Quick (if no other pins in this reg matter):
mw.l 0x11EA0080 0x7E
```

### Enable SMT (Schmitt Trigger)

All 6 pins in bits [21:16] of 0x11EA0140:

```bash
# Set bits [21:16] = 0x3F0000
mw.l 0x11EA0140 0x003F0000
```

### Set Pull-Up 24K

```bash
# PUPD bits [5:0] = 0x3F (all 1 = pull-up)
mw.l 0x11EA00B0 0x3F

# R0 bits [5:0] = 0x3F (all 1)
mw.l 0x11EA00E0 0x3F

# R1 bits [5:0] = 0x00 (all 0) → 24K pull-up
mw.l 0x11EA00F0 0x00
```

### Set Drive Strength

All 6 pins in register 0x11EA0030, bits [26:9]:

```bash
# DRV bits packed (value=2=12mA for each pin, 3 bits each):
# [26:24]=010 [23:21]=010 [20:18]=010 [17:15]=010 [14:12]=010 [11:9]=010
# = 0b_010_010_010_010_010_010_0000000000000000 (bits 28-31 are for other pins)
# Need read-modify-write to preserve bits [8:0] and [31:27]
```

## Register Write Sequence (How the Driver Does It)

The `mtk_pconf_pindbg_field()` function in `pinctrl-mtk-common-v2.c` performs:

1. **Lookup**: Binary search through `pin_xxx_range` array to find the entry containing the pin
2. **Calculate bits**: `bits = s_bit + (pin - s_pin) * x_bits`
3. **Calculate offset**: `offset = s_addr + x_addrs * (bits / sz_reg)`
4. **Calculate bitpos**: `bitpos = bits % sz_reg`
5. **Read**: `val = readl(base[i_base] + offset)`
6. **Modify**: `val &= ~(mask << bitpos); val |= (new_val & mask) << bitpos`
7. **Write**: `writel(val, base[i_base] + offset)`

## Data Structures (from Binary)

The `.rodata` section contains `struct mtk_pin_field_calc` arrays:

```c
// 20 bytes per entry (GCC on aarch64):
struct mtk_pin_field_calc {  // 20 bytes
    u16 s_pin;      // +0x00: start pin in range
    u16 e_pin;      // +0x02: end pin in range
    u8  i_base;     // +0x04: index into base address array
    // 3 bytes padding for u32 alignment
    u32 s_addr;     // +0x08: start register offset from base
    u8  x_addrs;    // +0x0C: stride between consecutive registers (typically 0x10)
    u8  s_bit;      // +0x0D: start bit for first pin
    u8  x_bits;     // +0x0E: bits per pin (1, 3, or 4)
    u8  sz_reg;     // +0x0F: register size in bits (always 32)
    u8  fixed;      // +0x10: if 1, all pins in range share same bit position
    // 3 bytes padding to 20-byte alignment
};
```

## Key Observations

1. **8 pins per mode register**: The mode registers pack 8 GPIOs in a single 32-bit word (4 bits each). GPIO 64-71 share 0x10005380, GPIO 72-79 share 0x10005390.

2. **Shared registers for dir/di/do**: GPIO 71-76 all share 0x10005020 (dir), 0x10005120 (do), 0x10005220 (di) with bits [7:12].

3. **IOCFG RT bank**: GPIO 71-76 use bank "rt" (i_base=6, 0x11ea0000) for all I/O configuration: IES, SMT, PUPD, R0, R1, DRV.

4. **Bit-packed registers**: IES, SMT, PUPD, R0, R1 all pack 6 consecutive GPIOs (71-76) in bits [1:6], [16:21], [0:5] etc. of single registers.

5. **Previous analysis was wrong**: The earlier `pinctrl-mt6789-ko-analysis.md` used an incorrect formula (`0x10005000 + 0x300 + GPIO_number * 0x10`). The correct formula uses 8-pin groups with 4-bit fields per pin.
