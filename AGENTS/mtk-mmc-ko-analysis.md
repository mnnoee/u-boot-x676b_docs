# mtk-mmc.ko — Decompiled Driver Analysis for MSDC1 (SD Card)

**Source**: `kernelmodules/pseudocode/mtk-mmc.ko.c` (3920 lines)
**Device**: Infinix X676B, MT6789 (Helio G99), MSDC1 @ `0x11240000`

---

## 1. Module Registration

| Item | Value |
|------|-------|
| `init_module()` | Line 3896 — calls `__platform_driver_register(&mt_msdc_driver, &__this_module)` |
| `cleanup_module()` | Line 3909 — calls `platform_driver_unregister(&mt_msdc_driver)` |
| `mt_msdc_driver` | Global pointer (line 194) — platform driver struct, probe = `msdc_drv_probe_cfi_jt` |

---

## 2. Probe Function: `msdc_drv_probe` (line 803)

This is the main initialization entry point. The driver allocates an `mmc_host` structure of size `0x570` bytes.

### 2.1 DT and I/O Mapping

| Step | Line | Action |
|------|------|--------|
| Check DT | 829 | `param_1[0x5e]` must be non-zero (of_node) |
| Alloc host | 834 | `mmc_alloc_host(0x570, puVar1)` — host structure size |
| Parse DT | 839 | `mmc_of_parse()` |
| Map regs | 842 | `devm_platform_ioremap_resource(param_1, 0)` → stored at `plVar11[0xf8]` — this is MSDC1 base `0x11240000` |
| Optional 2nd resource | 845 | `platform_get_resource(param_1, 0x200, 1)` → `plVar11[0xf9]` — top control regs |

### 2.2 Clock Acquisition (lines 857–908)

All clocks obtained via `devm_clk_get` / `devm_clk_get_optional`:

| Clock Name | Host Offset | Required? | DT Source | Description |
|------------|-------------|-----------|-----------|-------------|
| `"source"` | `plVar11[0x118]` | YES | `CLK_TOP_MSDC30_1_SEL` (116) | Source clock from topckgen mux |
| `DAT_0010c31e` | `plVar11[0x119]` | YES | — | Likely `"hclk"` — infra AHB clock |
| `"bus_clk"` | `plVar11[0x11a]` | Optional | — | Bus clock |
| `"crypto_clk"` | `plVar11[0x11b]` | Optional | — | Crypto engine clock |
| `"source_cg"` | `plVar11[0x11d]` | Optional | `CLK_IFRAO_MSDC1_SRC` (28) | Source clock gate |
| `"sys_cg"` | `plVar11[0x11e]` | Optional | — | System clock gate |
| `"crypto_cg"` | `plVar11[0x11c]` | Optional | — | Crypto clock gate |
| `"pclk_cg"` | `plVar11[0x11f]+0` | Optional | — | APB clock gate (bulk) |
| `"axi_cg"` | `plVar11[0x121]+0` | Optional | — | AXI clock gate (bulk) |
| `"ahb_cg"` | `plVar11[0x123]+0` | Optional | — | AHB clock gate (bulk) |

**Clock enable sequence in probe** (lines 894–914):
1. `clk_prepare(plVar11[0x11e])` — sys_cg (or hclk)
2. `clk_enable(plVar11[0x11e])`
3. Bulk prepare+enable 3 clocks: pclk_cg, axi_cg, ahb_cg

**DAT_0010c31e** — this string is the second required clock. Based on the existing `sd_card_support.md` and the DTS, this is `"hclk"` (CLK_IFRAO_MSDC1, infra AHB bus clock).

### 2.3 Reset Control (line 920)

```
__devm_reset_control_get(puVar1, &DAT_0010c4fb, 0, 0, 1, 1)
→ plVar11[0x117]
```
DAT_0010c4fb is likely `"msdc"` — the reset control name.

### 2.4 IRQ (line 928)

```c
platform_get_irq(param_1, 0) → *(plVar11 + 0x115)
```
GIC SPI 135 for MSDC1.

### 2.5 Pinctrl (lines 934–948)

```c
devm_pinctrl_get(puVar1)                  → plVar11[0x101]  // pinctrl handle
pinctrl_lookup_state(pHandle, "default")  → plVar11[0x102]  // default pin state
pinctrl_lookup_state(pHandle, "state_uhs") → plVar11[0x103] // UHS pin state
```

Pin state switching happens in `msdc_ops_switch_volt` (line 3056):
- 1.8V mode → selects `plVar11[0x101]` (stored at `param_1 + 0x808`)  
  Actually `plVar11[0x103]` = state_uhs (for 1.8V/UHS)
- 3.3V mode → selects `plVar11[0x102]` = default

```c
// In msdc_ops_switch_volt (line 3033):
if (signal_voltage == 1.8V)
    pinctrl_select_state(plVar11[0x101], plVar11[0x103]); // state_uhs
else
    pinctrl_select_state(plVar11[0x101], plVar11[0x102]); // default
```

### 2.6 DT Properties Read (lines 949–1001)

| Property | Host Offset | Description |
|----------|-------------|-------------|
| `"mediatek,latch-ck"` | `plVar11 + 0x934` | Latch clock setting |
| `"hs400-ds-delay"` | `plVar11 + 0x127` | HS400 data strobe delay |
| `"mediatek,hs200-cmd-int-delay"` | `plVar11 + 0x93c` | HS200 CMD interrupt delay |
| `"mediatek,hs400-cmd-int-delay"` | `plVar11 + 0x128` | HS400 CMD interrupt delay |
| `"host-index"` | `plVar11 + 0x14b` | **Host index** (1 for MSDC1) |
| `"ocr-voltage"` | `plVar11 + 0xc94` | OCR voltage mask |
| `"mediatek,hs400-cmd-resp-sel-rising"` | `plVar11 + 0x944` | Boolean |
| `"supports-cqe"` | `plVar11 + 0x947` | Command Queue Engine support |
| `"mediatek,mmc-qos"` | `plVar11 + 0x19b` | QoS support |

### 2.7 Host Index Check and Event Handler (lines 1002–1008)

**Critical for MSDC1**: When `host-index == 1`:
```c
plVar11[0x142] = (long)msdc_sd_event_cfi_jt;   // SD card event callback
plVar11[0x145] = 0xfffffffe0;                      // event mask
plVar11[0x146] = (long)(plVar11 + 0x146);          // work queue head (self-linked list)
plVar11[0x147] = (long)(plVar11 + 0x146);          // work queue tail
plVar11[0x148] = (long)sdcard_oc_handler_cfi_jt;  // overcurrent handler
```

### 2.8 Source Clock Rate (line 1012)

```c
iVar8 = clk_get_rate(plVar11[0x118]);  // source clock rate
*(int *)((long)plVar11 + 0x92c) = iVar8;  // store source clock rate
```

### 2.9 Clock Divider Calculation (lines 1018–1030)

```c
iVar10 = 0x3fb;                    // = 1019
if (*(char *)plVar11[0xf1] != '\b')  // if not 8-bit capable
    iVar10 = 0x3ffb;               // = 16379
uVar7 = 0x3fc;                     // = 1020
if (*(char *)plVar11[0xf1] != '\b')
    uVar7 = 0x3ffc;               // = 16380
uVar6 = (iVar10 + iVar8) / uVar7;  // source_rate / 1020 (or 16380)
*(uint *)(plVar11 + 0x74) = uVar6;  // stored at host offset 0x74
```

**This calculates the source clock frequency in kHz for timeout calculations.**

For MT6789 MSDC1 with typical source clock ~208 MHz:
- 208000 / 1020 ≈ 203 (for 8-bit), or 208000 / 16380 ≈ 12 (for non-8-bit)

### 2.10 Card Detection GPIO Fallback (lines 1031–1035)

```c
if ((no-gpio-cd bit 0 of *(plVar11 + 0x3d5)) == 0 &&
    mmc_can_gpio_cd(plVar11) == 0 &&
    *(char *)(plVar11[0xf1] + 0xe) != 0) {
    *(undefined1 *)((long)plVar11 + 0x946) = 1;  // Enable custom CD method
}
```

When GPIO CD is not available and the DT enables it, the driver uses a **custom card detect** method (bit-manipulation of register state, line 423):
```c
// msdc_get_cd (line 407), when custom CD active:
uVar1 = (*(uint *)(param_1 + 0x3d8) >> 10 ^
         *(uint *)(*(long *)(param_1 + 0x7c0) + 8) >> 1 ^ 0xffffffff) & 1;
```
This reads MSDC registers to detect card presence when GPIO is unavailable.

### 2.11 CQE (Command Queue Engine) Setup (lines 1058–1076)

```c
if (*(char *)((long)plVar11 + 0x3da) < 0) {  // CQE flag
    lVar13 = devm_kmalloc(*plVar11, 0xd8, 0xdc0);
    plVar11[0x141] = lVar13;
    *(uint *)(lVar13 + 0x30) |= 1;  // Enable CQE
    *(long *)(plVar11[0x141] + 8) = plVar11[0xf8] + 0x800;  // CQE regs = MSDC base + 0x800
    cqhci_init(plVar11[0x141], plVar11, 1);
}
```

**CQE registers are at MSDC1 base + 0x800 = 0x11240800**

### 2.12 DMA Setup (lines 1079–1125)

```c
*(undefined4 *)((long)plVar11 + 0x804) = 0x300000;  // DMA config
// Allocate DMA descriptor ring: 0x38 bytes
dma_alloc_attrs(puVar1, 0x38, plVar11 + 0xfd, 0xcc0, 0);
// Allocate DMA buffer: 0x4000 bytes (16KB)
dma_alloc_attrs(puVar1, 0x4000, plVar11 + 0xfe, 0xcc0, 0);
```

### 2.13 Init Hardware and IRQ Registration (lines 1133–1162)

```c
msdc_init_hw(plVar2);                    // Line 1133 - Initialize hardware
devm_request_threaded_irq(..., msdc_irq_cfi_jt, ...);  // Line 1134-1135

// SDIO EINT (if host-index == 2):
if ((int)*plVar3 == 2) {
    devm_gpiod_get_index(*plVar2, &DAT_0010bbaf, 0, 1);  // "sdio" GPIO
    gpiod_to_irq(uVar17);
    devm_request_threaded_irq(..., sdio_eint_irq_cfi_jt, 0x2008, "sdio-eint", plVar2);
}

pm_runtime_set_autosuspend_delay(plVar11[0xf0], 0x32);  // 50ms autosuspend
pm_runtime_enable(plVar11[0xf0]);
mmc_add_host(plVar11);                  // Register with MMC core
```

---

## 3. Hardware Init: `msdc_init_hw` (line 1564)

This initializes the MSDC controller registers.

### 3.1 Reset Sequence

```
Line 1574: if (reset control exists)
    reset_control_assert()
    usleep_range(10, 0x32)   // 10-50us delay
    reset_control_deassert()
```

### 3.2 Register Initialization

`param_1[8]` = MSDC1 base register address (0x11240000)

| Offset | Register | Value/Action | Description |
|--------|----------|--------------|-------------|
| `+0x00` (MSDC_CFG) | `param_1[8]+0` | `\|= 1` (bit 0) | Enable MSDC |
| `+0x10` (INT_MASK) | `param_1[8]+0x10` | `= 0` | Disable all interrupts |
| `+0x0C` (INT_STS) | `param_1[8]+0x0C` | Read (clear by read) | Clear pending interrupts |
| `+0x08` (SW_RST) | `param_1[8]+0x08` | Bit 2 set then poll | Software reset |

**msdc_reset_hw** (line 2015):
```
Bit 2 of MSDC_CFG (offset 0x00) set → software reset
Poll bit 2 until clear
Bit 31 of PATCH_BIT0 (offset 0x14) set → flush FIFO
Poll until bit 31 clear
Read INT_STS (offset 0x0C) to clear
```

### 3.3 Non-CQE Mode (line 1588–1622)

```
offset 0x30 (SDC_CFG):
  Bit 1 cleared → disable SDIO
  Bit 0 cleared → disable SD card mode (for CMD without data)

offset 0x08 (SW_RST):
  Bit 1 cleared → no DMA reset
```

### 3.4 CMD/DAT Pull Configuration (line 1615–1634)

```
offset 0xB8 (PAD_TUNE0):
  If no pad_tune1:
    Bits [18:16] = 2 → CMD pull strength
    Bit 30 = 1 → CMD pull enable
  If pad_tune1 exists:
    offset 0x64 (PAD_TUNE1):
      Bit 19 = 1 → DAT pull enable
```

### 3.5 Tuning Register Configuration (line 1636–1672)

```
offset 0xB8 (PAD_TUNE0):
  If data_tune flag set: |= 2 (bit 1)

offset 0x64 (PAD_CMD_TD_SEL / new_tx_rx):
  If flag set: |= 0x80000000

offset 0x68 (EMMC50_CFG):
  If flag set: |= 1

offset 0x30 (SDC_CFG):
  Bit 19 = 1 → SWRITE_TIMEOUT
  Bit 20 = 0 → clear
  Bit 19 = 1 → write timeout enable

offset 0x64:
  |= 0x80000 → busy check enable

offset 0x30:
  Bits [25:24] = 3 → bus width = 8-bit (will be overridden)
```

### 3.6 Tune Setting Init

```
msdc_init_tune_setting(param_1)  // line 1673 — called but not in this file
```

---

## 4. Clock Ungating: `msdc_ungate_clock` (line 1413)

Called during runtime resume and initial probe.

`param_1` is the clock field sub-struct (plVar11 + 0x20 region).

| Order | Field Offset | Clock | Action |
|-------|-------------|-------|--------|
| 1 | `param_1[0x2b]` | (6th clock) | prepare + enable |
| 2 | `param_1[0x29]` | (4th clock — source) | prepare + enable |
| 3 | `param_1[0x2a]` | (5th clock) | prepare + enable |
| 4 | `param_1[0x28]` | (3rd clock — hclk?) | prepare + enable |
| 5 | `param_1[0x2c]` | (7th clock) | prepare + enable |
| 6 | `param_1[0x2d]` | (8th clock) | prepare + enable |
| 7 | `param_1[0x2f]` (bulk×3) | pclk/axi/ahb | bulk prepare + enable |

**After enabling, waits for bit 7 of status register to be set** (line 1458):
```c
while ((*(uint *)param_1[8] >> 7 & 1) == 0)
    Yield();
```
This waits for the MSDC controller clock to be stable.

---

## 5. Clock Gating (Suspend): `msdc_runtime_suspend` (line 1326)

Reverse of ungate — disables in reverse order:
1. Bulk disable/unprepare 3 clocks (pclk/axi/ahb)
2. disable/unprepare each of 6 individual clocks
3. Saves timing settings
4. Disables SDIO interrupt if applicable

---

## 6. Clock Frequency and Divider: `msdc_set_mclk` (line 3325)

Sets the MSDC output clock (sclk) based on timing mode and target frequency.

### 6.1 Source Clock Field

```c
uVar9 = *(uint *)((long)param_1 + 0x1ac);  // Stored source clock rate (kHz)
```

This was set during probe from `clk_get_rate(plVar11[0x118])`.

### 6.2 Timing-Based Divider Calculation

**UHS/SDR modes** (timing < 0xB, i.e., SDR12/25/50/104/DDR50):

```
uVar9 = source_rate >> 2   // source / 4 (base divider)
If target_freq < uVar9:
    div = source / (target × 4)
    hw_div = div / 2       // actual HW divider

For SDR104 (timing = 8):
    If target_freq < source_rate/2:
        pad_div = source_rate/2
        divider_mode = 3
        hw_div = pad_div (no pre-divide)
    Else:
        HW bit set for DDR mode
        pad_div = source_rate/2
```

**Legacy modes** (timing >= 0xB, MMC/SD legacy):

```
uVar9 = source_rate (full)
If target < source:
    If target < source/2:
        div = source/(4×target)
        hw_div = (source/4) / div
    Else:
        hw_div = source/2
Else:
    divider_mode = 1  // no division
```

### 6.3 Register Write

**For MT8183-compatible (8-bit capable):**
```
offset 0x00 (MSDC_CFG):
  Bits [17:16] = divider_mode (0-3)
  Bits [15:8] = hw_div (8-bit divider)
  Clear bits [17:8] first, then set
```

**For non-MT8183:**
```
offset 0x00:
  Bits [21:20] = divider_mode
  Bits [19:8] = hw_div (12-bit divider)
```

### 6.4 Clock Free-Run for Voltage Switch (line 3466–3476)

When switching to 1.8V (UHS), enables clock free-run for 1ms:
```
MSDC_CFG |= bit 1  // Enable clock free-run
usleep_range(1000, 1500)  // 1-1.5ms
MSDC_CFG &= ~bit 1  // Disable clock free-run
```

---

## 7. Card Detection: `msdc_get_cd` (line 407)

### 7.1 Methods

**Method 1 — GPIO (mmc_gpio_get_cd):**
When `bit 0 of offset 0x3d5` is 0 AND custom CD is not enabled:
```c
uVar1 = mmc_gpio_get_cd(param_1);  // Standard GPIO card detect
```

**Method 2 — Custom (MSDC register-based):**
When custom CD is enabled (`offset 0x946 == 1`):
```c
// XOR of two register fields, inverted
uVar1 = (*(uint *)(param_1 + 0x3d8) >> 10 ^
         *(uint *)(*(long *)(param_1 + 0x7c0) + 8) >> 1 ^ 0xffffffff) & 1;
```

This reads:
- MSDC register at base + 0x08 (SW_RST area) bit 1 — DAT[0] line state
- Stored state at offset 0x3d8 bit 10

**Method 3 — Force present:**
When `offset 0xc83` (force_detect) is set:
```c
uVar3 = 1;  // Always report as present
offset 0xc82 = 0;
```

### 7.2 Card Detect Change

After detecting, logs "inserted" or "removed" and calls `tran_get_sd_tray_insert_status()`.

### 7.3 Overcurrent Handler (line 1517)

```c
sdcard_oc_handler → msdc_sd_power_off
```
Disables regulators and triggers card removal.

---

## 8. Set IOS: `msdc_ops_set_ios` (line 2825)

Handles power mode, bus width, and clock changes.

### 8.1 Power Modes

| Mode | Code | Action |
|------|------|--------|
| OFF (0) | Line 2843 | Disable vmmc regulator, disable vqmmc |
| ON (1) | Line 2879 | `msdc_init_hw()` + `mmc_regulator_set_ocr()` + 200ms delay |
| UP (2) | Line 2864 | Enable vqmmc regulator |

### 8.2 Bus Width: `msdc_set_buswidth` (line 3302)

```
offset 0x30 (SDC_CFG):
  Bits [17:16] cleared
  width=1: no bits set (1-bit)
  width=2: bit 16 = 1 (4-bit)
  width=3: bit 17 = 1 (8-bit)
```

---

## 9. Command Processing

### 9.1 Command Build: `msdc_cmd_prepare_raw_cmd` (line 2219)

**SD command index mapping to response type:**

| CMD | Response Type | Flags |
|-----|---------------|-------|
| CMD0 | 0 (none) | — |
| CMD1 | 3 (R3) | — |
| CMD2 | 0 | — |
| CMD3 | 0 | — |
| CMD7 | 2 (R1b) | — |
| CMD8 | 1 (R1/R7) | — |
| CMD13 | — | 0x800 if R2 type |
| CMD18 | 1 (R1) | `|= 0x20000000` (auto stop if >64KB) |
| CMD25 | 1 (R1) | `|= 0x20000000` (auto stop if >64KB) |
| CMD55 | 1 (R1) | — |
| CMD56 | — | `|= 0x40000000` (read) |
| CMD52 | — | `|= 0x400` (SDIO) |

**CMD register format (offset 0x34):**
```
Bits [5:0]   = CMD index
Bits [9:7]   = Response type
Bit 30       = Wait CRC
Bit 28       = Auto stop
Bit 12       = Write data
Bit 11       = Read data
Bit 16       = Block count
```

**DAT register (offset 0x50):**
```
Bits [11:0]  = Block size
```

### 9.2 Command Send: `msdc_start_command` (line 2047)

1. Check CMD bus ready (offset 0x3C, bits [1:0])
2. Check controller busy (offset 0x3C, bit 0) for DAT commands
3. Clear FIFO if needed (offset 0x14)
4. Write ARG to offset 0x38
5. Write CMD to offset 0x34
6. Set timeout: 0x4E20ms (20000ms) delayed work

### 9.3 Polling Mode: `msdc_command_resp_polling` (line 2103)

Polls INT_STS (offset 0x0C) for bits [10:7] = command response flags:
```
0x200 = CMD_ERR
0x100 = CMD_TIMEOUT
0x80  = CRC_ERR (for CMD)
0x40  = DAT_TIMEOUT
0x08  = AUTO_CMD_DONE
```

---

## 10. Voltage Switch: `msdc_ops_switch_volt` (line 3033)

```c
if (vqmmc_regulator exists) {
    if (target_voltage > 3.3V) → ERROR
    mmc_regulator_set_vqmmc(param_1, param_2);
    if (target == 1.8V):
        pinctrl_select_state(pHandle, state_uhs);    // Switch to UHS pins
    else:
        pinctrl_select_state(pHandle, default_state); // Switch to default pins
}
```

---

## 11. Tuning: `msdc_execute_tuning` (line 3078)

### 11.1 Auto-K (Automatic Calibration)

For host-index 1 (MSDC1 SD card):
```c
if (host_index - 1U < 2) {  // host 1 or 2
    sd_execute_autok(lVar4, param_2);
}
```

### 11.2 Bad SD Detection

The driver tracks tuning failures per SD card:
- `bad_sd_detecter` — failure counter
- `bad_sd_tolerance` — threshold
- `bad_sd_timer` / `bad_sd_forget` — timing window

When tolerance exceeded → forces card removal via `mmc_detect_change()`.

---

## 12. Data Transfer

### 12.1 DMA Setup: `msdc_dma_setup` (line 2441)

- DMA mode 3 (ADMA2) — `uVar7 = 3`
- DMA descriptor at `plVar11[0xfb]` (0x38 bytes)
- DMA buffer at `plVar11[0xfc]` (0x4000 bytes)
- DMA descriptors built as linked list with checksum

### 12.2 Data Start: `msdc_start_data` (line 2411)

```
offset 0x98 (DMA_CFG):
  Bit 0 = 1 → Start DMA
offset 0x10 (INT_MASK):
  |= 0xED000 → Enable data transfer interrupts
```

### 12.3 Data Done: `msdc_data_xfer_done` (line 1921)

```
offset 0x98: Bit 1 = 1 → Stop DMA
Poll offset 0x9C until bit 0 clear
offset 0x10: Clear data-related interrupt bits
```

---

## 13. SDIO Support (host-index == 2)

### 13.1 SDIO EINT (lines 1137–1156)

```c
if (host-index == 2) {
    devm_gpiod_get_index(..., "sdio", 0, GPIOD_OUT_LOW);
    gpiod_to_irq(irq);
    irq_modify_status(irq, 0, IRQF_NO_SUSPEND);
    devm_request_threaded_irq(..., sdio_eint_irq_cfi_jt,
                              IRQF_TRIGGER_FALLING | IRQF_ONESHOT,
                              "sdio-eint", plVar2);
}
```

### 13.2 SDIO IRQ Enable (line 2926)

Enables/disables SDIO interrupt:
```
offset 0x10 (INT_MASK): Bit 7 = SDIO IRQ enable
offset 0x30 (SDC_CFG): Bit 20 = SDIO interrupt enable
```

---

## 14. Hardware Reset: `msdc_hw_reset` (line 1257)

```
offset 0x7C (EMMC50_CFG): Bit 0 = 1 → Assert HW reset
__const_udelay(0xA7C6)   // ~43.7us delay
offset 0x7C: Bit 0 = 0 → Deassert HW reset
```

---

## 15. Key Constants Summary

| Constant | Value | Location | Description |
|----------|-------|----------|-------------|
| Host struct size | 0x570 | Line 834 | `mmc_alloc_host` size |
| DMA buffer size | 0x4000 (16KB) | Line 1082 | Data transfer buffer |
| DMA descriptor size | 0x38 (56B) | Line 1080 | ADMA2 descriptor ring |
| CQE base offset | 0x800 | Line 1066 | CQE registers = MSDC base + 0x800 |
| CMD timeout | 0x4E20 (20000) | Line 2061 | Delayed work timeout (ms) |
| Power-on delay | 0xC49C08 | Line 2886 | `__const_udelay` (~200ms) |
| HW reset delay | 0xA7C6 | Line 3269 | `__const_udelay` (~44us) |
| 1.8V free-run | 1000-1500us | Line 3472 | Clock free-run before voltage switch |
| Autosuspend | 0x32 (50ms) | Line 1160 | PM runtime autosuspend delay |
| DMA base addr mask | 0x80000080000 | Line 1049 | DMA descriptor address mask |
| Bus width mask | 0x3 | Line 3311 | Clear bits [17:16] |
| DDR50 bit | 0x580 | Line 3363 | Mask for SDR/DDR timing modes |

---

## 16. MSDC1 Register Map (inferred from pseudocode)

| Offset | Name | Description |
|--------|------|-------------|
| 0x00 | MSDC_CFG | Clock config, divider, enable |
| 0x04 | ? | — |
| 0x08 | SW_RST | Software reset (bit 2) |
| 0x0C | INT_STS | Interrupt status (read to clear) |
| 0x10 | INT_MASK | Interrupt mask |
| 0x14 | ? | FIFO status, patch bits |
| 0x30 | SDC_CFG | SD config (bus width, CMD/DAT enable, busy timeout) |
| 0x34 | SDC_CMD | Command register |
| 0x38 | SDC_ARG | Command argument |
| 0x3C | SDC_STS | Status (CMD/DAT ready, controller busy) |
| 0x40 | SDC_RESP0 | Response word 0 |
| 0x44 | SDC_RESP1 | Response word 1 |
| 0x48 | SDC_RESP2 | Response word 2 |
| 0x4C | SDC_RESP3 | Response word 3 |
| 0x50 | SDC_BLOCK_NUM | Block size/count |
| 0x64 | PAD_TUNE1 / NEW_TX_RX | New tuning register |
| 0x68 | EMMC50_CFG | Enhanced strobe, HW reset |
| 0x7C | EMMC50_CFG (alt) | HW reset control |
| 0x80 | ? | — |
| 0x98 | DMA_CFG | DMA control |
| 0x9C | DMA_STATUS | DMA status |
| 0xB8 | PAD_TUNE0 | Pad tuning (CMD/DAT pull) |
| 0x800 | CQE_BASE | Command Queue Engine registers |
| 0x204 | ? | Enhanced strobe config |
| 0x208 | ? | Enhanced strobe enable |
| 0x20C | ? | Enhanced strobe config |
| 0x7FC | ? | Enhanced strobe tuning |

---

## 17. Host Structure Key Offsets

| Offset | Size | Description |
|--------|------|-------------|
| 0x780 | 8 | MMC host pointer |
| 0x7C0 | 8 | MSDC register base (virtual) |
| 0x7C8 | 8 | 2nd register base (top control) |
| 0x7F0-0x7F8 | 8 | DMA descriptor phys addr |
| 0x808 | 8 | Pinctrl handle |
| 0x810 | 8 | Default pinctrl state |
| 0x818 | 8 | UHS pinctrl state |
| 0x8AC | 4 | SDIO EINT IRQ number |
| 0x8C0 | 8 | source clock |
| 0x8C8 | 8 | hclk |
| 0x8D0 | 8 | bus_clk |
| 0x8D8 | 8 | crypto_clk |
| 0x8E0 | 8 | source_cg |
| 0x8E8 | 8 | sys_cg |
| 0x8F0 | 8 | crypto_cg |
| 0x8F8 | 24 | Bulk clocks (pclk/axi/ahb) |
| 0x92C | 4 | Source clock rate (kHz) |
| 0x930 | 1 | Current timing mode |
| 0x944 | 1 | HS400 CMD resp rising |
| 0x945 | 1 | HS400 tuning flag |
| 0x946 | 1 | Custom card detect enable |
| 0x947 | 1 | Supports CQE |
| 0xA58 | 4 | Host type (0=eMMC, 1=SD, 2=SDIO) |
| 0xA5C | 1 | Tuning in progress |
| 0xA60 | 4 | Tuning retry counter |
| 0xC82 | 1 | Last card detect state |
| 0xC83 | 1 | Force card detect |
| 0xC84 | 4 | Tuning failure count |
| 0xC88 | 4 | HW reset retry count |
| 0xC90 | 4 | DVFSRC voltage target |
| 0xC94 | 4 | OCR voltage from DT |
| 0xC98 | 8 | DVFSRC regulator |
| 0xCA0 | 1 | Polling mode flag |
| 0xCA8 | 24 | CPU latency QoS request |
