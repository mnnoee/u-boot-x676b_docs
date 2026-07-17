# MUTEX and OVL Register Write Methods — Kernel Source Analysis

## Source
- `src_linux/linux/drivers/soc/mediatek/mtk-mutex.c`
- `src_linux/linux/drivers/gpu/drm/mediatek/mtk_disp_ovl.c`
- `src_linux/linux/drivers/soc/mediatek/mtk-mmsys.c`

## Conclusion

**ALL display pipeline setup registers are written via DIRECT CPU writel(), NOT through CMDQ/GCE.**

The CMDQ path exists as an alternative API but is NOT used by the standard DRM CRTC code path.

---

## MUTEX_EN

**Register:** `DISP_REG_MUTEX_EN(n) = 0x20 + 0x20 * n`
For mutex 0: `0x14001000 + 0x20 = 0x14001020`

**Direct writel (CPU path)** — `mtk-mutex.c:1023-1032`:
```c
void mtk_mutex_enable(struct mtk_mutex *mutex) {
    writel(1, mtx->regs + DISP_REG_MUTEX_EN(mutex->id));
}
```

**Called from `mtk_crtc.c:397`** with `cmdq_pkt = NULL`.

**CMDQ alternative** — `mtk-mutex.c:1034-1051`:
```c
int mtk_mutex_enable_by_cmdq(struct mtk_mutex *mutex, void *pkt) {
    cmdq_pkt_write(cmdq_pkt, mtx->cmdq_reg.subsys,
                   mtx->addr + DISP_REG_MUTEX_EN(mutex->id), 1);
}
```
This API exists but is NOT called by the CRTC code.

---

## OVL_EN

**Register:** `DISP_REG_OVL_EN = 0x000c` (NOT 0x000!)
For OVL0: `0x14005000 + 0x000c = 0x1400500c`

**Direct writel_relaxed (CPU path)** — `mtk_disp_ovl.c:253-265`:
```c
void mtk_ovl_start(struct device *dev) {
    writel_relaxed(0x1, ovl->regs + DISP_REG_OVL_EN);  // offset 0x000c
}
```

**The `start` callback has NO `cmdq_pkt` parameter** — CMDQ is architecturally impossible for OVL_EN.

---

## MMSYS Routing

**All routing done via direct CPU writes** — `mtk-mmsys.c:192-209`:
```c
void mtk_mmsys_ddp_connect(...) {
    for (i = 0; i < mmsys->data->num_routes; i++)
        mtk_mmsys_update_bits(mmsys, routes[i].addr, routes[i].mask,
                              routes[i].val, NULL);  // NULL cmdq_pkt!
}
```

**MT6789 routing table** (`mt6789-mmsys.h:45-69`):
- 0xf14: OVL0_2L output mux
- 0xf18: OVL0 output mux
- 0xf20: DITHER0 output mux
- 0xf28: RDMA0 input select
- 0xf30: DSI0 input select

---

## Key Takeaway for U-Boot

Direct CPU writes are architecturally valid for ALL display pipeline setup:
- MUTEX_EN, MUTEX_MOD, MUTEX_SOF
- OVL_EN, OVL config, OVL layer config
- MMSYS routing

No GCE/CMDQ implementation needed in U-Boot.
