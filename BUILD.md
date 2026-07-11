# Сборка U-Boot

## Требования

- Кросс-компилятор: `/usr/bin/aarch64-linux-gnu-gcc`
- Python venv: `/workdir/git/heliog99/mtkclient/venv/bin/python3`
- Make

## Конфигурация

```bash
cd /workdir/git/heliog99/u-boot-mtk-g99
make CROSS_COMPILE=aarch64-linux-gnu- mt6789_infinix_x676b_defconfig
```

## Сборка

```bash
make CROSS_COMPILE=aarch64-linux-gnu- -j$(nproc)
```

## Упаковка в LK формат

```bash
/workdir/git/heliog99/mtkclient/venv/bin/python3 gimme-uboot.py \
  /workdir/git/heliog99/u-boot-x676b/lk_a_dump.img \
  u-boot.bin \
  /workdir/git/heliog99/u-boot-x676b/uboot-lk-x676b-v{N}.img
```

## Прошивка

### Через fastboot (если U-Boot работает)
```bash
adb reboot bootloader
fastboot flash lk_a /workdir/git/heliog99/u-boot-x676b/uboot-lk-x676b-v{N}.img
fastboot flash lk_b /workdir/git/heliog99/u-boot-x676b/uboot-lk-x676b-v{N}.img
fastboot reboot
```

### Через mtkclient (DA mode)
```bash
cd /workdir/git/heliog99/mtkclient
python mtk w lk_a ../u-boot-x676b/uboot-lk-x676b-v{N}.img
python mtk w lk_b ../u-boot-x676b/uboot-lk-x676b-v{N}.img
```

## Откат

```bash
fastboot flash lk_a /workdir/git/heliog99/u-boot-x676b/lk_a_dump.img
fastboot flash lk_b /workdir/git/heliog99/u-boot-x676b/lk_a_dump.img
fastboot reboot
```

## Очистка

**ВАЖНО** - не оставлять мусор после сборки!
```bash
rm -rf build .config u-boot*
```

## UART Log

```bash
# Захват лога
picocom -b 921600 /dev/ttyUSB0 --logfile /workdir/git/heliog99/u-boot-x676b/uart-log-v{N}.txt

# Чтение лога
strings /workdir/git/heliog99/u-boot-x676b/uart-log-v{N}.txt
```

## LSP (compile_commands.json)

### U-Boot
```bash
cd /workdir/git/heliog99/u-boot-mtk-g99
make CROSS_COMPILE=aarch64-linux-gnu- mt6789_infinix_x676b_defconfig
python3 scripts/gen_compile_commands.py
```

### Linux kernel
```bash
cd /workdir/git/heliog99/src_linux/linux
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- defconfig
/workdir/git/heliog99/mtkclient/venv/bin/compiledb -o compile_commands.json
```
