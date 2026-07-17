# MT6789 Pinctrl Analysis: MSDC1 Pins (GPIO 71-76)

## Source Files Analyzed
- `kernelmodules/pseudocode/pinctrl-mt6789.ko.c` — decompiled .ko (minimal, thin wrapper)
- `src_linux/linux/drivers/pinctrl/mediatek/pinctrl-mt6789.c` — **actual pinctrl driver** (full register tables)
- `src_linux/linux/drivers/pinctrl/mediatek/pinctrl-mtk-mt6789.h` — pin function definitions
- `src_linux/linux/include/dt-bindings/pinctrl/mediatek,mt6789-pinfunc.h` — DT-binding function numbers
- `src_linux/linux/arch/arm64/boot/dts/mediatek/mt6789.dtsi` — IOCFG base addresses

## IOCFG Bank Base Addresses (from DTS)

| Bank Name | reg-name | Physical Base |
|-----------|----------|---------------|
| gpio (base) | "base" | 0x10005000 |
| iocfg_rb | "rb" | 0x11c30000 |
| iocfg_bm | "bm" | 0x11d10000 |
| iocfg_br | "br" | 0x11d40000 |
| iocfg_lm | "lm" | 0x11e20000 |
| iocfg_bl | "bl" | 0x11e60000 |
| iocfg_rt | "rt" | 0x11ea0000 |
| eint | "eint" | 0x1000b000 |

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

Source: `pinctrl-mtk-mt6789.h` lines 757-824, `mediatek,mt6789-pinfunc.h` lines 471-513

## Register Layout for GPIO 71-76

### How Register Addressing Works

The pinctrl-mtk framework uses field calculation tables. Each entry defines:
- `i_base`: index into base address array (0=base, 1=rb, 2=bm, 3=br, 4=lm, 5=bl, 6=rt)
- `s_addr`: start address offset from base
- `x_addrs`: stride between pins (when in a range)
- `s_bit`: start bit within register
- `x_bits`: number of bits per pin

**GPIO 71-76 are in bank "rt" (0x11ea0000) for IES/SMT/PUPD/R0/R1/DRV**
**All pins share bank "base" (0x10005000) for mode/dir/di/do**

### Pin Mode Register (Function Select)

- **Base bank**: 0 (base = 0x10005000)
- **Offset**: 0x0300 + pin_number * 0x10
- **Width**: 4 bits [3:0]
- **Range**: PIN_FIELD(0, 187, 0x0300, 0x10, 0, 4)

| GPIO | Mode Register Address | Bits | Value for MSDC1 |
|------|-----------------------|------|-----------------|
| 71 (CLK) | **0x10005770** | [3:0] | 1 |
| 72 (CMD) | **0x10005780** | [3:0] | 1 |
| 73 (DAT0) | **0x10005790** | [3:0] | 1 |
| 74 (DAT1) | **0x100057A0** | [3:0] | 1 |
| 75 (DAT2) | **0x100057B0** | [3:0] | 1 |
| 76 (DAT3) | **0x100057C0** | [3:0] | 1 |

Calculation: 0x10005000 + 0x300 + (GPIO_number * 0x10)

### Direction Register

- **Base bank**: 0 (base = 0x10005000)
- **Offset**: 0x0000 + pin_number * 0x10
- **Width**: 1 bit [0] — 1=output, 0=input

| GPIO | Direction Register | Bit |
|------|-------------------|-----|
| 71 | 0x10005470 | [0] |
| 72 | 0x10005480 | [0] |
| 73 | 0x10005490 | [0] |
| 74 | 0x100054A0 | [0] |
| 75 | 0x100054B0 | [0] |
| 76 | 0x100054C0 | [0] |

### IES (Input Enable / IO Strap)

- **Base bank**: 6 (rt = 0x11ea0000)
- **All pins in same register**: 0x11ea0080
- **Individual bits per pin**

| GPIO | IES Register | Bit |
|------|-------------|-----|
| 71 | 0x11EA0080 | [1] |
| 72 | 0x11EA0080 | [2] |
| 73 | 0x11EA0080 | [3] |
| 74 | 0x11EA0080 | [4] |
| 75 | 0x11EA0080 | [5] |
| 76 | 0x11EA0080 | [6] |

### SMT (Schmitt Trigger)

- **Base bank**: 6 (rt = 0x11ea0000)
- **All pins in same register**: 0x11ea0140
- **Individual bits per pin**

| GPIO | SMT Register | Bit |
|------|-------------|-----|
| 71 | 0x11EA0140 | [16] |
| 72 | 0x11EA0140 | [17] |
| 73 | 0x11EA0140 | [18] |
| 74 | 0x11EA0140 | [19] |
| 75 | 0x11EA0140 | [20] |
| 76 | 0x11EA0140 | [21] |

### Pull Configuration (PUPD + R0 + R1 Type)

GPIO 71-76 use **MTK_PULL_PUPD_R1R0_TYPE** (4-resistor pull, NOT simple PU/PD)

This means three registers control pull behavior:
1. PUPD register — selects pull-up (1) or pull-down (0)
2. R0 register — resistor selector bit 0
3. R1 register — resistor selector bit 1

**Pull-Up/Down Select (PUPD)**

- **Base bank**: 6 (rt = 0x11ea0000)
- **All pins in same register**: 0x11ea00b0
- **Individual bits per pin**

| GPIO | PUPD Register | Bit |
|------|--------------|-----|
| 71 | 0x11EA00B0 | [0] |
| 72 | 0x11EA00B0 | [1] |
| 73 | 0x11EA00B0 | [2] |
| 74 | 0x11EA00B0 | [3] |
| 75 | 0x11EA00B0 | [4] |
| 76 | 0x11EA00B0 | [5] |

**R0 Resistor Selector**

- **Base bank**: 6 (rt = 0x11ea0000)
- **All pins in same register**: 0x11ea00e0
- **Individual bits per pin**

| GPIO | R0 Register | Bit |
|------|------------|-----|
| 71 | 0x11EA00E0 | [0] |
| 72 | 0x11EA00E0 | [1] |
| 73 | 0x11EA00E0 | [2] |
| 74 | 0x11EA00E0 | [3] |
| 75 | 0x11EA00E0 | [4] |
| 76 | 0x11EA00E0 | [5] |

**R1 Resistor Selector**

- **Base bank**: 6 (rt = 0x11ea0000)
- **All pins in same register**: 0x11ea00f0
- **Individual bits per pin**

| GPIO | R1 Register | Bit |
|------|------------|-----|
| 71 | 0x11EA00F0 | [0] |
| 72 | 0x11EA00F0 | [1] |
| 73 | 0x11EA00F0 | [2] |
| 74 | 0x11EA00F0 | [3] |
| 75 | 0x11EA00F0 | [4] |
| 76 | 0x11EA00F0 | [5] |

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

### Drive Strength

- **Base bank**: 6 (rt = 0x11ea0000)
- **All pins in same register**: 0x11ea0030
- **3 bits per pin**

| GPIO | Drive Register | Bits |
|------|---------------|------|
| 71 | 0x11EA0030 | [11:9] |
| 72 | 0x11EA0030 | [14:12] |
| 73 | 0x11EA0030 | [17:15] |
| 74 | 0x11EA0030 | [20:18] |
| 75 | 0x11EA0030 | [23:21] |
| 76 | 0x11EA0030 | [26:24] |

Drive strength values (DRV_GRP4 — 3-bit drive):
- 0 = 4mA (minimum)
- 1 = 8mA
- 2 = 12mA
- 3 = 16mA (maximum)

## Complete Register Map Summary

```
GPIO 71 (MSDC1_CLK) — Function 1
  Mode:    0x10005770 [3:0] = 1
  Dir:     0x10005470 [0]   = 0 (input) or 1 (output)
  IES:     0x11EA0080 [1]   = 1 (enable)
  SMT:     0x11EA0140 [16]  = 1 (enable Schmitt)
  PUPD:    0x11EA00B0 [0]   = 1 (pull-up)
  R0:      0x11EA00E0 [0]   = 1
  R1:      0x11EA00F0 [0]   = 0  → 24K pull-up
  DRV:     0x11EA0030 [11:9] = 2 (12mA)

GPIO 72 (MSDC1_CMD) — Function 1
  Mode:    0x10005780 [3:0] = 1
  Dir:     0x10005480 [0]   = 0 (input) or 1 (output)
  IES:     0x11EA0080 [2]   = 1 (enable)
  SMT:     0x11EA0140 [17]  = 1 (enable Schmitt)
  PUPD:    0x11EA00B0 [1]   = 1 (pull-up)
  R0:      0x11EA00E0 [1]   = 1
  R1:      0x11EA00F0 [1]   = 0  → 24K pull-up
  DRV:     0x11EA0030 [14:12] = 2 (12mA)

GPIO 73 (MSDC1_DAT0) — Function 1
  Mode:    0x10005790 [3:0] = 1
  Dir:     0x10005490 [0]   = 0 (input) or 1 (output)
  IES:     0x11EA0080 [3]   = 1 (enable)
  SMT:     0x11EA0140 [18]  = 1 (enable Schmitt)
  PUPD:    0x11EA00B0 [2]   = 1 (pull-up)
  R0:      0x11EA00E0 [2]   = 1
  R1:      0x11EA00F0 [2]   = 0  → 24K pull-up
  DRV:     0x11EA0030 [17:15] = 2 (12mA)

GPIO 74 (MSDC1_DAT1) — Function 1
  Mode:    0x100057A0 [3:0] = 1
  Dir:     0x100054A0 [0]   = 0 (input) or 1 (output)
  IES:     0x11EA0080 [4]   = 1 (enable)
  SMT:     0x11EA0140 [19]  = 1 (enable Schmitt)
  PUPD:    0x11EA00B0 [3]   = 1 (pull-up)
  R0:      0x11EA00E0 [3]   = 1
  R1:      0x11EA00F0 [3]   = 0  → 24K pull-up
  DRV:     0x11EA0030 [20:18] = 2 (12mA)

GPIO 75 (MSDC1_DAT2) — Function 1
  Mode:    0x100057B0 [3:0] = 1
  Dir:     0x100054B0 [0]   = 0 (input) or 1 (output)
  IES:     0x11EA0080 [5]   = 1 (enable)
  SMT:     0x11EA0140 [20]  = 1 (enable Schmitt)
  PUPD:    0x11EA00B0 [4]   = 1 (pull-up)
  R0:      0x11EA00E0 [4]   = 1
  R1:      0x11EA00F0 [4]   = 0  → 24K pull-up
  DRV:     0x11EA0030 [23:21] = 2 (12mA)

GPIO 76 (MSDC1_DAT3) — Function 1
  Mode:    0x100057C0 [3:0] = 1
  Dir:     0x100054C0 [0]   = 0 (input) or 1 (output)
  IES:     0x11EA0080 [6]   = 1 (enable)
  SMT:     0x11EA0140 [21]  = 1 (enable Schmitt)
  PUPD:    0x11EA00B0 [5]   = 1 (pull-up)
  R0:      0x11EA00E0 [5]   = 1
  R1:      0x11EA00F0 [5]   = 0  → 24K pull-up
  DRV:     0x11EA0030 [26:24] = 2 (12mA)
```

## How to Configure MSDC1 Pins from U-Boot

To set GPIO 71-76 to MSDC1 function (function=1), write to the mode registers:

```
# Set GPIO71-76 mode to 1 (MSDC1 function)
mw.l 0x10005770 0x1   # GPIO71 = MSDC1_CLK
mw.l 0x10005780 0x1   # GPIO72 = MSDC1_CMD
mw.l 0x10005790 0x1   # GPIO73 = MSDC1_DAT0
mw.l 0x100057A0 0x1   # GPIO74 = MSDC1_DAT1
mw.l 0x100057B0 0x1   # GPIO75 = MSDC1_DAT2
mw.l 0x100057C0 0x1   # GPIO76 = MSDC1_DAT3

# Enable IES (input enable) for all 6 pins
# Set bits [6:1] in 0x11EA0080
mw.l 0x11EA0080 0x7E  # bits 1-6 = 1

# Enable SMT (Schmitt trigger) for all 6 pins
# Set bits [21:16] in 0x11EA0140
mw.l 0x11EA0140 0x3F0000  # bits 16-21 = 1

# Set pull-up 24K for all 6 pins
# PUPD: bits [5:0] = 0x3F (all 1 = pull-up)
# R0:   bits [5:0] = 0x3F (all 1)
# R1:   bits [5:0] = 0x00 (all 0)
mw.l 0x11EA00B0 0x3F   # PUPD: pull-up
mw.l 0x11EA00E0 0x3F   # R0: resistor select
mw.l 0x11EA00F0 0x00   # R1: resistor select → 24K

# Set drive strength (default 12mA = value 2)
# DRV bits packed: GPIO71[11:9]=2, GPIO72[14:12]=2, etc.
# Each 3-bit field = 2 (010), packed:
# [26:24]=010 [23:21]=010 [20:18]=010 [17:15]=010 [14:12]=010 [11:9]=010
# = 0x09249200 → but need to preserve other bits
# Read-modify-write: set bits [26:9] to pattern 010_010_010_010_010_010
```

## Decompiled .ko Pseudocode Notes

The decompiled `pinctrl-mt6789.ko.c` is only 271 lines and contains:
- `mt6789_pinctrl_probe()` — calls `mtk_paris_pinctrl_probe(param_1, &mt6789_data)`
- `init_module()` — calls `__platform_driver_register(&mt6789_pinctrl_driver, &__this_module)`
- CFI check functions

The actual pin configuration data lives in the `mt6789_data` static symbol (a `struct mtk_pin_soc`) which references:
- `mt6789_reg_cals` — register calculation tables (mode, dir, di, do, ies, pu, pd, drv, pupd, r0, r1, rsel)
- `mtk_pins_mt6789[]` — pin description table with all 188 pins and their functions
- `mt6789_pull_type[]` — per-pin pull type (PU_PD vs PUPD_R1R0 vs PU_PD_RSEL)

The decompiled pseudocode did NOT capture these data structures. The actual kernel source was used instead.
