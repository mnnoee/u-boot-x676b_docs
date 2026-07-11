# MT6789 (Helio G99) — Карта регистров

## Источники
- Decompiled LK (`lk_unpack_v0/lk_a_lk.bin.c`)
- Linux kernel (`src_linux/linux/drivers/gpu/drm/mediatek/`, `drivers/soc/mediatek/`, `drivers/clk/mediatek/`)
- DTB из прошивки (`firmware/.../extracted_dtb/main_tree.dts`)
- GitHub forks (Volla, Xiaomi, OPPO MT6789 kernels)
- lore.kernel.org (MT6779/MT6789 patches)

---

## Display Subsystem (MMSYS) — Base: 0x14000000

| Block | Base | Size | Clock Gate Bit | Compatible |
|-------|------|------|----------------|------------|
| MMSYS config | `0x14000000` | 0x1000 | — | `mediatek,mt6789-mmsys` |
| SMI Common | `0x14002000` | 0x1000 | — | `mediatek,mt6789-smi-common` |
| MUTEX | `0x14001000` | 0x1000 | MM0 bit 0 | `mediatek,disp_mutex0` |
| OVL0 | `0x14005000` | 0x1000 | MM0 bit 2 | `mediatek,disp_ovl0` |
| OVL_2L0 | `0x14006000` | 0x1000 | MM0 bit 4 | `mediatek,disp_ovl0_2l` |
| RDMA0 | `0x14007000` | 0x1000 | MM0 bit 3 | `mediatek,disp_rdma0` |
| COLOR0 | `0x14009000` | 0x1000 | MM0 bit 10 | `mediatek,disp_color0` |
| CCORR0 | `0x1400b000` | 0x1000 | MM0 bit 9 | `mediatek,disp_ccorr0` |
| AAL0 | `0x1400c000` | 0x1000 | MM0 bit 8 | `mediatek,disp_aal0` |
| GAMMA0 | `0x1400d000` | 0x1000 | MM0 bit 13 | `mediatek,disp_gamma0` |
| POSTMASK0 | `0x1400e000` | 0x1000 | MM0 bit 14 | `mediatek,disp_postmask0` |
| DITHER0 | `0x1400f000` | 0x1000 | MM0 bit 16 | `mediatek,disp_dither0` |
| DSC0 | `0x14012000` | 0x1000 | — | `mediatek,disp_dsc_wrap0` |
| DSI0 | `0x14013000` | 0x1000 | MM0 bit 19, MM1 bit 0 | `mediatek,mt6789-dsi` |
| WDMA0 | `0x14014000` | 0x1000 | MM0 bit 5 | `mediatek,disp_wdma0` |
| SMI sub-common u0 | `0x1401b000` | 0x1000 | — | `mediatek,mt6789-smi-common` |
| SMI sub-common u1 | `0x1401c000` | 0x1000 | — | `mediatek,mt6789-smi-common` |

### MMSYS Clock Gate Registers (MT6789)

**`DISP_REG_CONFIG_MMSYS_CG_CON0` = 0x100** (relative to 0x14000000)

| Bit | Clock Name | Источник |
|-----|------------|----------|
| 0 | disp_mutex0 | clk-mt6789-mm.c |
| 1 | apb_bus | clk-mt6789-mm.c |
| 2 | disp_ovl0 | clk-mt6789-mm.c |
| 3 | disp_rdma0 | clk-mt6789-mm.c |
| 4 | disp_ovl0_2l | clk-mt6789-mm.c |
| 5 | disp_wdma0 | clk-mt6789-mm.c |
| 7 | disp_rsz0 | clk-mt6789-mm.c |
| 8 | disp_aal0 | clk-mt6789-mm.c |
| 9 | disp_ccorr0 | clk-mt6789-mm.c |
| 10 | disp_color0 | clk-mt6789-mm.c |
| 13 | disp_gamma0 | clk-mt6789-mm.c |
| 14 | disp_postmask0 | clk-mt6789-mm.c |
| 16 | disp_dither0 | clk-mt6789-mm.c |
| 19 | disp_dsi0 | clk-mt6789-mm.c |
| 22 | smi_common | clk-mt6789-mm.c |

### MMSYS Routing Registers

| Register | Offset | Описание |
|----------|--------|----------|
| `DISP_REG_CONFIG_DISP_OVL0_MOUT_EN` | 0x040 | OVL0 output mux enable |
| `DISP_REG_CONFIG_DISP_OVL1_MOUT_EN` | 0x044 | OVL1 output mux enable |
| `DISP_REG_CONFIG_DISP_OD_MOUT_EN` | 0x048 | OD output mux enable |
| `DISP_REG_CONFIG_DISP_GAMMA_MOUT_EN` | 0x04C | GAMMA output mux enable |
| `DISP_REG_CONFIG_DISP_UFOE_MOUT_EN` | 0x050 | UFOE output mux enable |
| `DISP_REG_CONFIG_DISP_COLOR0_SEL_IN` | 0x084 | COLOR0 input select |
| `DISP_REG_CONFIG_DSIE_SEL_IN` | 0x0A4 | DSIE input select |
| `DISP_REG_CONFIG_DSIO_SEL_IN` | 0x0A8 | DSIO input select |
| `DISP_REG_CONFIG_DPI_SEL_IN` | 0x0AC | DPI input select |
| `DISP_REG_CONFIG_DISP_RDMA0_SOUT_EN` | 0x0C4 | RDMA0 output select |

### MUTEX Registers (от 0x14001000)

| Register | Формула | Описание |
|----------|---------|----------|
| `DISP_REG_MUTEX_EN(n)` | 0x20 + 0x20*n | Mutex n enable |
| `DISP_REG_MUTEX(n)` | 0x24 + 0x20*n | Mutex n control |
| `DISP_REG_MUTEX_RST(n)` | 0x28 + 0x20*n | Mutex n reset |
| `DISP_REG_MUTEX_MOD(n)` | 0x2C + 0x20*n | Mutex n module map |
| `DISP_REG_MUTEX_SOF(n)` | 0x30 + 0x20*n | Mutex n signal select |
| `DISP_REG_MUTEX_MOD2(n)` | 0x34 + 0x20*n | Mutex n module map 2 |

---

## DSI (MIPI DSI Controller) — Base: 0x14013000

### DSI из DTB прошивки
```
dsi@14013000 {
    compatible = "mediatek,mt6789-dsi";
    reg = <0x00 0x14013000 0x00 0x1000>;
};
```

### DSI Panel (ILI7835) — из декомпилированного LK
- **Panel name**: `ili7835_fhdp_dsi_cmd_edo_edo_x67`
- **Resolution**: 1080 x 2400 (0x438 x 0x960)
- **Color depth**: 32 bpp
- **DSI lanes**: 4
- **DSI clock divider**: 256
- **Mode**: Command mode
- **Compatible (MIPI TX)**: `mediatek,mt6789-mipi-tx-cphy`

### DSI Register Map (из mtk_dsi.c + LK)

| Offset | Register | Описание | LK Access |
|--------|----------|----------|-----------|
| **0x000** | `DSI_START` | Command mode trigger (bit 0 = kick) | 5 accesses: `&0xfffffffe \| 1` |
| **0x008** | `DSI_INTEN` | Interrupt enable | — |
| **0x00c** | `DSI_INTSTA` | Interrupt status (poll + write-1-to-clear) | 1 access: `= ~val` |
| **0x010** | `DSI_CON_CTRL` | Control (bit 0=RESET, bit 1=EN, bit 2=DPHY_RESET) | — |
| **0x014** | `DSI_MODE_CTRL` | Tx mode (0=CMD, 1=SYNC_PULSE, 2=SYNC_EVENT, 3=BURST) | 2 accesses: `& 3` (busy check) |
| **0x018** | `DSI_TXRX_CTRL` | Lane config (bits [5:2]=LANE_NUM, bit 9=EXT_TE_EN) | — |
| **0x01c** | `DSI_PSCTRL` | Pixel stream (bits [13:0]=WC, bits [17:16]=format) | — |
| **0x020** | `DSI_VSA_NL` | Vertical sync active | — |
| **0x024** | `DSI_VBP_NL` | Vertical back porch | — |
| **0x028** | `DSI_VFP_NL` | Vertical front porch | — |
| **0x02c** | `DSI_VACT_NL` | Vertical active lines | — |
| **0x038** | `DSI_SIZE_CON` | Size (bits [30:16]=height, bits [14:0]=width) | — |
| **0x050** | `DSI_HSA_WC` | Horizontal sync active word count | — |
| **0x054** | `DSI_HBP_WC` | Horizontal back porch word count | — |
| **0x058** | `DSI_HFP_WC` | Horizontal front porch word count | — |
| **0x060** | `DSI_CMDQ_SIZE` | Command queue size | — |
| **0x064** | `DSI_HSTX_CKL_WC` | High-speed Tx clock word count | — |
| **0x06c** | `DSI_PHY_TIMECON` | PHY timing control | 1 access: `& 0xfffbffff` |
| **0x074** | `DSI_RX_DATA0` | Receive data 0 | — |
| **0x078** | `DSI_RX_DATA1` | Receive data 1 | — |
| **0x07c** | `DSI_RX_DATA2` | Receive data 2 | — |
| **0x080** | `DSI_RX_DATA3` | Receive data 3 | — |
| **0x084** | `DSI_RACK` | Receive acknowledge (bit 0) | — |
| **0x104** | `DSI_PHY_LCCON` | PHY lane clock (bit 0=HS_TX_EN, bit 1=ULPM_EN) | — |
| **0x108** | `DSI_PHY_LD0CON` | PHY lane data 0 (bit 0=HS_TX_EN) | — |
| **0x110** | `DSI_PHY_TIMECON0` | PHY timing 0 (LPX, HS_PREP, HS_ZERO, HS_TRAIL) | — |
| **0x114** | `DSI_PHY_TIMECON1` | PHY timing 1 (TA_GO, TA_SURE, TA_GET, DA_HS_EXIT) | — |
| **0x118** | `DSI_PHY_TIMECON2` | PHY timing 2 (CONT_DET, DA_HS_SYNC, CLK_ZERO, CLK_TRAIL) | — |
| **0x11c** | `DSI_PHY_TIMECON3` | PHY timing 3 (CLK_HS_PREP, CLK_HS_POST, CLK_HS_EXIT) | — |
| **0x130** | `DSI_VM_CMD_CON` | VM command control (bit 0=VM_CMD_EN, bit 5=TS_VFP_EN) | 1 access: `\|= 0x21` |
| **0xd00** | DSI_CMDQ | Command queue base (mt6789 specific) | — |

### DSI Function Pointers (из LK)

| Переменная | Вероятная функция | Вызывается с |
|------------|-------------------|-------------|
| `_DAT_0010ab58` | DSI DCS write (single byte) | `(cmd_byte)` |
| `_DAT_0010ab88` | DSI DCS write (command + params) | `(cmd_id, param_byte, data_ptr, count)` |
| `_DAT_0010ab98` | DSI generic write / VM command | `(payload_ptr, word_count, flag)` |

### DSI Command Mode Flow (из LK)

```
1. Записать данные в CMDQ (через _DAT_0010ab88/_DAT_0010ab98)
2. Проверить DSI_STATUS (0x014) == idle (bits [1:0] == 0)
3. Записать DSI_START (0x000): bit 0 clear → set (kick)
4. Ждать DSI_INTSTA (0x00c): CMD_DONE_INT_FLAG (bit 1)
```

### DSI GPIO Pin Mapping (из LK)

**DSI Port 1 (GPIO Bank 0x11D40000)**:
| Register | Функция |
|----------|---------|
| `0x11D40100` | Pull-up/pull-down enable (bits 0-9) |
| `0x11D40090` | GPIO direction |
| `0x11D400B0` | Output data |
| `0x11D400C0` | Output enable |
| `0x11D40120` | Drive strength (bits [11:8], [23:20]) |
| `0x11D400D0` | GPIO mode mux |
| `0x11D40000` | Pin select (CLK/DATA mapping) |
| `0x11D40010` | Pin selection additional |

**DSI Port 0 (GPIO Bank 0x11E20000)**:
| Register | Функция |
|----------|---------|
| `0x11E201E0` | DSI data pin modes (bits 15-17) |
| `0x11E20120` | Pull config |
| `0x11E20160` | Pull config |
| `0x11E20170` | Pull config |
| `0x11E20200` | Drive strength [31:28] |
| `0x11E20210` | Drive strength full |
| `0x11E20220` | Drive strength [3:0], [7:4] |
| `0x11E20180` | GPIO mode bits [27:14] |
| `0x11E20190` | GPIO mode |
| `0x11E201A0` | GPIO mode |
| `0x11E20030` | CLK/data pin selection |
| `0x11E20020` | Data pin selection |
| `0x11E200D0` | GPIO mode (pins 12-15) |

**IOCFG (Pin Mux)**:
| Register | Значение | Описание |
|----------|----------|----------|
| `0x100053A0` | `0x11111100` | DSI port 1 pin mux |
| `0x10005360` | `0x11000000` | DSI port 0 pin mux (high byte) |
| `0x10005370` | `0x11111111` | DSI port 0 pin mux (all data) |
| `0x10005380` | `0x11` | DSI port 0 pin mux (low byte) |

---

## GCE (Global Command Engine / CMDQ) — Base: 0x10228000

```
gce@10228000 {
    compatible = "mediatek,mt6789-gce";
    reg = <0x00 0x10228000 0x00 0x4000>;
};
```

### GCE Display Event IDs (из mt6779-gce.h)

| Event | ID | Описание |
|-------|----|----------|
| `CMDQ_EVENT_DISP_RDMA0_SOF` | 0 | RDMA0 start-of-frame |
| `CMDQ_EVENT_DISP_OVL0_SOF` | 9 | OVL0 start-of-frame |
| `CMDQ_EVENT_DISP_WDMA0_SOF` | 12 | WDMA0 start-of-frame |
| `CMDQ_EVENT_DISP_COLOR0_SOF` | 13 | COLOR0 start-of-frame |
| `CMDQ_EVENT_DISP_RDMA0_EOF` | 28 | RDMA0 end-of-frame |
| `CMDQ_EVENT_DISP_DSI0_SOF` | 19 | DSI0 start-of-frame |
| `CMDQ_EVENT_DISP_MASK0_SOF` | 21 | MASK0 start-of-frame |

---

## Display Power Management (MTCMOS)

### Power ON Sequence (из LK)
```
1. 0x10006398 &= 0xFFFFFFFC    // TOPAXI_PROTECTEN clear [1:0]
2. 0x10006300 |= 5             // TOPAXI_PROTECTEN enable bits [0,2]
3. Poll 0x1000616c & 1 == 1    // Wait for power ready
4. 0x10001B88 = 0x8401         // MTCMOS power-up sequence
5. 0x100012A4 = 0x80           // Enable MTCMOS
```

### Power OFF Sequence (из LK)
```
1. 0x100012A0 = 0x80           // MTCMOS power-down
2. Poll 0x10001228 >> 7 & 1    // Wait for power down status
3. 0x10001B84 = 0x8401         // Power-down sequence
4. Poll 0x10001B90 ^ 0xFFFFFFFF & 0x8401 != 0  // Wait for ack
5. 0x10006300 &= 0xFFFFFFFB    // Clear protection
6. Poll 0x1000616c & 1 == 0    // Wait for idle
7. 0x10006398 |= 3             // Set protection
8. 0x10006300 &= 0xFFFFFFFE    // Disable
```

### Display Power Domain
- Power domain index: `0x0A` (в SCPSYS)
- SCPSYS base: `0x10006000`

---

## Infra/Top Registers

| Register | Адрес | Описание |
|----------|-------|----------|
| TOPAXI_PROTECTEN | `0x10006300` | Bus protection enable |
| TOPAXI_PROTECTEN_STA | `0x1000616c` | Bus protection status |
| TOPAXI_PROTECTEN_MM | (string ref) | MM subsystem bus protection |
| dis_pwr_con | (string ref) | Display power controller |
| DPM start1 | `0x100063BC` | DPM trigger (bit 0 = start, bit 4 = done) |
| MTCMOS control | `0x100012A0`-`0x100012A4` | Power switch |
| MTCMOS sequence | `0x10001B84`-`0x10001B90` | Power sequencing |

---

## Clock Controllers

### MT6789 Clock Files (kernel source)
| File | Base | Описание |
|------|------|----------|
| `clk-mt6789-topckgen.c` | `0x10000000` | Top clock generator |
| `clk-mt6789-infra_ao.c` | `0x10001000` | Infra always-on |
| `clk-mt6789-apmixed.c` | `0x1000C000` | AP mixed |
| `clk-mt6789-mm.c` | `0x14000000` | Multimedia (display) |
| `clk-mt6789-mdp.c` | `0x14020000` | MDP |
| `clk-mt6789-venc.c` | `0x17000000` | Video encoder |
| `clk-mt6789-vdec.c` | `0x16000000` | Video decoder |
| `clk-mt6789-cam.c` | `0x1A000000` | Camera |
| `clk-mt6789-img.c` | `0x15020000` | Image |
| `clk-mt6789-audiosys.c` | — | Audio |
| `clk-mt6789-mfgcfg.c` | — | GPU |

---

## DISP PWM — Base: 0x1100E000

```
disp_pwm0@1100e000 {
    compatible = "mediatek,disp_pwm0", "mediatek,mt6789-disp-pwm";
    reg = <0x00 0x1100e000 0x00 0x1000>;
    clocks = <0x2b 0x23 0x2a 0x1c 0x2a 0x85>;
    clock-names = "main", "mm", "pwm_src";
};
```

---

## SPI Controller — Base: 0x10007000

| Register | Адрес | Описание |
|----------|-------|----------|
| SPI_CFG | `0x10007000` | Control register |
| SPI_CFG | `0x10007008` | Clock divider/mode (= 0x1971) |
| SPI_CFG | `0x10007004` | CS/timing config |
| SPI_CFG | `0x10007014` | Timing (= 0x1209) |
| SPI_CFG | `0x10007030` | FIFO enable |
| SPI_CFG | `0x10007038` | FIFO config |
| SPI_CFG | `0x10007048` | Transaction command |
| SPI_CFG | `0x10007088` | Pattern (= 0x1B2A) |
| SPI_CFG | `0x1000708C` | Pattern (= 0x4F59) |

---

## GPIO/Pinctrl — Pin Base Addresses

| Bank | Base | Описание |
|------|------|----------|
| GPIO | `0x10005000` | Main GPIO controller |
| IOCFG_RM | `0x11C20000` | IO config right-middle |
| IOCFG_BR | `0x11D10000` | IO config bottom-right |
| IOCFG_LM | `0x11E20000` | IO config left-middle (DSI port 0) |
| IOCFG_LB | `0x11E70000` | IO config left-bottom |
| IOCFG_RT | `0x11EA0000` | IO config right-top |
| IOCFG_LT | `0x11F20000` | IO config left-top |
| IOCFG_TL | `0x11F30000` | IO config top-left |
| EINT | `0x1000B000` | External interrupt |

---

## GCE Subsystem Mapping (из DTB)

```
gce-subsys = <
    0x66 0x14000000 0x01    // MMSYS config → subsys_id 1
    0x66 0x14010000 0x02    // DISP (DSI?) → subsys_id 2
    0x66 0x14020000 0x03    // MDP → subsys_id 3
>;
```

---

## MMDVFS (Voltage/Frequency Scaling)

### Display OPP Table
| OPP | Frequency | Voltage |
|-----|-----------|---------|
| 0 | 208 MHz (0xC65D400) | 550000 uV |
| 1 | 312 MHz (0x1298BE00) | 600000 uV |
| 2 | 412 MHz (0x18CBA800) | 650000 uV |
| 3 | 545 MHz (0x208B4C80) | 724000 uV |

### MDP OPP Table
| OPP | Frequency | Voltage |
|-----|-----------|---------|
| 0 | 228 MHz (0xDA64340) | 550000 uV |
| 1 | 343 MHz (0x14810600) | 600000 uV |
| 2 | 435 MHz (0x19FCD500) | 650000 uV |
| 3 | 593 MHz (0x2367B880) | 724000 uV |

### MUX Paths
- **disp0**: TOP_UNIVPLL_D6_D2 → TOP_UNIVPLL_D4_D2 → TOP_UNIVPLL_D6 → TOP_MAINPLL_D4
- **mdp0**: TOP_MMPLL_D5_D2 → TOP_MMPLL_D4_D2 → TOP_MAINPLL_D5 → TOP_TVDPLL

---

## Framebuffer и FPS (из декомпилированного LK)

### _DAT_0010a258 — FB Physical Base Address
- **Запись**:唯一 в `FUN_00044cdc` (line 49014)
- **Значение**: `*puVar5` — из mblock query по имени `"framebuffer"`
- **literal 0xfe0c8000**: НЕ найден в коде. Адрес динамический из mblock allocator
- **Цепочка**: `FUN_00010a00("framebuffer")` → mblock entry → `*entry` → `_DAT_0010a258`

### _DAT_0010a274 — FPS Value
- **Запись**: `FUN_00045050` (lines 49256-49282)
- **Вычисление**: `FUN_00045ec4()` → `100000000 / delta` (units: 1/100 fps)
- **Default**: `6000` (= 60.00 fps) если FPS detection fails
- **Формат**: `"%s, fps=%d"` — логируется при инициализации
- **Подтверждение**: continuous-scan гипотеза — FPS в 1/100 единицах, default 6000 = 60 fps

---

## Ссылки

- Kernel: `src_linux/linux/drivers/gpu/drm/mediatek/mtk_disp_*.c`
- Clocks: `src_linux/linux/drivers/clk/mediatek/clk-mt6789-mm.c`
- MMSYS: `src_linux/linux/drivers/soc/mediatek/mt6789-mmsys.h`
- DTB: `firmware/.../extracted_dtb/main_tree.dts`
- LK: `lk_unpack_v0/lk_a_lk.bin.c`
- GitHub: `github.com/HelloVolla/android_kernel_volla_mt8781` (MT6789 BSP DTS)
