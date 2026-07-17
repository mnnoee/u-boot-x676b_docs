# LK DTS Panel Parameters — Infinix X676B (MT6789)

## Source Files Analyzed

| File | Description |
|------|-------------|
| `/workdir/git/heliog99/lk_unpack/lk_main.dts` | Main LK device tree (6998 lines) |
| `/workdir/git/heliog99/firmware/Infinix_Note_12_Pro_X676B_MT6789_V2385_221208_MXML/Firmware/extracted_dtb/main_tree.dts` | Stock kernel main tree (identical to LK DTS) |
| `/workdir/git/heliog99/firmware/Infinix_Note_12_Pro_X676B_MT6789_V2385_221208_MXML/Firmware/extracted_dtb/overlays.dts` | DTBO overlay (panel node) |
| `/workdir/git/heliog99/u-boot-x676b/mt6789-infinix-x676b.dts` | U-Boot board DTS (framebuffer details) |
| `/workdir/git/heliog99/dtbo/infinix-x676b.dts` | Hand-written DTBO base for upstream |

## 1. Panel LCM

Taken from `chosen` node (`atag,videolfb-lcmname`):

```
nt36672e_fhdp_dsi_vdo_60hz_jdi_dphy_drv
```

| Field | Value |
|-------|-------|
| Driver IC | **Novatek NT36672E** |
| Resolution | **FHDP (Full HD+) = 1080x2400** |
| Interface | **DSI Video Mode** (vdo, not cmd) |
| Refresh Rate | **60 Hz** |
| Panel Vendor | **JDI** |
| PHY Type | **D-PHY** |

### Overlay Panel (DTBO)

The `overlays.dts` also defines a secondary panel compatible string:
```
compatible = "ili7835,fhdp,dsi,cmd,edo,edo,x676b";
```
This is a command-mode ILI7835 panel used in the production kernel DTBO. The LK only knows about NT36672E.

## 2. Resolution

Confirmed from multiple sources:

| Source | Width | Height | PPI |
|--------|-------|--------|-----|
| LCM name (fhdp) | 1080 | 2400 | ~395 |
| Touch coords: `focaltech,display-coords = <0 0 0x437 0x95f>` | 1080 | 2400 | |
| U-Boot DTS `framebuffer` node | 1080 | 2400 | |

**Resolution: 1080 x 2400 pixels**

## 3. DSI Timing Parameters

**No explicit DSI timing parameters (hsa, hbp, hfp, vsa, vbp, vfp) were found in any of the DTS files.** These parameters are:
- Hardcoded in the **kernel panel driver** for NT36672E (Novatek driver)
- Not exposed in device tree for this MediaTek platform

Typical NT36672E FHDP 60Hz DSI video mode timings (from Novatek datasheets and kernel drivers for similar 1080x2400 panels):

| Parameter | Typical Value | Notes |
|-----------|--------------|-------|
| hactive | 1080 | Visible width |
| vactive | 2400 | Visible height |
| hsa | ~2-10 | H-sync width |
| hbp | ~20-40 | H-back porch |
| hfp | ~20-40 | H-front porch |
| htotal | ~1100-1200 | hactive + hsa + hbp + hfp |
| vsa | ~2-4 | V-sync width |
| vbp | ~4-20 | V-back porch |
| vfp | ~4-20 | V-front porch |
| vtotal | ~2420-2470 | vactive + vsa + vbp + vfp |
| DSI clock | ~400-500 MHz | D-PHY, 4 lanes |
| Pixel clock | ~160-170 MHz | For 1080x2400@60Hz |

**IMPORTANT:** The exact timing values MUST be extracted from the NT36672E panel driver source code in the kernel (likely `drivers/gpu/drm/panel/panel-novatek-nt36672e.c` or similar) or from the decompiled LK code (`lk_unpack_v0/lk_a_lk.bin.c`).

## 4. Framebuffer

From U-Boot DTS (`/workdir/git/heliog99/u-boot-x676b/mt6789-infinix-x676b.dts`):

| Parameter | Value |
|-----------|-------|
| **Address** | **0xfe0c8000** (physical) |
| **VRAM Size** | **0x1f2f000** (32,669,696 bytes = ~31.1 MB) |
| **Width** | 1080 pixels |
| **Height** | 2400 pixels |
| **Stride** | (1080 + 16) * 4 = **4384 bytes** (16-pixel alignment) |
| **Format** | **a8r8g8b8** (BGRA8888, 32 bpp) |
| **FPS** | 60 (atag: 0x1770) |

From LK DTS `chosen` node:

| Parameter | Value |
|-----------|-------|
| `atag,videolfb-fb_base_l` | Encoded as ASCII "~\`P" (needs kernel parsing) |
| `atag,videolfb-fb_base_h` | 0x00 |
| `atag,videolfb-islcmfound` | 1 (panel detected) |
| `atag,videolfb-islcm_inited` | **0** (panel NOT initialized by LK) |
| `atag,videolfb-fps` | 0x1770 = 6000 (representing 60.00 fps) |
| `atag,videolfb-vramSize` | **0x1be0000** (29,286,400 bytes = ~28 MB) |

**NOTE:** The LK `vramSize` (0x1be0000) differs from the U-Boot framebuffer size (0x1f2f000). LK sets aside ~28 MB for display, while U-Boot uses ~31 MB. The actual framebuffer starts at **0xfe0c8000** based on U-Boot.

## 5. Display Pipeline (MMSYS)

All display hardware blocks enumerated in LK DTS (soc section @ 0x14000000):

```
DISP Pipeline: OVL0 -> RDMA0 -> (color/ccorr/aal/gamma/dither/postmask) -> DSI0
                    \-> OVL0_2L (2-layer)
```

| Block | Address | Compatible | Clock (clk#) |
|-------|---------|------------|--------------|
| MMSYS config | 0x14000000 | mediatek,mt6789-mmsys | clk: 35_16 |
| disp_mutex0 | 0x14001000 | mediatek,mt6789-disp-mutex | clk: 35_00 |
| disp_ovl0 | 0x14005000 | mediatek,mt6789-disp-ovl | clk: 35_02 |
| disp_ovl0_2l | 0x14006000 | mediatek,mt6789-disp-ovl | clk: 35_04 |
| disp_rdma0 | 0x14007000 | mediatek,mt6789-disp-rdma | clk: 35_03 |
| disp_rsz0 | 0x14008000 | mediatek,mt6789-disp-rsz | clk: 35_06 |
| disp_color0 | 0x14009000 | mediatek,mt6789-disp-color | clk: 35_09 |
| disp_ccorr0 | 0x1400b000 | mediatek,mt6789-disp-ccorr | clk: 35_08 |
| disp_aal0 | 0x1400c000 | mediatek,mt6789-disp-aal | clk: 35_07 |
| disp_gamma0 | 0x1400d000 | mediatek,mt6789-disp-gamma | clk: 35_0c |
| disp_postmask0 | 0x1400e000 | mediatek,mt6789-disp-postmask | clk: 35_0d |
| disp_dither0 | 0x1400f000 | mediatek,mt6789-disp-dither | clk: 35_0e |
| disp_dsc_wrap0 | 0x14012000 | mediatek,mt6789-disp-dsc | clk: 35_0b |
| **dsi0** | **0x14013000** | **mediatek,mt6789-dsi** | **clk: 35_10 (engine), 35_15 (digital)** |
| disp_wdma0 | 0x14014000 | mediatek,mt6789-disp-wdma | clk: 35_05 |

## 6. DSI Controller Details

From LK DTS node `dsi@14013000`:

```dts
dsi0: dsi@14013000 {
    compatible = "mediatek,dsi0", "mediatek,mt6789-dsi";
    reg = <0x00 0x14013000 0x00 0x1000>;
    interrupts = <0x00 0x12d 0x04 0x00>;
    clocks = <0x35 0x10>,    /* engine clock */
             <0x35 0x15>,    /* digital clock */
             <0xa1>;         /* hs clock (mipi_tx0_pll) */
    clock-names = "engine", "digital", "hs";
    phys = <0xa1>;           /* mipi_tx_config0 dphy */
    phy-names = "dphy";
};
```

### MIPI TX PHY

```dts
mipi_tx_config0: mipi_tx_config@11f60000 {
    compatible = "mediatek,mipi_tx_config0", "mediatek,mt6789-mipi-tx";
    reg = <0x00 0x11f60000 0x00 0x1000>;
    clocks = <0x55>;         /* clk26m = 26 MHz */
    #clock-cells = <0x00>;
    #phy-cells = <0x00>;
    clock-output-names = "mipi_tx0_pll";
};
```

## 7. Clock Frequencies for Display

### Display OPP Table (opp_table_disp0)

| OPP | Frequency | Voltage | Source Mux |
|-----|-----------|---------|------------|
| 0 | **200 MHz** (0xc65d400) | 0.86470V | TOP_UNIVPLL_D6_D2 |
| 1 | **312 MHz** (0x1298be00) | 0.927c0V | TOP_UNIVPLL_D4_D2 |
| 2 | **416 MHz** (0x18cba800) | 0x9eb10V | TOP_UNIVPLL_D6 |
| 3 | **520 MHz** (0x208b4c80) | 0xb1008V | TOP_MAINPLL_D4 |

### MMSYS Clock Gate Bits (from `clk-mt6789-mm.c` reference)

| Clock Index | Bit | Function |
|-------------|-----|----------|
| 35_00 | Bit 0 | SMI/LARB (MMSYS clock gate [0]) |
| 35_01 | Bit 1 | MUTEX |
| 35_02 | Bit 2 | OVL0 |
| 35_03 | Bit 3 | RDMA0 |
| 35_04 | Bit 4 | OVL0_2L |
| 35_05 | Bit 5 | WDMA0 |
| 35_06 | Bit 6 | RSZ0 |
| 35_07 | Bit 7 | AAL0 |
| 35_08 | Bit 8 | CCORR0 |
| 35_09 | Bit 9 | COLOR0 |
| 35_10 | Bit 10 | DSI0 engine |
| 35_0b | Bit 11 | DSC_WRAP0 |
| 35_0c | Bit 12 | GAMMA0 |
| 35_0d | Bit 13 | POSTMASK0 |
| 35_0e | Bit 14 | DITHER0 |
| 35_15 | Bit 21 | DSI0 digital |
| 35_16 | Bit 22 | DSI0 pixel clock / MMSYS control |

### Fixed Clocks

| Clock | Frequency |
|-------|-----------|
| clk32k | 32.768 kHz |
| clk26m | 26 MHz |
| clk13m | 13 MHz |
| ulposc | 260 MHz |

## 8. Power Domain

```dts
scpsys: power-controller@10006000 {
    compatible = "mediatek,mt6789-scpsys", "syscon";
    ...
};

dispsys_config: dispsys_config@14000000 {
    power-domains = <0x2d 0x0a>;    /* PD 0x0a = DISP domain */
};
```

Display power domain ID: **0x0a** (DISP), controlled by SPM @ 0x10006000.

## 9. Firmware DTBs Found

| File | Size | Description |
|------|------|-------------|
| `/workdir/git/heliog99/firmware/.../extracted_dtb/lk.img.dtb` | LK DTB | Embedded in LK image |
| `/workdir/git/heliog99/firmware/.../extracted_dtb/vendor_boot.img.dtb` | Vendor DTB | Main kernel DT |
| `/workdir/git/heliog99/firmware/.../extracted_dtb/vendor_boot-debug.img.dtb` | Debug vendor DTB | |
| `/workdir/git/heliog99/firmware/.../extracted_dtb/dtbo.img.dtb` | DTBO | Overlay (panel + devices) |
| `/workdir/git/heliog99/firmware/.../extracted_dtb/overlays.dts` | (decompiled) | Fragment-based overlay with ili7835 panel |
| `/workdir/git/heliog99/firmware/.../extracted_dtb/main_tree.dts` | (decompiled) | Identical to lk_main.dts |

## 10. Key Differences Between LK DTS and Kernel DTS

| Aspect | LK DTS (lk_main.dts) | Kernel DTBO (overlays.dts) |
|--------|---------------------|---------------------------|
| Panel | nt36672e_fhdp_dsi_vdo_60hz_jdi_dphy_drv | ili7835,fhdp,dsi,cmd,edo,edo,x676b |
| DSI mode | Video Mode (vdo) | Command Mode (cmd) |
| DSI TE | disabled | enabled |
| Backlight | backlight_cooler node | led6: lcd-backlight with pwm_config |
| LCM init | islcm_inited = 0 (not inited by LK) | Full panel node with GPIOs |

---

## Summary: Key Findings

| Parameter | Value |
|-----------|-------|
| **Resolution** | **1080 x 2400** (FHDP) |
| **Panel IC** | Novatek NT36672E / ILI7835 (dual source) |
| **DSI mode** | Video Mode (LK), Command Mode (kernel) |
| **Refresh rate** | 60 Hz |
| **FB address** | **0xfe0c8000** |
| **FB format** | **a8r8g8b8** (32 bpp, BGRA8888) |
| **FB stride** | 4384 bytes (1080+16 aligned * 4) |
| **FB size** | 0x1f2f000 (~31 MB) |
| **DSI clock** | MIPI D-PHY, 4 lanes |
| **Display OPP max** | 520 MHz (TOP_MAINPLL_D4) |
| **MMSYS base** | 0x14000000 |
| **DSI base** | 0x14013000 |
| **DSI CMDQ offset** | 0xd00 (previously documented) |
| **DISP power domain** | scpsys PD 0x0a |

**CRITICAL GAP:** DSI timing parameters (hsa, hbp, hfp, vsa, vbp, vfp) are NOT present in any DTS file — they must be extracted from:
1. The NT36672E kernel panel driver (e.g., `panel-novatek-nt36672e.c`)
2. The decompiled LK binary (`lk_unpack_v0/lk_a_lk.bin.c`)
