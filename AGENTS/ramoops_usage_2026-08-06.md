# Ramoops (pstore) на Infinix X676B — инструкция по использованию

Дата: 2026-08-06

## Что это

**Ramoops** — механизм сохранения kernel logs (panic/oops/console) в выделенной области RAM, которая переживает warm reboot. После краша ядра логи доступны при следующей загрузке через `/sys/fs/pstore/`.

## Конфигурация

### Device Tree

В `patch_dtb.py` добавлена нода в `reserved-memory`:

```dts
ramoops@9fe00000 {
    compatible = "ramoops";
    reg = <0x00 0x9fe00000 0x00 0x00100000>;  /* 1 MB */
    record-size = <0x00040000>;    /* 256 KB для dmesg/oops */
    console-size = <0x00040000>;   /* 256 KB для console */
    ftrace-size = <0x00040000>;    /* 256 KB для ftrace */
    pmsg-size = <0x00040000>;      /* 256 KB для userspace */
};
```

**Адрес `0x9fe00000`** выбран в свободной зоне между `0x9f670000` (конец misc12) и `0x9fe70000` (начало misc13). Не конфликтует с загрузочными артефактами (Image/DTB/initramfs в `0xa0000000+`).

### Kernel

Alpine Linux `linux-lts 6.18.42` имеет:
- `CONFIG_PSTORE=y` — встроено
- `CONFIG_PSTORE_RAM=m` — **модуль** (грузится в `/init`)
- `CONFIG_PSTORE_CONSOLE=n` — выключено (но буфер выделен, можно включить пересборкой)

### Initramfs

В `/init` добавлена загрузка модуля после `simpledrm`:

```sh
modprobe ramoops
```

Модуль автоматически находит ноду `ramoops` в DTB и резервирует память.

## Использование

### После нормальной загрузки

При первой загрузке (или после cold boot) `/sys/fs/pstore/` пуст — crash logs ещё нет:

```sh
ls /sys/fs/pstore/
# (пусто)
```

### После kernel panic/oops

1. Ядро паникует/зависает
2. Перезагружаем устройство (кнопка питания)
3. Загружаемся снова

Логи появятся в `/sys/fs/pstore/`:

```sh
ls -lh /sys/fs/pstore/
# -r--r--r-- 1 root root  10K Aug  6 12:00 dmesg-ramoops-0
# -r--r--r-- 1 root root  15K Aug  6 12:00 console-ramoops-0

# Читаем последний dmesg
cat /sys/fs/pstore/dmesg-ramoops-0

# Читаем console output
cat /sys/fs/pstore/console-ramoops-0
```

### Типы записей

| Файл | Содержимое |
|------|------------|
| `dmesg-ramoops-N` | Kernel ring buffer до момента краша |
| `console-ramoops-N` | Console output (если `CONFIG_PSTORE_CONSOLE=y`) |
| `ftrace-ramoops-N` | Ftrace буфер (если `CONFIG_PSTORE_FTRACE=y`) |
| `pmsg-ramoops-N` | Userspace messages через `/dev/pmsg0` |

Номер `N` — счётчик записей (0, 1, 2...), инкрементируется при каждом сохранении.

### Ручная запись в pmsg

Userspace может писать в `/dev/pmsg0`, эти сообщения переживут reboot:

```sh
echo "test message before reboot" > /dev/pmsg0
# reboot
cat /sys/fs/pstore/pmsg-ramoops-0
```

### Очистка логов

Удаление файла из `/sys/fs/pstore/` очищает соответствующий буфер в RAM:

```sh
rm /sys/fs/pstore/dmesg-ramoops-0
```

## Ограничения

1. **Только warm reboot** — после полного выключения питания (cold boot) RAM обнуляется, логи теряются
2. **Console buffer пуст** — `CONFIG_PSTORE_CONSOLE=n`, только dmesg и pmsg работают
3. **Размер буфера** — 1 MB делится на 4 типа × 256 KB, старые записи перезаписываются
4. **Нет сжатия в модуле** — `CONFIG_PSTORE_COMPRESS=y` работает, но `ramoops.ko` не использует (нужно проверить)

## Отладка

Проверить, что ramoops загрузился:

```sh
dmesg | grep ramoops
# [    2.345678] ramoops: using 0x100000@0x9fe00000, ecc: 0
```

Проверить, что pstore смонтирован:

```sh
mount | grep pstore
# pstore on /sys/fs/pstore type pstore (rw,nosuid,nodev,noexec,relatime)
```

Посмотреть параметры модуля:

```sh
modinfo ramoops
cat /sys/module/ramoops/parameters/*
```

## Как вызвать kernel panic для теста

**ВНИМАНИЕ**: это уронит систему!

```sh
# Только для теста в initramfs-окружении (не потеряете данных)
echo c > /proc/sysrq-trigger
```

После reboot логи появятся в `/sys/fs/pstore/dmesg-ramoops-0`.

## Артефакты

После обновления нужно перезалить:
- `x676b-boot.dtb` — обновлён, содержит ramoops ноду
- `initramfs-alpine.gz` — обновлён, `/init` грузит `ramoops.ko`

Команды те же (см. `alpine_ram_boot_2026-08-05.md`):

```sh
sudo cp /workdir/git/heliog99/alpine-boot/{Image,x676b-boot.dtb,initramfs-alpine.gz} /mnt/
```

## Ссылки

- [Kernel docs: Ramoops](https://www.kernel.org/doc/html/latest/admin-guide/ramoops.html)
- [Kernel docs: pstore](https://www.kernel.org/doc/html/latest/admin-guide/pstore-blk.html)
- Device Tree binding: `Documentation/devicetree/bindings/reserved-memory/ramoops.yaml`
