# Загрузка Alpine целиком из RAM на Infinix X676B (2026-08-05)

Статус: **артефакты собраны, на железе не проверено.**

Цель — не полноценная система, а проверка «стоковый DT + mainline-ядро стартуют».
Полноценной системы этим путём не получится, см. «Ограничения».

## Почему именно так

Стоковое ядро Alpine `linux-lts 6.18.42` — generic aarch64 под SBC. В нём выключено:

| Опция | Следствие |
|---|---|
| `CONFIG_SCSI_UFS_MEDIATEK`, `CONFIG_PHY_MTK_UFS` | UFS ядро не видит → `root=` невозможен, рутфс только в RAM |
| `CONFIG_SERIAL_8250_MT6577` | штатной консоли на UART нет |
| `CONFIG_USB_MUSB_MEDIATEK`, `CONFIG_PHY_MTK_TPHY` | USB мёртв, гаджета не будет |
| `CONFIG_USB_CONFIGFS_ACM`, `CONFIG_USB_G_SERIAL` | ACM-гаджета нет и в принципе |

Вкомпилено полезное: `SERIAL_EARLYCON=y`, `SERIAL_8250=y` (то есть
`earlycon=uart8250,mmio32,0x11002000` заработает, если припаять UART),
`FRAMEBUFFER_CONSOLE=y`, `DRM_FBDEV_EMULATION=y`, `VT_CONSOLE=y`.
`DRM_SIMPLEDRM=m` — модуль, грузится из initramfs.

Отсюда единственный канал вывода — **экран телефона**.

## Ловушка: vmlinuz-lts это zboot, а не gzip

`mkimage -C gzip -d /boot/vmlinuz-lts` даёт **битый** uImage. Файл не gzip:

```
[0x00] 4d 5a 00 00   "MZ"     PE/EFI
[0x04] 7a 69 6d 67   "zimg"   zboot-контейнер
[0x08] 0000cc08               смещение payload
[0x0c] 00c59839               размер payload
[0x18] "gzip"                 чем сжат payload ВНУТРИ
[0x38] cd 23 82 81            НЕ "ARM\x64" → booti тоже откажет
```

`mkimage` не проверяет данные, он верит флагу `-C`, поэтому заголовок
получается с пометкой gzip поверх PE. Настоящий `Image` достаётся так:

```sh
tail -c +52233 /boot/vmlinuz-lts | gunzip > Image   # 52233 = 0xcc08 + 1
```

Результат: 39 911 424 байта, `magic@0x38 = ARM\x64`, `text_offset=0`,
`image_size=0x2610000`, `flags=0xa` (LE, 4K, bit3 — можно грузить по любому физадресу).

## Карта памяти (из `bdinfo` живого устройства)

```
memory[0]  [0x40000000-0xbfffffff]   = 2 GB
FB base    0xfeabe000  1080x2400x32  ← ВЫШЕ окна памяти, резервировать не нужно
reserved   17 no-map регионов от LK
```

Важно: `0x70000000-0x75ffffff` занят (modem) — под initrd не годится.
`LK_DEFAULT_FB` в драйвере (`0xfeabe000`) верен; `0xfe0c8000` из `CLAUDE.md` — протухло.

Свободная дыра `0xa0000000–0xbc5e4fff`, всё кладём туда:

| Что | Адрес | Размер |
|---|---|---|
| `Image` | `0xa0000000` | 38.1 MB |
| `x676b-boot.dtb` | `0xa3000000` | 188 KB |
| `initramfs-alpine.gz` | `0xa4000000` | 135 MB |

## Артефакты

Каталог `/workdir/git/heliog99/alpine-boot/`, собирается двумя скриптами
в `/workdir/git/heliog99/`:

- `patch_dtb.py` — из `real_base.dtb` (стоковый, `mediatek,MT6789`) делает `x676b-boot.dtb`:
  - `memory` 1 GB → 2 GB;
  - +15 no-map регионов LK в `reserved-memory` (иначе ядро затрёт TEE/modem);
  - `ramoops@0x9fe00000` — 1 MB буфер для pstore (kernel crash logs);
  - `/chosen/framebuffer` = `simple-framebuffer`, `a8r8g8b8`, 1080x2400, stride 4320.
    Прописывается статически, потому что board-код U-Boot `fdt_simplefb_add_node()`
    не вызывает, хотя `CONFIG_FDT_SIMPLEFB=y`;
  - `bootargs`.
- `mk_initramfs.sh` — cpio.gz со всем rootfs `alpinelinux_aarch64/` (без `boot/`,
  `proc`, `sys`, `dev`, `run`). Владелец проставляется ключом `cpio -R 0:0`,
  не `chown` под fakeroot — в каталоге висят хостовые `/proc` и `/sys` от `arch-chroot`.

`/init` внутри initramfs монтирует devtmpfs/proc/sys, грузит `simpledrm`
(до этого `console=tty0` уходит в `dummycon`, в никуда), печатает баннер, dmesg
и уходит в heartbeat. `openrc` намеренно не запускается — он перерисует экран.

## Команды

Заливка (в U-Boot: bootmenu → «Enable USB mass storage», либо `ums 0 scsi 2`).
Раздел 62 `tranfs`, 304 MB — влезает 173 MB артефактов.

```sh
# на ПК, диск найти по lsblk
sudo mkfs.ext4 -O ^orphan_file,^metadata_csum_seed -m 0 -L ubootboot /dev/sdX62
sudo mount /dev/sdX62 /mnt
sudo cp /workdir/git/heliog99/alpine-boot/{Image,x676b-boot.dtb,initramfs-alpine.gz} /mnt/
sync && sudo umount /mnt
```

Загрузка (шелл U-Boot). **Номер раздела в `dev:part` — шестнадцатеричный**,
хотя `scsi part` печатает список в десятичной. `tranfs` = 62 dec = `3e` hex;
если написать `2:62`, U-Boot поймёт это как 0x62 = 98 и скажет
`** Invalid partition 98 **`.

```
setenv initrd_high 0xffffffffffffffff
setenv fdt_high 0xffffffffffffffff
scsi reset
scsi scan
ext4load scsi 2:3e 0xa0000000 /Image
ext4load scsi 2:3e 0xa3000000 /x676b-boot.dtb
ext4load scsi 2:3e 0xa4000000 /initramfs-alpine.gz
booti 0xa0000000 0xa4000000:${filesize} 0xa3000000
```

`initrd_high`/`fdt_high` в `0xffffffffffffffff` — чтобы U-Boot не пытался
перекладывать initrd и DTB внутрь `bootm_size`, а использовал их на месте.
`${filesize}` выставляется последним `ext4load`, то есть равен размеру initramfs.

## Ограничения

- Клавиатуры нет (USB мёртв, кнопки громкости не HID) — консоль **односторонняя**,
  только читать. Шелла не будет.
- Рутфс в tmpfs, изменения не переживают ребут.
- Раннюю загрузку до `modprobe simpledrm` не видно вообще.

Для настоящей системы нужна пересборка `linux-lts` через aports с фрагментом:
`SCSI_UFS_MEDIATEK`, `PHY_MTK_UFS`, `USB_MUSB_MEDIATEK`, `PHY_MTK_TPHY`,
`USB_CONFIGFS_ACM`, `SERIAL_8250_MT6577`, плюс `PSTORE_RAM=y` и `DRM_SIMPLEDRM=y`
вкомпилить, а не модулями.

## Ramoops (pstore)

В DTB добавлена нода `ramoops@0x9fe00000` — 1 MB буфер для kernel crash logs.
Alpine kernel имеет `CONFIG_PSTORE_RAM=m`, модуль загружается в `/init`.

После краша ядра (panic/oops) логи сохраняются в RAM-буфере и доступны после
следующего ребута в `/sys/fs/pstore/`:

```sh
# Список логов после краша
ls -lh /sys/fs/pstore/

# Чтение последнего dmesg
cat /sys/fs/pstore/dmesg-ramoops-0

# Чтение console log
cat /sys/fs/pstore/console-ramoops-0
```

Размеры буферов (по 256 KB каждый):
- `record-size` — основные записи (dmesg/oops)
- `console-size` — console output
- `ftrace-size` — ftrace буфер
- `pmsg-size` — userspace messages (через `/dev/pmsg0`)

**Внимание**: буфер находится в области `0x9fe00000`, которая находится ПЕРЕД
последним `no-map` регионом LK (`misc13` на `0x9fe70000`). Это свободная зона,
не конфликтует с Image/DTB/initramfs во время загрузки (те лежат в `0xa0000000+`).
