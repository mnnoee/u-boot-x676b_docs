# F26M Crystal Oscillator Enable — MT6789

## Problem Statement

The 26MHz crystal oscillator (F26M) may be **disabled** on UFS-boot boards. The Preloader (PL) initializes the PMIC and crystal oscillator for eMMC-boot boards, but on UFS-boot boards, the crystal may not be started because UFS uses its own internal oscillator. When U-Boot replaces LK, the crystal must be explicitly enabled.

## Clock Architecture

### Clock Tree (from Linux kernel DTS)

```
clk26m (fixed-clock, 26MHz)
  ├── tck_26m_mx9_ck (1:1)  → CLK_TOP_TCK_26M_MX9 (ID=55)
  ├── f26m_d2 (1:2)         → CLK_TOP_F26M_CK_D2 (ID=56)
  └── f26m_ck (1:1)         → CLK_TOP_F26M (ID=63)  ← this is the main F26M output
```

### Key Clock IDs

| ID | Name | Description |
|----|------|-------------|
| 63 | `CLK_TOP_F26M` | 26MHz fixed clock (parent for many peripherals) |
| 56 | `CLK_TOP_F26M_CK_D2` | 13MHz (F26M / 2) |
| 55 | `CLK_TOP_TCK_26M_MX9` | 26MHz (same as F26M) |

### U-Boot Clock Driver

File: `u-boot-mtk-g99/drivers/clk/mediatek/clk-mt6789.c`

```c
// Fixed clock definitions
static const struct mtk_fixed_clk top_fixed_clks[] = {
    FIXED_CLK(CLK_TOP_F26M, CLK_XTAL, 26000000),  // Line 55
    FIXED_CLK(CLK_TOP_CLKRTC, CLK_XTAL, 32000),
};

// Clock tree configuration
static const struct mtk_clk_tree mt6789_clk_tree = {
    .xtal_rate = 26 * MHZ,      // Line 171
    .xtal2_rate = 26 * MHZ,     // Line 172
    .fdivs_offs = CLK_TOP_MAINPLL_D4,  // ID=1
    .muxes_offs = CLK_TOP_AXI_SEL,     // ID=95
    // ...
};
```

**Critical**: `CLK_TOP_F26M` (ID=63) is a **fixed clock** — U-Boot's clock framework treats it as always-on. There is NO code to actually enable the physical crystal oscillator.

## Physical Crystal Oscillator Control

### Overview

The 26MHz crystal oscillator on MT6789 is controlled by:

1. **PMIC (MT6358/MT6366)** — Provides power to the crystal oscillator
2. **SPM (System Power Manager)** — Controls the crystal oscillator's power state
3. **Topckgen** — Routes the 26MHz clock to peripherals via MUX/GATE registers

### Key Register Bases

| Block | Base Address | Description |
|-------|-------------|-------------|
| Topckgen | `0x10000000` | Clock MUX/GATE configuration |
| Infracfg_AO | `0x10001000` | Clock gates, bus protection |
| SPM | `0x10006000` | System Power Manager |
| ApMixed | `0x1000C000` | PLL configuration |
| PWRAP | `0x10026000` | PMIC wrapper (for PMIC register access) |

### Topckgen Register Map (0x10000000)

| Offset | Address | Description |
|--------|---------|-------------|
| 0x000-0x07C | 0x10000000-0x1000007C | MUX registers (R/W, bits [10:8] = parent select) |
| 0x080 | 0x10000080 | GATE_CLR (write-only, write 1 to enable) |
| 0x084 | 0x10000084 | GATE_SET (write-only, write 1 to disable) |
| 0x088 | 0x10000088 | GATE_CLR_1 (write-only) |
| 0x08C | 0x1000008C | GATE_SET_1 (write-only) |
| 0x090 | 0x10000090 | GATE_STATUS_0 (read-only) |
| 0x094 | 0x10000094 | GATE_STATUS_1 (read-only) |
| 0x004 | 0x10000004 | CLK_CFG_UPDATE (write to apply MUX changes) |

**Important**: SET/CLR registers are **write-only**. Reading them always returns 0. This is normal behavior.

### MSDC30_1_SEL MUX (SD Card Clock)

- Register: `0x10000088` (CLK_CFG_2_CLR) — this is the GATE_CLR, NOT the MUX!
- **Actual MUX register**: `0x10000028` (CLK_CFG_2)
- Bits [10:8] = parent select
- Parent 0 = F26M (26MHz)

From U-Boot driver:
```c
MUX_GATE(CLK_TOP_MSDC30_1_SEL, msdc30_1_parents, 0x88, 8, 3, 15),
// offset=0x88, bit=8, width=3, gate_bit=15
```

**Wait** — offset 0x88 is the CLR register. The actual MUX value is in the direct register at offset 0x28.

### Infracfg_AO Clock Gate Registers (0x10001000)

| Group | SET Offset | CLR Offset | STA Offset |
|-------|-----------|-----------|-----------|
| IFRAO0 | 0x80 | 0x84 | 0x90 |
| IFRAO1 | 0x88 | 0x8C | 0x94 |
| IFRAO2 | 0xA4 | 0xA8 | 0xAC |
| IFRAO3 | 0xC0 | 0xC4 | 0xC8 |
| IFRAO4 | 0xE0 | 0xE4 | 0xE8 |
| IFRAO5 | 0xD0 | 0xD4 | 0xD8 |

F26M-dependent gates:

| Gate | Group | Bit | Parent |
|------|-------|-----|--------|
| AUXADC | IFRAO1 | 10 | F26M |
| AUXADC_MD | IFRAO1 | 14 | F26M |
| AUDIO_26M_BCLK | IFRAO2 | 4 | F26M |
| UNIPRO_TICK | IFRAO2 | 12 | F26M |
| UFS_SAP_BCLK | IFRAO2 | 13 | F26M |

### SPM Registers (0x10006000)

From LK decompiled code (`lk_a_lk.bin.c`):

| Address | Register | Purpose |
|---------|----------|---------|
| `0x10006000` | SPM_SPM_CTRL | SPM control (written with `0xb160001` during MD power-on) |
| `0x10006008` | SPM_SPM_CLK_CFG | SPM clock config (OR with `0x200000`) |
| `0x1000616C` | SPM_PWR_STATUS | Power domain status (bit 0 = domain ready) |
| `0x10006300` | SPM_SPM_PDN_CON | Power domain control |
| `0x10006398` | SPM_CK_PDN_CON | Clock power-down control |
| `0x100063BC` | SPM_DPM_CTRL | DPM control |
| `0x10006830` | SPM status | Status register (bits [23:22] for clock status) |

### LK MTCMOS Power-On Sequence (from `FUN_00046f14`)

```c
// Step 1: Release clock power-down
SPM_CK_PDN_CON (0x10006398) &= ~0x3;      // Clear bits [1:0]

// Step 2: Enable power domain
SPM_SPM_PDN_CON (0x10006300) |= 0x5;      // Set bits [2:0] = 0x5

// Step 3: Wait for power ready
while ((SPM_PWR_STATUS (0x1000616C) & 1) == 0) { }

// Step 4: Release bus protection
INFRA_TOPAXI_PROT_EN_CLR (0x100012A4) = 0x80;  // Clear bit 7
```

### LK MTCMOS Power-Off Sequence (from `FUN_000471ac`)

```c
// Step 1: Assert bus protection
INFRA_TOPAXI_PROT_EN_SET (0x100012A0) = 0x80;  // Set bit 7

// Step 2: Wait for protection active
while ((INFRA_TOPAXI_PROT_EN_STA (0x10001228) >> 7 & 1) == 0) { }

// Step 3: Power down domain
SPM_SPM_PDN_CON (0x10006300) &= ~0x4;      // Clear bit 2

// Step 4: Wait for power down
while ((SPM_PWR_STATUS (0x1000616C) & 1) != 0) { }

// Step 5: Disable clock and power
SPM_CK_PDN_CON (0x10006398) |= 0x3;        // Set bits [1:0]
SPM_SPM_PDN_CON (0x10006300) &= ~0x1;      // Clear bit 0
```

### PMIC Access via PWRAP

The PWRAP (PMIC Wrapper) at `0x10026000` provides access to PMIC registers:

| Address | Register | Description |
|---------|----------|-------------|
| `0x10026880` | WACS3_CMD | Write/Access command |
| `0x10026884` | WACS3_WDATA | Write data |
| `0x10026894` | WACS3_RDATA | Read data |
| `0x100268A4` | WACS3_VLD | Valid/clear |
| `0x100268A8` | WACS3_STATUS | Status (bits [3:1] = state) |

PWRAP write sequence (from `FUN_00039a00` in BL2_ext):
```c
// Wait for idle
while ((PWRAP_STATUS & 0xe) != 0) { }

// Write data
PWRAP_WDATA = value & 0xffff;
PWRAP_CMD = (pmic_addr & 0xffff) | 0x20000000;  // Write command
```

PWRAP read sequence:
```c
// Issue read
PWRAP_CMD = pmic_addr & 0xffff;  // Read command (bit 29 = 0)

// Wait for ready
while ((PWRAP_STATUS & 0xe) != 0xc) { }

// Read data
*result = (short)PWRAP_RDATA;
PWRAP_VLD = 1;  // Clear
```

## BL2_ext Analysis — Crystal NOT Enabled

### What BL2_ext Does

From decompiled `lk_a_bl2_ext.bin.c`:

1. **Thermal init** (`FUN_00039a88`, line 41694):
   - Enables LVTS clock gate: `0x10001120 |= 1` (infracfg_ao offset 0x120)
   - Sets clock gate: `0x10001124 |= 1`
   - Reads devinfo (not PWRAP — `thunk_FUN_0002ba28` is a devinfo reader)

2. **PMIC accesses** via PWRAP (`FUN_00039a00`):
   - Reads PMIC registers at addresses: 0xD88, 0xD0A, 0xD8A, 0xC2C, 0x1260, 0x120A, 0x1208, 0x10C0
   - These are **buck/LDO registers** (0xC00-0xDFF range), NOT TOP registers
   - **No access to PMIC TOP_CON (0x0004) which controls XOSC_EN**

3. **No crystal enable sequence found**:
   - No PWRAP write to PMIC address 0x0004 (TOP_CON)
   - No SPM crystal control register writes
   - Crystal oscillator is assumed to be already running

### What BL2_ext Does NOT Do

- Does NOT enable the 26MHz crystal oscillator via PMIC
- Does NOT configure topckgen MUX registers for 26MHz routing
- Does NOT release bus protection for display/MSDC domains
- Does NOT enable any infracfg_ao clock gates for F26M consumers

## Why F26M May Be Disabled on UFS-Boot Boards

### Boot Flow

```
Preloader (PL) → BL2_ext → LK/U-Boot → Kernel
```

### eMMC-Boot Boards

1. Preloader initializes PMIC (MT6358/MT6366)
2. Preloader enables 26MHz crystal oscillator
3. Preloader configures basic clocks
4. BL2_ext runs with crystal already enabled
5. LK runs with crystal already enabled

### UFS-Boot Boards

1. Preloader initializes PMIC
2. **Preloader may SKIP crystal enable** because UFS uses its own internal oscillator
3. BL2_ext runs — **crystal may be OFF**
4. LK/U-Boot runs — **crystal is OFF**
5. Kernel boots — **all F26M-dependent peripherals fail**

### Evidence

From `msdc1-pinmux-critical-fix.md`:
> "Причина: source clock не тактирует карту (F26M crystal может быть отключён на UFS-boot плате)"

From `session_crash_initr_caches.md`:
> "Краш внутри clk_get_rate(&host->src_clk)" — the MSDC clock source rate cannot be read because F26M is not running.

## Solution: Enable F26M in U-Boot

### Approach 1: Direct Register Write (Simplest)

Add crystal enable to board init (`board/mediatek/mt6789/mt6789.c`):

```c
#include <asm/io.h>

// MT6789 register bases
#define SPM_BASE        0x10006000
#define INFRACFG_AO_BASE 0x10001000

// SPM registers
#define SPM_CK_PDN_CON  (SPM_BASE + 0x398)
#define SPM_SPM_PDN_CON (SPM_BASE + 0x300)
#define SPM_PWR_STATUS  (SPM_BASE + 0x16C)

// Infracfg_AO bus protection
#define INFRA_PROT_EN_SET   (INFRACFG_AO_BASE + 0x2A0)
#define INFRA_PROT_EN_CLR   (INFRACFG_AO_BASE + 0x2A4)
#define INFRA_PROT_EN_STA   (INFRACFG_AO_BASE + 0x228)

void mt6789_enable_crystal(void)
{
    u32 val;

    // Step 1: Release clock power-down (enable F26M clock)
    val = readl(SPM_CK_PDN_CON);
    val &= ~0x3;  // Clear bits [1:0] to enable crystal clock
    writel(val, SPM_CK_PDN_CON);

    // Step 2: Enable power domain
    val = readl(SPM_SPM_PDN_CON);
    val |= 0x5;   // Set bits [2:0]
    writel(val, SPM_SPM_PDN_CON);

    // Step 3: Wait for power ready (with timeout)
    int timeout = 100000;
    while ((readl(SPM_PWR_STATUS) & 1) == 0 && timeout-- > 0)
        ;

    // Step 4: Release bus protection for DISP domain
    writel(0x00800040, INFRA_PROT_EN_CLR);  // BIT(6)|BIT(23)

    // Step 5: Wait for bus protection release
    timeout = 100000;
    while ((readl(INFRA_PROT_EN_STA) & 0x00800040) && timeout-- > 0)
        ;
}
```

### Approach 2: PWRAP Access to PMIC (More Complete)

If the crystal is controlled by the PMIC (XOSC_EN bit in TOP_CON), use PWRAP:

```c
#define PWRAP_BASE      0x10026000
#define PWRAP_WACS_CMD  (PWRAP_BASE + 0x880)
#define PWRAP_WACS_WDAT (PWRAP_BASE + 0x884)
#define PWRAP_WACS_RDAT (PWRAP_BASE + 0x894)
#define PWRAP_WACS_VLD  (PWRAP_BASE + 0x8A4)
#define PWRAP_STATUS    (PWRAP_BASE + 0x8A8)

// PMIC register addresses (MT6358/MT6366)
#define PMIC_TOP_CON    0x0004  // TOP control register

static void pwrap_write(u16 pmic_addr, u16 value)
{
    // Wait for idle
    while ((readl(PWRAP_STATUS) & 0xe) != 0)
        ;

    writel(value, PWRAP_WACS_WDAT);
    writel((pmic_addr & 0xffff) | 0x20000000, PWRAP_WACS_CMD);
}

static u16 pwrap_read(u16 pmic_addr)
{
    // Wait for idle
    while ((readl(PWRAP_STATUS) & 0xe) != 0)
        ;

    writel(pmic_addr & 0xffff, PWRAP_WACS_CMD);

    // Wait for ready
    while ((readl(PWRAP_STATUS) & 0xe) != 0xc)
        ;

    u16 val = readl(PWRAP_WACS_RDAT);
    writel(1, PWRAP_WACS_VLD);
    return val;
}

void mt6789_enable_crystal_via_pmic(void)
{
    u16 val;

    // Read PMIC TOP_CON
    val = pwrap_read(PMIC_TOP_CON);

    // Enable XOSC (bit 3 on MT6358/MT6366)
    val |= (1 << 3);
    pwrap_write(PMIC_TOP_CON, val);

    // Wait for crystal to stabilize (~5ms)
    mdelay(5);
}
```

### Approach 3: Combined (Recommended)

```c
void board_init_crystal(void)
{
    // 1. Enable crystal via PMIC (if needed)
    mt6789_enable_crystal_via_pmic();

    // 2. Release SPM clock power-down
    mt6789_release_spm_clock();

    // 3. Release bus protection
    mt6789_release_bus_protection();

    // 4. Verify crystal is running
    //    Read SPM status or topckgen MUX register
    u32 mux_val = readl(0x10000028);  // MSDC30_1_SEL MUX
    printf("Crystal enable: MUX=0x%08x\n", mux_val);
}
```

## Debugging: How to Check if Crystal is Running

### Method 1: Read SPM Status

```bash
# In U-Boot console:
md.l 0x10006830 1   # SPM status register
# Bit [23:22] should indicate clock status
```

### Method 2: Read Topckgen MUX

```bash
md.l 0x10000028 1   # MSDC30_1_SEL MUX register
# If crystal is running, MUX should show valid parent
```

### Method 3: Read Infracfg_AO Clock Gate Status

```bash
md.l 0x10001090 1   # IFRAO0 status
md.l 0x10001094 1   # IFRAO1 status (AUXADC gate at bit 10)
```

### Method 4: Check PWRAP

```bash
# Read PMIC TOP_CON via PWRAP
# This requires PWRAP access which may not be available in simple console
```

## Important Notes

1. **SET/CLR registers are write-only**: Reading `0x10000088` (GATE_CLR) always returns 0. This is NORMAL. Read the STATUS register (`0x10000090`) instead.

2. **Bus protection must be released BEFORE accessing protected domains**: Write to CLR register first, then check STA register.

3. **Crystal stabilization time**: After enabling the crystal, wait at least 5ms before using the clock.

4. **The crystal oscillator is shared**: Once enabled, it stays running. No need to disable it.

5. **PMIC register addresses**: The exact XOSC_EN bit location depends on the PMIC variant (MT6358 vs MT6366). Check the PMIC datasheet for your specific device.

## References

- `src_linux/linux/drivers/clk/mediatek/clk-mt6789-topckgen.c` — Linux topckgen driver
- `src_linux/linux/drivers/clk/mediatek/clk-mt6789-infra_ao.c` — Linux infracfg_ao driver
- `src_linux/linux/drivers/pmdomain/mediatek/mt6789-pm-domains.h` — Bus protection entries
- `src_linux/linux/include/dt-bindings/clock/mediatek,mt6789-clk.h` — Clock IDs
- `src_linux/linux/include/linux/mfd/mt6358/registers.h` — PMIC register definitions
- `lk_unpack_v0/lk_a_lk.bin.c` — LK decompiled (MTCMOS sequences)
- `lk_unpack_v0/lk_a_bl2_ext.bin.c` — BL2_ext decompiled (no crystal enable found)
- `u-boot-mtk-g99/drivers/clk/mediatek/clk-mt6789.c` — U-Boot clock driver
- `docs/AGENTS/topckgen-bus-protection.md` — Bus protection details
- `docs/AGENTS/msdc1-pinmux-critical-fix.md` — F26M crystal issue on UFS boards
- `docs/AGENTS/session_crash_initr_caches.md` — Clock driver crash analysis
