# DTBO / Panel Configuration — Infinix X676B (NOTE 12 PRO, MT6789)

## Дата составления
2026-07-13

## 1. PANEL NAME / MODEL

### 1.1. Основное название (из DTBO overlay)
```
compatible = "ili7835,fhdp,dsi,cmd,edo,edo,x676b";
```
- **Panel driver IC**: ILI7835 (или ILI7835F)
- **Kernel module**: `ili7835_fhdp_dsi_cmd_edo_edo_x676b.ko`
- **Тип**: Command mode (DSI CMD)
- **Производитель**: ILITEK (предположительно)

### 1.2. Альтернативное название (из LK)
```
atag,videolfb-lcmname = "nt36672e_fhdp_dsi_vdo_60hz_jdi_dphy_drv";
```
- **Panel driver IC**: Novatek NT36672E
- **Тип**: Video mode (DSI VDO)
- LK прошит с NT36672E, но реальная панель в DTBO — ILI7835 (kernel переопределяет)

### 1.3. Compatable strings summary
| Source | Compatible / Name | Mode |
|--------|------------------|------|
| LK DTS (chosen node) | `nt36672e_fhdp_dsi_vdo_60hz_jdi_dphy_drv` | Video |
| DTBO overlay (fragment@17) | `ili7835,fhdp,dsi,cmd,edo,edo,x676b` | Command |
| Kernel module alias | `of:N*T*Cili7835,fhdp,dsi,cmd,edo,edo,x676b` | Command |

---

## 2. RESOLUTION

| Parameter | Value | Source |
|-----------|-------|--------|
| **Width** | **1080 pixels** | Все источники |
| **Height** | **2400 pixels** | Все источники |
| **Format** | **a8r8g8b8** (32 bpp, BGRA8888) | U-Boot DTS, LK |
| **Physical size** | 74 x 131 mm | NT36672E driver (аналогично для ILI7835) |
| **PPI** | ~395 | Расчёт |

Подтверждено:
- `focaltech,display-coords = <0x00 0x00 0x437 0x95f>` — 0x437=1079, 0x95f=2399 (touch mapping)
- U-Boot DTS framebuffer: width=1080, height=2400
- LK decompile: 0x438 x 0x960

---

## 3. DSI CONFIGURATION

### 3.1. DSI Controller

| Parameter | Value |
|-----------|-------|
| **Base address** | **0x14013000** |
| **Compatible** | `mediatek,mt6789-dsi` |
| **DSI mode** | **Command mode** (CMD, не VIDEO) |
| **Number of lanes** | **4** (D-PHY) |
| **PHY type** | **D-PHY** (MIPI D-PHY) |
| **MIPI TX PHY base** | **0x11f60000** |
| **Compatible (MIPI TX)** | `mediatek,mt6789-mipi-tx` |
| **DSI clock source** | `mipi_tx0_pll` |
| **HS clock source** | `clk26m` -> PLL |
| **Engine clock** | CLK_MM_DISP_DSI0 (clk index: 35_10) |
| **Digital clock** | CLK_MM_DSI0_DIGITAL (clk index: 35_15) |
| **CMDQ offset** | **0xd00** (MT6789-specific, NOT 0x200) |
| **TE (Tearing Effect)** | Enabled (GPIO 0x53, pinmux mode 1) |

### 3.2. DSI Clocks (from main_tree.dts)
```dts
clocks = <0x35 0x10>,    /* engine  = CLK_MM_DISP_DSI0 */
         <0x35 0x15>,    /* digital = CLK_MM_DSI0_DIGITAL */
         <0xa1>;         /* hs      = mipi_tx0_pll */
clock-names = "engine", "digital", "hs";
```

### 3.3. Display OPP Table (DVFS)
| OPP | Frequency | Voltage |
|-----|-----------|---------|
| 0 | 200 MHz | 0.86470V |
| 1 | 312 MHz | 0.927c0V |
| 2 | 416 MHz | 0x9eb10V |
| 3 | **520 MHz** | 0xb1008V |

---

## 4. DSI TIMING PARAMETERS (из NT36672E kernel driver)

Для ILI7835 command-mode панели точные тайминги в DTS НЕ УКАЗАНЫ
(они зашиты в kernel модуле `ili7835_fhdp_dsi_cmd_edo_edo_x676b.ko`, исходников которого нет).

Для справки — тайминги NT36672E (1080x2408, video mode, 60Hz) из panel-novatek-nt36672e.c:

```c
static const struct drm_display_mode nt36672e_1080x2408_60hz = {
    .name = "1080x2408",
    .clock = 181690,           // kHz
    .hdisplay = 1080,
    .hsync_start = 1080 + 76,  // HBP = 76
    .hsync_end   = 1080 + 76 + 12,       // HSA = 12
    .htotal      = 1080 + 76 + 12 + 56,   // HFP = 56
    .vdisplay = 2408,
    .vsync_start = 2408 + 46,  // VBP = 46
    .vsync_end   = 2408 + 46 + 10,       // VSA = 10
    .vtotal      = 2408 + 46 + 10 + 10,   // VFP = 10
    .flags = 0,
};
```

### Расшифровка NT36672E таймингов
| Parameter | Value |
|-----------|-------|
| Pixel clock | 181.69 MHz |
| Hactive | 1080 |
| HBP (hback porch) | 76 |
| HSA (hsync width) | 12 |
| HFP (hfront porch) | 56 |
| Htotal | 1224 |
| Vactive | 2408 |
| VBP | 46 |
| VSA | 10 |
| VFP | 10 |
| Vtotal | 2474 |

**Важно**: ILI7835 (command mode panel) может иметь ДРУГИЕ тайминги.
Точные значения нужно извлекать из:
- Decompiled LK (`lk_unpack_v0/lk_a_lk.bin.c`) — функции LCM driver ILI7835
- Kernel module `ili7835_fhdp_dsi_cmd_edo_edo_x676b.ko` (available in `vendor_boot/root.1/lib/modules/`)

---

## 5. PANEL GPIOs (из DTBO overlay)

| Signal | GPIO | Active |
|--------|------|--------|
| **Reset** | GPIO 0x55 (85) | Low |
| **Bias** | GPIO 0x40 (64) | High/Low (VCI1/VCI0) |
| **TE (Tearing Effect)** | GPIO 0x53 (83), pinmux 0x5301 | EINT mode |

### Panel power sequencing pins
- `lcd_bias_vci1_gpio` (GPIO 64, output-high)
- `lcd_bias_vci0_gpio` (GPIO 64, output-low)
- `lcm_rst_out1_gpio` (reset high)
- `lcm_rst_out0_gpio` (reset low)

---

## 6. BACKLIGHT CONFIGURATION

```dts
led@6 {
    compatible = "mediatek,lcd-backlight";
    led_mode = <0x04>;        // Mode 4 = PWM through PMIC
    data = <0x01>;
    pwm_config = <0x00 0x01 0x00 0x00 0x00>;
};

backlight {                   // alternative (fragment@13)
    led_mode = <0x04>;
    pwm_config = <0x00 0x01 0x00 0x00 0x00>;
    max-hw-brightness = <0x3ff>;  // 10-bit backlight
};
```

---

## 7. DISPLAY PIPELINE (MMSYS @ 0x14000000)

### Routing (из kernel DTS + LK)
```
OVL0 (0x14005000) -> RDMA0 (0x14007000) -> ... -> DSI0 (0x14013000)
```

| Block | Address | Clock gate bit |
|-------|---------|---------------|
| MMSYS config | 0x14000000 | 35_16 (DSI0 pixel) |
| disp_mutex0 | 0x14001000 | 35_00 (SMI/LARB) |
| disp_ovl0 | 0x14005000 | 35_02 |
| disp_ovl0_2l | 0x14006000 | 35_04 |
| disp_rdma0 | 0x14007000 | 35_03 |
| disp_color0 | 0x14009000 | 35_09 |
| disp_ccorr0 | 0x1400b000 | 35_08 |
| disp_aal0 | 0x1400c000 | 35_07 |
| disp_gamma0 | 0x1400d000 | 35_0c |
| disp_postmask0 | 0x1400e000 | 35_0d |
| disp_dither0 | 0x1400f000 | 35_0e |
| dsi0 | 0x14013000 | 35_10 (engine), 35_15 (digital) |

---

## 8. FRAMEBUFFER (из U-Boot DTS и LK DTS)

| Parameter | LK DTS | U-Boot DTS |
|-----------|--------|------------|
| **Physical address** | 0xfe0c8000 | 0xfe0c8000 |
| **VRAM size** | 0x1be0000 (~28 MB) | 0x1f2f000 (~31 MB) |
| **Width** | 1080 | 1080 |
| **Height** | 2400 | 2400 |
| **Stride** | (1080+16)*4 = 4384 | 4384 |
| **BPP** | 32 (a8r8g8b8) | 32 (BGRA8888) |
| **FPS** | 60 (0x1770) | 60 |

---

## 9. FILES ANALYZED

| File | Description |
|------|-------------|
| `/workdir/git/heliog99/dtbo/infinix-x676b.dts` | Hand-written DTS (basic, no display node yet) |
| `/workdir/git/heliog99/dtbo/infinix-x676b-overlay.dts` | Hand-written DTBO overlay (no panel fragment yet) |
| `/workdir/git/heliog99/dtbo/mt6789.dtsi` | SoC-level DTSI (no dsi0 node yet) |
| `/workdir/git/heliog99/dtbo/README.md` | DTBO project documentation |
| `/workdir/git/heliog99/firmware/.../extracted_dtb/overlays.dts` | **Stock DTBO** (panel1@0 with ili7835) |
| `/workdir/git/heliog99/firmware/.../extracted_dtb/main_tree.dts` | **Stock kernel DTB** (dsi0, display pipeline, chosen) |
| `/workdir/git/heliog99/unpack/dtbo/dt/dt.0.dts` | Unpacked device DTBO (same as overlays.dts) |
| `/workdir/git/heliog99/docs/AGENTS/lk_dts_panel_params.md` | Previous panel params analysis |
| `/workdir/git/heliog99/src_linux/mtk_g99-mainline/drivers/gpu/drm/panel/panel-novatek-nt36672e.c` | NT36672E kernel panel driver (timings reference) |
| `/workdir/git/heliog99/docs/MT6789_REGISTERS.md` | Register map (section 94-101: ILI7835 panel info) |

---

## 10. SUMMARY

### Panel: ILI7835F (aka ILITEK ILI7835)

| Parameter | Value |
|-----------|-------|
| **Panel name** | `ili7835_fhdp_dsi_cmd_edo_edo_x676b` |
| **Compatible** | `"ili7835,fhdp,dsi,cmd,edo,edo,x676b"` |
| **Resolution** | **1080 x 2400** (FHDP) |
| **DSI interface** | **4 lanes, D-PHY, Command mode** |
| **DSI base** | **0x14013000** |
| **CMDQ FIFO offset** | **0xd00** |
| **TE pin** | GPIO 83 (0x53), EINT mode |
| **Reset pin** | GPIO 85 (0x55), active low |
| **Bias pin** | GPIO 64 (0x40) |
| **Backlight** | PWM mode 4, 10-bit (max 0x3ff) |
| **Format** | RGB888 (24-bit) |
| **Framebuffer** | 32bpp a8r8g8b8 @ 0xfe0c8000 |
| **DSI clock** | MIPI D-PHY PLL from 26 MHz ref |
| **Refresh rate** | 60 Hz |

### Key finding: LK vs Kernel panel mismatch
- **LK** uses NT36672E (Novatek, video mode) — `atag,videolfb-lcmname`
- **Kernel DTBO** uses ILI7835 (ILITEK, command mode) — `compatible = "ili7835,..."`
- LK does NOT initialize the real panel (islcm_inited=0)
- The real panel initialization happens in kernel via the DTBO panel node + kernel module

### Missing (need extraction)
- Exact DSI timing parameters for ILI7835 command mode panel
- ILI7835 panel init sequence (DCS commands)
- These are in the kernel module `ili7835_fhdp_dsi_cmd_edo_edo_x676b.ko` (no source available)
- LK decompile (`lk_unpack_v0/lk_a_lk.bin.c`) has ILI7835 functions at string refs
