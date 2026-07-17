# USB Host Status on MT6789 (Infinix X676B)

**Date**: 2026-07-16
**Agent**: USB research session

---

## 1. Тип USB-контроллера на MT6789

**MT6789 НЕ имеет xHCI или EHCI хост-контроллера.** На SoC установлен только один USB-контроллер — **MUSB (musb-hdrc)** dual-role (OTG) на физическом адресе `0x11200000`.

### Stock DTS (firmware/.../main_tree.dts)
```dts
usb: usb0@11200000 {
    compatible = "mediatek,mt6789-usb20";
    reg = <0x00 0x11200000 0x00 0x10000
           0x00 0x11f40000 0x00 0x10000>;
    interrupts = <0x00 0x80 0x04 0x00>;
    clock-names = "sys_clk", "ref_clk", "src_clk";
    phys = <&u2port0 3>;
    dr_mode = "otg";
    ...
};
```

### U-Boot DTSI (arch/arm/dts/mt6789.dtsi)
```dts
usb: usb0@11200000 {
    compatible = "mediatek,mt8518-musb", "mediatek,mtk-musb";
    reg = <0x11200000 0x10000>;
    phys = <&u2port0 PHY_TYPE_USB2>;
    ...
};
```

### USB PHY
- **USB20 PHY** at `0x11f40000` (USB2.0 только — нет USB3 SuperSpeed)
- Регистры PHY: 0x318, 0x31c, 0x320, 0x368, 0x36c (из стокового LK)
- PHY programming уже реализован в `mt85xx.c` (функции `mt_usb_phy_poweron/savecurrent/recover`)

### Сравнение: vicky defconfig (другой девайс)
В `mt6789_vicky_defconfig` есть `CONFIG_USB_XHCI_HCD=y` и `CONFIG_USB_MTU3=y` — это для другого варианта платформы, НЕ для X676B. На X676B нет xHCI.

---

## 2. Текущая конфигурация USB (только gadget, без host)

**Файл**: `configs/mt6789_infinix_x676b_defconfig`

Строки 113–119:
```
CONFIG_USB=y                          # Включена USB-подсистема
CONFIG_USB_MUSB_GADGET=y              # Режим периферии (gadget) ВКЛ
CONFIG_USB_MUSB_PIO_ONLY=y            # PIO без DMA
CONFIG_USB_MUSB_MT85XX=y              # Драйвер MediaTek MT85XX MUSB
# CONFIG_DM_USB_GADGET is not set     # DM USB gadget отключён
CONFIG_USB_GADGET=y                   # Фреймворк gadget
CONFIG_USB_FUNCTION_ACM=y             # ACM (serial) функция
```

### Чего НЕ хватает (критично для USB-флешек):
```
# CONFIG_USB_MUSB_HOST is not set     # Режим host ОТКЛЮЧЁН
# CONFIG_USB_STORAGE is not set       # Mass storage ОТКЛЮЧЁН
```

### `.config` (собранный):
```
CONFIG_DM_USB=y
# CONFIG_USB_MUSB_HOST is not set
CONFIG_USB_MUSB_GADGET=y
```

`DM_USB=y` вероятно выбран какой-то другой опцией, но это не имеет значения MUSB_HOST выключен.

---

## 3. Анализ драйвера `drivers/usb/musb-new/mt85xx.c`

### Как работает выбор режима host/gadget

**В `musb_usb_probe()` (строка 309):**
```c
#ifdef CONFIG_USB_MUSB_HOST
    pdata.mode = MUSB_HOST;
    host->host = musb_init_controller(&pdata, &glue->dev, base);
    ret = musb_lowlevel_init(host);
    printf("MTK MUSB OTG (Host)\n");
#else
    pdata.mode = MUSB_PERIPHERAL;
    host->host = musb_register(&pdata, &glue->dev, base);
    printf("MTK MUSB OTG (Peripheral)\n");
#endif
```

**В регистрации драйвера (строка 437):**
```c
U_BOOT_DRIVER(mtk_musb) = {
    .name   = "mtk_musb",
#ifdef CONFIG_USB_MUSB_HOST
    .id     = UCLASS_USB,               // Host регистрируется как USB host
#else
    .id     = UCLASS_USB_GADGET_GENERIC, // Gadget регистрируется как gadget
#endif
    ...
};
```

**Вывод**: При текущей конфигурации драйвер:
- Регистрируется как `UCLASS_USB_GADGET_GENERIC` (не как UCLASS_USB)
- Инициализируется в режиме `MUSB_PERIPHERAL`
- **Не может быть обнаружен `usb start` командой** (она ищет UCLASS_USB)

### PHY инициализация
Драйвер уже включает полную инициализацию USB20 PHY:
- `mt_usb_phy_poweron()` подача питания на PHY
- `mt_usb_phy_savecurrent()` сохранение состояния перед выключением
- `mt_usb_phy_recover()` восстановление PHY (вызывается из `mtk_musb_enable`)
Это работает для обоих режимов.

### Поддержка MT85XX
`CONFIG_USB_MUSB_MT85XX` стандартная опция для MediaTek MUSB. Она включает компиляцию mt85xx.c.

---

## 4. Что нужно для работы USB-флешек

### Цепочка зависимостей Kconfig:

```
CONFIG_USB_MUSB_HOST (bool)
    depends on DM && OF_CONTROL
    select USB_HOST                        # Автовключает USB_HOST

USB_HOST (bool, из drivers/usb/host/Kconfig)
    select DM_USB                          # Автовключает DM_USB

CONFIG_USB_STORAGE (bool, из drivers/usb/Kconfig)
    depends on USB_HOST                    # USB_HOST должен быть включен
```

### Минимальные изменения в defconfig:

Добавить в `configs/mt6789_infinix_x676b_defconfig`:
```
CONFIG_USB_MUSB_HOST=y    # Включает host-режим MUSB
CONFIG_USB_STORAGE=y      # Включает поддержку USB mass storage (флешки)
```

Это автоматически:
- `select USB_HOST` > `select DM_USB`
- Переключит драйвер mt85xx.c в host-режим (UCLASS_USB, MUSB_HOST)
- Включит `usb_storage` драйвер
- Позволит командам `usb start`, `usb tree`, `usb storage` работать

### Не нужно (уже есть):
- `CONFIG_CMD_USB=y`              (строка 54)
- `CONFIG_CMD_USB_MASS_STORAGE=y` (строка 55) это для UMS (USB Mass Storage gadget, не путать)
- `CONFIG_USB_MUSB_MT85XX=y`      (строка 116)
- `CONFIG_USB_MUSB_PIO_ONLY=y`    (строка 115)

### Возможный конфликт:
`CONFIG_USB_MUSB_GADGET` и `CONFIG_USB_MUSB_HOST` могут быть включены **одновременно**. В этом случае:
- В `musb_usb_probe()` приоритет отдаётся `MUSB_HOST` (т.к. `#ifdef CONFIG_USB_MUSB_HOST` проверяется первым)
- Gadget функциональность (fastboot, ACM) может перестать работать через MUSB (но fastboot использует `CONFIG_USB_FUNCTION_FASTBOOT`, который может работать через отдельный механизм нужно тестировать)

**Рекомендация**: Сначала попробовать включить только `CONFIG_USB_MUSB_HOST=y` и `CONFIG_USB_STORAGE=y`, **удалив** `CONFIG_USB_MUSB_GADGET=y`, чтобы избежать конфликтов. Если fastboot нужен, его можно добавить через `CONFIG_USB_FUNCTION_FASTBOOT` (он не зависит от MUSB_GADGET).

---

## 5. Сводка

| Компонент | Статус | Что нужно |
|-----------|--------|-----------|
| xHCI/EHCI | Нет на SoC | Использовать MUSB |
| MUSB gadget | Включён | Работает для serial/fastboot |
| MUSB host | Отключён | `CONFIG_USB_MUSB_HOST=y` |
| USB mass storage | Отключён | `CONFIG_USB_STORAGE=y` |
| PHY init | Готова | mt85xx.c уже программирует PHY |
| DTS compat | Готова | `mediatek,mt8518-musb` совпадает с mt85xx.c |

### Команды для тестирования после пересборки (на устройстве):
```
=> usb start             # Инициализировать USB host
=> usb tree              # Показать дерево USB устройств
=> usb storage           # Показать найденные storage устройства
=> ls usb 0:1            # Список файлов на флешке
=> fatload usb 0:1 0x40000000 boot.scr   # Загрузка с флешки
```

### Ссылки:
- Драйвер: `u-boot-mtk-g99/drivers/usb/musb-new/mt85xx.c`
- Defconfig: `u-boot-mtk-g99/configs/mt6789_infinix_x676b_defconfig`
- DTS: `u-boot-mtk-g99/arch/arm/dts/mt6789.dtsi` (строка 344)
- Stock DTS: `firmware/Infinix_Note_12_Pro_X676B_MT6789_V2385_221208_MXML/Firmware/extracted_dtb/main_tree.dts` (строка 5783)
- Kconfig MUSB: `u-boot-mtk-g99/drivers/usb/musb-new/Kconfig`
- Kconfig USB: `u-boot-mtk-g99/drivers/usb/Kconfig`
- Kconfig host: `u-boot-mtk-g99/drivers/usb/host/Kconfig`
