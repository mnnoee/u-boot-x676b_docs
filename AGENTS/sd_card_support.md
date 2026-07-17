# SD Card Support — MT6789 (Infinix X676B)

## Дата
2026-07-16

## Источники
- stock DTS: `firmware/*/extracted_dtb/main_tree.dts`
- kernel DTS: `src_linux/linux/arch/arm64/boot/dts/mediatek/mt6789.dtsi`
- driver: `u-boot-mtk-g99/drivers/mmc/mtk-sd.c`
- defconfig: `u-boot-mtk-g99/configs/mt6789_infinix_x676b_defconfig`

## Hardware
- **Controller**: MSDC1 @ `0x11240000` (SD/SDIO)
- **Top control**: @ `0x11ef0000`
- **Interrupt**: GIC SPI 135
- **Bus**: 4-bit, SD high-speed, UHS-I (SDR12/25/50/104/DDR50)
- **CD GPIO**: GPIO 7 (active low)
- **Supplies**: vmmc (3.3V), vqmmc (1.8V/3.3V)

## Часы (clocks)
```dts
clocks = <&topckgen CLK_TOP_MSDC30_1_SEL>,     /* source - muxed top clock */
         <&infracfg_ao CLK_IFRAO_MSDC1>,         /* hclk - infra clock gate */
         <&infracfg_ao CLK_IFRAO_MSDC1_SRC>;     /* source_cg - source gate */
clock-names = "source", "hclk", "source_cg";
```

| Clock name | dt-bindings macro | Value | Provider |
|------------|-------------------|-------|----------|
| source     | CLK_TOP_MSDC30_1_SEL | 116 | topckgen @ 0x10000000 |
| hclk       | CLK_IFRAO_MSDC1      | 21  | infracfg_ao @ 0x10001000 |
| source_cg  | CLK_IFRAO_MSDC1_SRC  | 28  | infracfg_ao @ 0x10001000 |

## Compatible
- U-Boot uses `"mediatek,mt8183-mmc"` → `mt8183_compat`
- Kernel uses `"mediatek,mt6789-mmc"` + `"mediatek,mt8183-mmc"`
- `mt8183_compat` has: clk_div_bits=12, pad_tune0=y, async_fifo=y, data_tune=y, busy_check=y, stop_clk_fix=y

## Что нужно для включения

### 1. DTSI (mt6789.dtsi) — раскомментировать часы в sdhc2
### 2. Board DTS (mt6789-infinix-x676b.dts) — добавить cd-gpios, pinctrl
### 3. Defconfig — уже есть CONFIG_MMC_MTK=y, CONFIG_CMD_MMC=y

## Статус на устройстве
- **[ ]** Часы sdhc2 были закомментированы — драйвер не получал source/hclk
- **[ ]** pinctrl не настроен — пины могут быть не переведены в SD-режим
- **[ ]** cd-gpios не указан — fallback (всегда считает карту вставленной)
