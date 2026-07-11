# Инструменты и окружение

## Устройство
- Infinix X676B (NOTE 12 PRO), MediaTek MT6789 (Helio G99)
- Linux 5.10.149 GKI, Android 12, A/B разметка
- ADB root доступен

## Кросс-компилятор
- `/usr/bin/aarch64-linux-gnu-gcc` — GCC для ARM64

## Python venv
- `/workdir/git/heliog99/mtkclient/venv/bin/python3`
- Установлен `liblk` (v3.2.0) — упаковка U-Boot в LK формат
- Установлен `compiledb` — генерация compile_commands.json

**ВАЖНО**: Всегда использовать venv для Python пакетов. Не устанавливать пакеты системно!

## U-Boot сборка
```bash
cd /workdir/git/heliog99/u-boot-mtk-g99

# Конфигурация
make CROSS_COMPILE=aarch64-linux-gnu- mt6789_infinix_x676b_defconfig

# Сборка
make CROSS_COMPILE=aarch64-linux-gnu- -j$(nproc)

# Упаковка в lk.img
/workdir/git/heliog99/mtkclient/venv/bin/python3 gimme-uboot.py \
  /workdir/git/heliog99/u-boot-x676b/lk_a_dump.img \
  u-boot.bin \
  /workdir/git/heliog99/u-boot-x676b/uboot-lk-x676b-v{N}.img

# Очистка после сборки (ВАЖНО - не оставлять мусор!)
rm -rf build .config u-boot*
```

## Прошивка
```bash
# Через fastboot (если U-Boot работает)
adb reboot bootloader
fastboot flash lk_a uboot-lk-x676b-v{N}.img
fastboot flash lk_b uboot-lk-x676b-v{N}.img
fastboot reboot

# Через mtkclient (DA mode)
cd /workdir/git/heliog99/mtkclient
python mtk w lk_a ../u-boot-x676b/uboot-lk-x676b-v{N}.img
python mtk w lk_b ../u-boot-x676b/uboot-lk-x676b-v{N}.img
```

## UART Log
- Picocom: `picocom -b 921600 /dev/ttyUSB0`
- Логи сохраняются в `u-boot-x676b/uart-log-v{N}.txt`
- Читать через: `strings uart-log-v{N}.txt`

## Keypad и кнопки (v89)

### Маппинг кнопок
| Кнопка | Символ | Действие в bootmenu |
|--------|--------|---------------------|
| Volume Up | `\r` (Enter) | Выбор пункта |
| Volume Down | `ESC [ B` | Скролл вниз |

### Навигация
- **Volume Down** — движение вниз, зациклено: последний → первый
- **Volume Up** — выбор текущего пункта (или Enter)
- **Зацикливание**: первый → последний, последний → первый

### Тестирование кнопок
```
=> button vup
on
=> button vdown
on
=> button list
vdown           on
vup             on
```

### Реализация
- `common/console.c` — `button_input_poll()` опрашивает keypad и добавляет символы в FIFO буфер
- `cmd/bootmenu.c` — зацикливание навигации (BKEY_DOWN, BKEY_UP)
- `drivers/button/button-mtk-kp.c` — драйвер keypad (MT6789矩阵 контроллер)

## Анализ DTB
- Реальный DTB из firmware: `заметки/devtree/base_tree.dts`
- DTB из mainline kernel: `src_linux/linux/arch/arm64/boot/dts/mediatek/mt6789.dtsi`
- DTB в U-Boot: `u-boot-mtk-g99/arch/arm/dts/mt6789.dtsi`

## Декомпиляция DTB
```bash
dtc -I dtb -O dts input.dtb -o output.dts
```

## MT6789 USB PHY Registers (из stock LK)
Register bank 0x11f40300+ (НЕ 0x000+!):
- `0x318` — PHY control 1
- `0x31c` — PHY control 2
- `0x320` — PHY control 3
- `0x368` — PHY analog control
- `0x36c` — PHY calibration (eye-vrt, eye-term, discth, rx_sqth)

Stock LK functions:
- `FUN_0004a1f0` — poweron
- `FUN_0004a240` — savecurrent
- `FUN_0004a2bc` — recover + calibration

## LSP (Language Server Protocol)

### Генерация compile_commands.json

**U-Boot:**
```bash
cd /workdir/git/heliog99/u-boot-mtk-g99
make CROSS_COMPILE=aarch64-linux-gnu- mt6789_infinix_x676b_defconfig
python3 scripts/gen_compile_commands.py
```

**Linux kernel:**
```bash
cd /workdir/git/heliog99/src_linux/linux
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- defconfig
/workdir/git/heliog99/mtkclient/venv/bin/compiledb -o compile_commands.json
```

### Установка clangd
clangd уже установлен: `/usr/bin/clangd`

Для работы с project-специфичными флагами, clangd читает `.clangd` файл в корне проекта.
