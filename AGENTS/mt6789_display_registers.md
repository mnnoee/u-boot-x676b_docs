# MT6789 Display Registers — Verified from Kernel Source

## DISP Power Domain (SPM)

**Source:** `src_linux/linux/drivers/pmdomain/mediatek/mt6789-pm-domains.h:139-152`

```c
[MT6789_POWER_DOMAIN_DISP] = {
    .name = "disp",
    .sta_mask = BIT(21),
    .ctl_offs = 0x354,
    .pwr_sta_offs = 0x016c,
    .pwr_sta2nd_offs = 0x0170,
    .sram_pdn_bits = GENMASK(8, 8),
    .sram_pdn_ack_bits = GENMASK(12, 12),
    .bp_cfg = {
        BUS_PROT_WR_IGN(INFRA, MT6789_TOP_AXI_PROT_EN_MM_DISP, 0x02D4, 0x02D8, 0x02EC),
        BUS_PROT_WR_IGN(INFRA, MT6789_TOP_AXI_PROT_EN_DISP, 0x02A0, 0x02A4, 0x0228),
    },
},
```

| Register | Offset | Address |
|----------|--------|---------|
| DISP_PWR_CON | 0x354 | 0x10006354 |
| PWR_STATUS | 0x016c | 0x1000616c |
| PWR_STATUS_2ND | 0x0170 | 0x10006170 |

**Bit layout of DISP_PWR_CON (0x10006354):**
- Bit 0: PWR_RST_B (reset, active low)
- Bit 1: PWR_ISO (isolation)
- Bit 2: PWR_ON (power on)
- Bit 3: PWR_ON_2ND (secondary power on)
- Bit 4: PWR_CLK_DIS (clock disable)
- Bit 8: SRAM_PDN (SRAM power down)
- Bit 12: SRAM_PDN_ACK

**Power on sequence** (`mtk-pm-domains.c:256-267`):
```c
regmap_set_bits(base, 0x354, BIT(2));       // PWR_ON
regmap_set_bits(base, 0x354, BIT(3));       // PWR_ON_2ND
// poll until (read(0x16c) & BIT(21)) && (read(0x170) & BIT(21))
regmap_clear_bits(base, 0x354, BIT(4));     // clear PWR_CLK_DIS
regmap_clear_bits(base, 0x354, BIT(1));     // clear PWR_ISO
regmap_set_bits(base, 0x354, BIT(0));       // set PWR_RST_B
```

**Status check:** Bit 21 in BOTH 0x1000616c AND 0x10006170 must be set.

**MT6789 is NOT in mtk-scpsys.c** — it uses the newer `mtk-pm-domains.c` driver.

---

## MUTEX_EN Register

**Source:** `src_linux/linux/drivers/soc/mediatek/mtk-mutex.c:26,1023-1032`

```c
#define DISP_REG_MUTEX_EN(n)  (0x20 + 0x20 * (n))
```

For mutex 0: `0x14001000 + 0x20 = 0x14001020`

**Written via DIRECT writel(), NOT CMDQ:**
```c
void mtk_mutex_enable(struct mtk_mutex *mutex) {
    writel(1, mtx->regs + DISP_REG_MUTEX_EN(mutex->id));
}
```

Called from `mtk_crtc.c:397` with `cmdq_pkt = NULL`.

**MOD bits** (`mtk-mutex.c:975-977`):
```c
reg = readl_relaxed(mtx->regs + offset);
reg |= BIT(mod_id);
writel_relaxed(reg, mtx->regs + offset);
```

**SOF** (`mtk-mutex.c:981-983`):
```c
writel_relaxed(sof, mtx->regs + DISP_REG_MUTEX_SOF(mutex->id));
```

---

## OVL_EN Register

**Source:** `src_linux/linux/drivers/gpu/drm/mediatek/mtk_disp_ovl.c:26,253-265`

```c
#define DISP_REG_OVL_EN  0x000c   // NOT 0x000!
```

For OVL0: `0x14005000 + 0x000c = 0x1400500c`

**Written via DIRECT writel_relaxed(), NOT CMDQ:**
```c
void mtk_ovl_start(struct device *dev) {
    writel_relaxed(0x1, ovl->regs + DISP_REG_OVL_EN);  // offset 0x000c
}
```

**IMPORTANT:** We were writing to 0x14005000 (base), but OVL_EN is at 0x1400500c!

---

## MMSYS Routing

**Source:** `src_linux/linux/drivers/soc/mediatek/mtk-mmsys.c:172-209`

All MMSYS routing done via **direct CPU writes** (cmdq_pkt=NULL):
```c
mtk_mmsys_update_bits(mmsys, routes[i].addr, routes[i].mask, routes[i].val, NULL);
```

**MT6789 routing table** (`mt6789-mmsys.h:45-69`):
- 0xf14: OVL0_2L output mux
- 0xf18: OVL0 output mux
- 0xf20: DITHER0 output mux
- 0xf28: RDMA0 input select
- 0xf30: DSI0 input select
- 0xf0c: RDMA0 RSZ0 output select
- 0xf38: DSC0 output mux

---

## Clock Gates

**Source:** `src_linux/linux/drivers/clk/mediatek/clk-mt6789-mm.c`

| Bit | Clock | Set addr | Clear addr | Status addr |
|-----|-------|----------|------------|-------------|
| 0 | CLK_MM_DISP_MUTEX0 | 0x14000104 | 0x14000108 | 0x14000100 |
| 1 | CLK_MM_APB_BUS | same | same | same |
| 2 | CLK_MM_DISP_OVL0 | same | same | same |
| 3 | CLK_MM_DISP_RDMA0 | same | same | same |

---

## MT6789 MUTEX Module IDs

**Source:** `src_linux/linux/drivers/soc/mediatek/mtk-mutex.c:52-65`

```c
#define MT6789_MUTEX_MOD_DISP_OVL0          0
#define MT6789_MUTEX_MOD_DISP_OVL0_2L       1
#define MT6789_MUTEX_MOD_DISP_RDMA0         2
#define MT6789_MUTEX_MOD_DISP_RSZ0          3
#define MT6789_MUTEX_MOD_DISP_COLOR0        4
#define MT6789_MUTEX_MOD_DISP_CCORR0        5
#define MT6789_MUTEX_MOD_DISP_AAL0          7
#define MT6789_MUTEX_MOD_DISP_GAMMA0        8
#define MT6789_MUTEX_MOD_DISP_POSTMASK0     9
#define MT6789_MUTEX_MOD_DISP_DITHER0       10
#define MT6789_MUTEX_MOD_DISP_DSC_WRAP0_CORE0 13
#define MT6789_MUTEX_MOD_DISP_DSI0          14
#define MT6789_MUTEX_MOD_DISP_WDMA0         15
#define MT6789_MUTEX_MOD_DISP_PWM0          16
```

---

## Key Conclusion

**All display pipeline registers (MUTEX_EN, OVL_EN, MMSYS routing) are written via direct CPU writel() in the kernel driver.** CMDQ path exists as an alternative API but is NOT used by the standard DRM CRTC code path. Direct CPU writes are architecturally valid for U-Boot.

**OVL_EN is at offset 0x000c, NOT 0x000** — this was our mistake.
