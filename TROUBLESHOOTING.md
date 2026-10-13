# Решение проблем

## Сборка

### Ошибка: `aarch64-linux-gnu-gcc: command not found`
Установить кросс-компилятор:
```bash
# Arch Linux
sudo pacman -S aarch64-linux-gnu-gcc

# Ubuntu/Debian
sudo apt install gcc-aarch64-linux-gnu
```

### Ошибка: `scripts/gen_compile_commands.py: No such file or directory`
Убедиться что вы в директории U-Boot:
```bash
cd /workdir/git/heliog99/u-boot-mtk-g99
python3 scripts/gen_compile_commands.py
```

### Ошибка: `ModuleNotFoundError: No module named 'compiledb'`
Использовать venv:
```bash
/workdir/git/heliog99/mtkclient/venv/bin/pip install compiledb
```

## Прошивка

### Ошибка: `fastboot: error: cannot get notification channel`
Перезапустить adb сервер:
```bash
adb kill-server
adb start-server
adb reboot bootloader
```

### Ошибка: `FAILED (remote: not allowed)`
Убедиться что bootloader unlocked:
```bash
fastboot oem unlock
```

### Ошибка: `error: cannot load'uboot-lk-x676b-v{N}.img'`
Проверить путь к файлу:
```bash
ls -la /workdir/git/heliog99/u-boot-x676b/uboot-lk-x676b-v*.img
```

## UART

### Ошибка: `picocom: /dev/ttyUSB0: No such file or directory`
Проверить подключение UART адаптера:
```bash
ls -la /dev/ttyUSB*
dmesg | grep tty
```

### Нет данных в UART
- Проверить baud rate: `921600`
- Проверить подключение TX/RX/GND
- Проверить testpoints на устройстве (рядом с fingerprint connector)

## U-Boot

### U-Boot не загружается
- Проверить UART лог
- Убедиться что образ прошит правильно:
  ```bash
  fastboot flash lk_a /workdir/git/heliog99/u-boot-x676b/uboot-lk-x676b-v{N}.img
  fastboot flash lk_b /workdir/git/heliog99/u-boot-x676b/uboot-lk-x676b-v{N}.img
  ```

### U-Boot загружается, но нет USB
См. [USB_ISSUES.md](USB_ISSUES.md)

### U-Boot загружается, но нет дисплея

**Дисплей работает** (начиная с v183/v187). Если текста нет:

- Проверить, что `CONFIG_VIDEO_MTK_G99=y` в `configs/mt6789_infinix_x676b_defconfig`
- Проверить, что после ручной записи в кадр сделан `dc cvac` — OVL читает
  из DRAM, а `mw.l` пишет только в кэш CPU:
  ```bash
  dc cvac 0xfeabe000 0xa3e000
  ```
- Снять диагностический буфер драйвера (сразу за кадром):
  ```bash
  md.l 0xff4a1400 20
  ```
  magic `0xDEADBEEF` на месте — драйвер отработал полностью
- Подробности: [docs/hardware/display.md](https://github.com/mnnoee/u-boot-mtk-g99/blob/mt6789-stable/docs/hardware/display.md)

### SD-карта не определяется

Текущий блокер проекта. Полный журнал попыток v161…v219 и следующий шаг:
[docs/hardware/sd-card-msdc1.md](https://github.com/mnnoee/u-boot-mtk-g99/blob/mt6789-stable/docs/hardware/sd-card-msdc1.md).

## Ядро Linux

### Kernel panic при загрузке
Проблема с Device Tree. См. [DEVICE_TREE.md](DEVICE_TREE.md)

### Нет UART в ядре
Добавить в bootargs:
```
mtk_printk_ctrl.disable_uart=0
```

## Откат

Если что-то пошло не так, можно откатить к заводской прошивке:
```bash
fastboot flash lk_a /workdir/git/heliog99/u-boot-x676b/lk_a_dump.img
fastboot flash lk_b /workdir/git/heliog99/u-boot-x676b/lk_a_dump.img
fastboot reboot
```
