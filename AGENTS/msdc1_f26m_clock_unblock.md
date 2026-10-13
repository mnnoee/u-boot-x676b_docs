# MSDC1 F26M Clock Unblock — MT6789

## Problem
SD card (MSDC1, 0x11240000) doesn't respond to CMD0 — response registers all zeros.
Source clock (F26M crystal) may not be running on UFS-boot board.

## Root Cause Analysis

### 1. U-Boot Clock Driver Bug (FIXED in v180)
U-Boot used `MUX_GATE` macro with CLR register address (0x88) for MSDC30_1_SEL:
```c
MUX_GATE(CLK_TOP_MSDC30_1_SEL, msdc30_1_parents, 0x88, 8, 3, 15),
```
**Problem**: `clk_enable()` reads CLR register (returns 0 on write-only), writes back → no-op.
Gate bit 15 never gets cleared, clock stays disabled.

**Fix**: Manual struct initialization with CLK_MUX_SETCLR_UPD flag:
```c
{
    .id = CLK_TOP_MSDC30_1_SEL,
    .mux_reg = 0x80,        // VAL register (read-only)
    .mux_set_reg = 0x84,    // SET register
    .mux_clr_reg = 0x88,    // CLR register
    .upd_reg = 0x04,        // UPDATE register
    .upd_shift = 29,        // UPDATE bit
    .mux_shift = 8,         // MUX bits [10:8]
    .mux_mask = 0x7,        // 3-bit mask
    .gate_reg = 0x80,       // Gate in VAL register
    .gate_shift = 15,       // Gate bit 15
    .parent = msdc30_1_parents,
    .num_parents = ARRAY_SIZE(msdc30_1_parents),
    .flags = CLK_MUX_SETCLR_UPD,
},
```
With `CLK_MUX_SETCLR_UPD`, framework uses SET/CLR registers properly:
- `clk_enable()`: writes BIT(15) to mux_clr_reg (0x88) → clears gate bit
- `set_parent()`: writes mask to clr, index to set, then UPDATE

### 2. Bus Protection (FIXED in v180)
INFRA bus protection blocks access to clock gates. BL2_ext pattern:
```c
writel(0x18000000, 0x1000108C);  // SET flash domain access
```

### 3. SPM Crystal Enable (EXPERIMENTAL in v180)
SPM registers written to attempt F26M crystal enable:
```c
// Deassert SW reset for MD domain
writel(val & ~0x03, 0x10006398);
// Power on sequence
writel(val | 0x05, 0x10006300);
```
**WARNING**: These are MD power domain registers (0x10006398 = MD ext_buck_iso, 0x10006300 = MD ctl).
May affect modem subsystem. Monitor for side effects.

### 4. INFRA Clock Gates (FIXED in v180)
Previously skipped due to bus protection hang:
```c
writel(BIT(4), 0x10001088);   // CLK_IFRAO_MSDC1 (hclk)
writel(BIT(16), 0x10001088);  // CLK_IFRAO_MSDC1_SRC (source_cg)
```

## Register Map

### TopCKGen (0x10000000)
| Register | Address | Description |
|----------|---------|-------------|
| MSDC30_1_SEL VAL | 0x10000080 | MUX state (read-only) |
| MSDC30_1_SEL SET | 0x10000084 | SET bits (write-1) |
| MSDC30_1_SEL CLR | 0x10000088 | CLR bits (write-1) |
| CLK_CFG_UPDATE | 0x10000004 | Bit 29 = apply MUX change |

### INFRA_AO (0x10001000)
| Register | Address | Description |
|----------|---------|-------------|
| infra_ao1 SET | 0x10001088 | Clock gate SET |
| infra_ao1 CLR | 0x1000108C | Clock gate CLR |
| infra_ao1 STA | 0x10001094 | Clock gate status |

### MSDC30_1_SEL MUX parents
| Index | Parent | Frequency |
|-------|--------|-----------|
| 0 | F26M | 26 MHz |
| 1 | univpll_d6_d2 | 160 MHz |
| 2 | mainpll_d6_d2 | 178 MHz |
| 3 | mainpll_d7_d2 | 152.6 MHz |
| 4 | msdcpll_d2 | ~400 MHz |

## Files Modified (v180)
1. `drivers/clk/mediatek/clk-mt6789.c` — MSDC30_1_SEL MUX with CLK_MUX_SETCLR_UPD
2. `drivers/mmc/mtk-sd.c` — bus protection release, SPM crystal enable, INFRA clocks

## Build
```bash
cd /workdir/git/heliog99/u-boot-mtk-g99
make CROSS_COMPILE=aarch64-linux-gnu- mt6789_infinix_x676b_defconfig
make CROSS_COMPILE=aarch64-linux-gnu- -j$(nproc)
/workdir/git/heliog99/mtkclient/venv/bin/python3 gimme-uboot.py \
  /workdir/git/heliog99/u-boot-x676b/lk_a_dump.img \
  u-boot.bin \
  /workdir/git/heliog99/u-boot-x676b/uboot-lk-x676b-v180.img
```

## Flash & Test
```bash
# Close serial port first!
# Flash via mtkclient
cd /workdir/git/heliog99/mtkclient
python mtk w lk_a ../u-boot-x676b/uboot-lk-x676b-v180.img

# Test
picocom -b 921600 /dev/ttyACM1
# Boot device, interrupt U-Boot
mmc rescan
mmc info
```

## Expected EARLY_W output sequence
```
msdc:ungate:release_bus_prot
msdc:bus_prot:released
msdc:ungate:spm_crystal
msdc:f26m:spm_pdn_con
msdc:f26m:spm_power_ready  (or spm_reset_deassert + spm_power_on)
msdc:ungate:src_clk
msdc:ungate:infra_clks
msdc:infra_clks:enabled
msdc:ungate:done
```

## Fallback
If F26M still not working, switch MUX to MSDCPLL_D2 (parent index 4):
```
mw.l 0x10000088 0x00000700   # CLR bits [10:8]
mw.l 0x10000084 0x00000400   # SET index 4 (msdcpll_d2)
mw.l 0x10000004 0x20000000   # UPDATE bit 29
```
