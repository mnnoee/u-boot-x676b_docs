# OVL Layer Register Layout — MT6789 (from Kernel Source)

## Source Files
- `src_linux/linux/drivers/gpu/drm/mediatek/mtk_disp_ovl.c`
- `src_linux/mtk_g99-mainline/drivers/gpu/drm/mediatek/mtk_disp_ovl.c`

## MT6789 OVL Base Addresses

| Block | Base Address | Source |
|-------|-------------|--------|
| **ovl0** (4-layer) | `0x14005000` | `linux/arch/arm64/boot/dts/mediatek/mt6789.dtsi:1518` |
| **ovl_2l0** (2-layer) | `0x14006000` | `linux/arch/arm64/boot/dts/mediatek/mt6789.dtsi:1546` |

## Driver Data (mt6789 uses mt8192 fallback)

```c
// mtk_disp_ovl.c:701-712
static const struct mtk_disp_ovl_data mt8192_ovl_driver_data = {
    .addr = DISP_REG_OVL_ADDR_MT8173,   // = 0x0f40
    .gmc_bits = 10,
    .layer_nr = 4,
    .fmt_rgb565_is_0 = true,
    .smi_id_en = true,
    ...
};
```

## Per-Layer Register Offsets

Each layer `n` (0-3) occupies a **0x20 (32-byte) block**:

| Register | Offset | Layer 0 | Layer 1 | Layer 2 | Layer 3 |
|----------|--------|---------|---------|---------|---------|
| **LAYER_ADDR** | `0x0f40 + 0x20*n` | **0x14005f40** | **0x14005f60** | **0x14005f80** | **0x14005fa0** |
| LAYER_HDR_ADDR | `0x0f44 + 0x20*n` | 0x14005f44 | 0x14005f64 | 0x14005f84 | 0x14005fa4 |
| LAYER_HDR_PITCH | `0x0f48 + 0x20*n` | 0x14005f48 | 0x14005f68 | 0x14005f88 | 0x14005fa8 |

## Other Key Registers (OVL0 base 0x14005000)

| Offset | Register | Purpose |
|--------|----------|---------|
| 0x000c | OVL_EN | OVL enable (bit 0) |
| 0x0014 | OVL_RST | Software reset |
| 0x0020 | OVL_ROI_SIZE | ROI size (height<<16 \| width) |
| 0x0024 | OVL_DATAPATH_CON | Datapath control |
| 0x0028 | OVL_ROI_BGCLR | Background color |
| 0x002c | OVL_SRC_CON | Source enable (bit per layer) |
| 0x0030+0x20*n | OVL_CON(n) | Layer n control |
| 0x0038+0x20*n | OVL_SRC_SIZE(n) | Layer n source size |
| 0x003c+0x20*n | OVL_OFFSET(n) | Layer n display offset |
| 0x0040+0x20*n | OVL_PITCH_MSB(n) | Pitch MSB |
| 0x0044+0x20*n | OVL_PITCH(n) | Pitch LSB |
| 0x00c0+0x20*n | OVL_RDMA_CTRL(n) | RDMA control |
| 0x00c8+0x20*n | OVL_RDMA_GMC(n) | GMC threshold |

## Confirmed by LK

LK writes buffer address at offset `0xf40` (line 46988):
```c
*(uint *)(lVar1 + 0xf40) = param_13 + param_16 * param_15 + iVar7 * param_14;
```
Where `lVar1 = base + (iVar8 + param_10 * 0x20)` — confirms 0x20 layer spacing.

## Confirmed Layer 1 Layout (from live device 2026-08-05)

Fresh dump `md.l 0x14005050 8` and `md.l 0x14005f60 1`:

| Offset | Address | Value | Register |
|--------|---------|-------|----------|
| +0x050 | 0x14005050 | 0x000021ff | **L1_CON**: alpha=0xff, AEN=1, format=2<<12 (ARGB8888) |
| +0x054 | 0x14005054 | 0x00000000 | (reserved/unused) |
| +0x058 | 0x14005058 | 0x09600438 | **L1_SRC_SIZE**: 2400×1080 (height<<16 \| width) |
| +0x05c | 0x1400505c | 0x00000000 | **L1_OFFSET**: x=0, y=0 |
| +0x060 | 0x14005060 | 0x00100000 | **L1_PITCH_MSB** (or DMA_CTRL) |
| +0x064 | 0x14005064 | 0x000010e0 | **L1_PITCH**: 4320 = 1080×4 bytes |
| +0x0f60 | 0x14005f60 | 0xfeabe000 | **L1_ADDR**: same FB as Layer 0 |

**Key finding**: Layer 1 is enabled by LK, points to the same FB (`0xfeabe000`), and is the active
layer (SRC_CON bit 1 = enabled). The driver configures Layer 0 but it's disabled (SRC_CON bit 0 = 0).

## How to Read from Device

```
# L0-L3 address registers:
=> md.l 0x14005f40 4
# L1 config (CON, SRC_SIZE, PITCH):
=> md.l 0x14005050 8
# SRC_CON (which layers are enabled):
=> md.l 0x1400502c 1
```
