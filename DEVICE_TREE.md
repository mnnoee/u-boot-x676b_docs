# Device Tree для Infinix X676B

## Структура DTB

### Реальный DTB (из firmware)

**USB node:**
```dts
usb0@11200000 {
    compatible = "mediatek,mt6789-usb20";
    reg = <0x00 0x11200000 0x00 0x10000 0x00 0x11f40000 0x00 0x10000>;
    clocks = <0x2b 0x22 0x2a 0x1d 0x3b 0x0d>;
    clock-names = "sys_clk", "ref_clk", "src_clk";
    phys = <0x63 0x03>;
    dr_mode = "otg";
    usb-role-switch;
};
```

**USB PHY node:**
```dts
usb-phy@11f40000 {
    compatible = "mediatek,generic-tphy-v2";
    ranges;
    
    u2port0: usb-phy@11f40000 {
        reg = <0x00 0x11f40000 0x00 0x700>;
        #phy-cells = <0x01>;
        mediatek,eye-vrt = <0x04>;
        mediatek,eye-term = <0x04>;
        mediatek,discth = <0x07>;
    };
};
```

### Mainline kernel DTS (`src_linux/linux/arch/arm64/boot/dts/mediatek/mt6789.dtsi`)

**USB node:**
```dts
usb2: usb@11200000 {
    compatible = "mediatek,mt6789-musb",
                 "mediatek,mtk-musb";
    reg = <0 0x11200000 0 0x10000>;
    interrupts = <GIC_SPI 128 IRQ_TYPE_LEVEL_HIGH &ppi_cluster0>;
    interrupt-names = "mc";
    
    clocks = <&infracfg_ao CLK_IFRAO_SSUSB>,
             <&topckgen CLK_TOP_USB_TOP_SEL>,
             <&apmixedsys CLK_APMIXED_USBPLL>;
    clock-names = "main", "mcu", "univpll";
    
    phys = <&u2port0 PHY_TYPE_USB2>;
};
```

**USB PHY node:**
```dts
u2phy: t-phy@11f40000 {
    compatible = "mediatek,mt6789-tphy",
                 "mediatek,generic-tphy-v2";
    #address-cells = <1>;
    #size-cells = <1>;
    ranges = <0 0 0x11f40000 0x1000>;
    
    u2port0: usb-phy@0 {
        reg = <0x0 0x700>;
        clocks = <&clk26m>;
        clock-names = "ref";
        #phy-cells = <1>;
        mediatek,eye-src = <4>;
    };
};
```

### U-Boot DTS (`u-boot-mtk-g99/arch/arm/dts/mt6789.dtsi`)

**USB PHY node (УСТАРЕВШИЙ):**
```dts
u2phy: t-phy@11f40000 {
    compatible = "mediatek,generic-tphy-v2";
    #address-cells = <2>;
    #size-cells = <2>;
    ranges;        ← ПУСТОЙ ranges!
    status = "okay";

    u2port0: usb-phy@0 {
        reg = <0 0x11f40000 0 0x700>;   ← АБСОЛЮТНЫЙ адрес (4 entries)
        clocks = <&clk26m>;
        clock-names = "ref";
        #phy-cells = <1>;
        status = "okay";
        mediatek,eye-vrt = <4>;
        mediatek,eye-term = <4>;
        mediatek,discth = <7>;
    };
};
```

## Проблема: DTB Mismatch

U-Boot использует **НЕПРАВИЛЬНЫЙ** device tree для USB PHY!

### Разница

#### 1. `#address-cells` и `#size-cells`
- **U-Boot**: `<2>` и `<2>` (4 total cells)
- **Kernel**: `<1>` и `<1>` (2 total cells)

#### 2. `ranges` property
- **U-Boot**: пустой `ranges;` — child наследует parent address space 1:1
- **Kernel**: `ranges = <0 0 0x11f40000 0x1000>;` — правильный mapping

#### 3. `reg` в child node
- **U-Boot**: `<0 0x11f40000 0 0x700>` — абсолютный адрес, 4 entries
- **Kernel**: `<0x0 0x700>` — относительный адрес, 2 entries

### Почему это проблема?

Когда PHY driver парсит child node, он ожидает:
```c
// Linux kernel drivers/phy/mediatek/phy-mtk-tphy.c
reg = of_get_address(child, 0, &size, NULL);
```

С `#address-cells = <1>`, `of_get_address()` читает **2 cells** (addr + size).

Но U-Boot DTS даёт **4 cells**, что ломает parsing или даёт неправильный адрес!

### Решение

Исправить `u-boot-mtk-g99/arch/arm/dts/mt6789.dtsi`:

```dts
u2phy: t-phy@11f40000 {
    compatible = "mediatek,mt6789-tphy",
                 "mediatek,generic-tphy-v2";
    #address-cells = <1>;          ← Изменить с <2> на <1>
    #size-cells = <1>;             ← Изменить с <2> на <1>
    ranges = <0 0 0x11f40000 0x1000>;  ← Добавить правильный mapping
    status = "okay";

    u2port0: usb-phy@0 {
        reg = <0x0 0x700>;         ← Изменить на относительный адрес
        clocks = <&clk26m>;
        clock-names = "ref";
        #phy-cells = <1>;
        status = "okay";
        mediatek,eye-vrt = <4>;
        mediatek,eye-term = <4>;
        mediatek,discth = <7>;
    };
};
```

## DTB файлы в проекте

| Файл | Описание |
|------|----------|
| `dtbo/mt6789.dtsi` | Базовый SoC DTS |
| `dtbo/infinix-x676b.dts` | Device-specific DTS |
| `dtbo/infinix-x676b-overlay.dts` | DTBO overlay |
| `dtbo/infinix-x676b-uboot.dts` | DTS для U-Boot |
| `u-boot-x676b/mt6789-infinix-x676b-v*.dts` | Версии DTS для U-Boot |
| `src_linux/linux/arch/arm64/boot/dts/mediatek/mt6789.dtsi` | Mainline kernel DTS |
| `u-boot-mtk-g99/arch/arm/dts/mt6789.dtsi` | U-Boot DTS (текущий) |

## Декомпиляция DTB

```bash
dtc -I dtb -O dts input.dtb -o output.dts
```

## Сравнение DTB

Для сравнения DTB файлов можно использовать:
```bash
dtc -I dtb -O dts file1.dtb -o file1.dts
dtc -I dtb -O dts file2.dtb -o file2.dts
diff file1.dts file2.dts
```
