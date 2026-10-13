# U-Boot for Infinix X676B (MT6789)

## Устройство
- **Модель**: Infinix X676B (NOTE 12 PRO)
- **SoC**: MediaTek MT6789 (Helio G99)
- **CPU**: 6x Cortex-A55 @ 2.0GHz + 2x Cortex-A76 @ 2.2GHz
- **RAM**: 8 GB (4+4 banks)
- **Storage**: SK Hynix H9QT1G6DN6X132 UFS 2.2, 238.3 GB
- **Kernel**: Linux 5.10.149 (GKI)
- **Android**: 12 (API 31)
- **Разметка**: A/B seamless updates

## Структура проекта

```
/workdir/git/heliog99/
├── u-boot-mtk-g99/          # Upstream U-Boot (основной репозиторий)
├── u-boot-x676b/            # Рабочий каталог с патчами и логами
├── src_linux/
│   ├── linux/               # Linux kernel mainline
│   ├── mtk_g99-mainline/    # PostmarketOS port
│   └── u-boot-mt6789/       # JamiKettunen U-Boot fork
├── dtbo/                    # Device Tree для X676B
├── firmware/                # Заводская прошивка
├── mtkclient/               # Инструмент для прошивки
├── backups/                 # Бекапы партиций
└── заметки/                 # Рабочие заметки
```

## Текущий статус

> Состояние по коммиту `v219` от 2026-08-10.
> Актуальная документация — [`u-boot-mtk-g99/docs/`](https://github.com/mnnoee/u-boot-mtk-g99/tree/mt6789-stable/docs).

### Что работает

- ✅ **Дисплей** — текст U-Boot выводится на AMOLED 1080×2400 (v183/v187).
      Собственный драйвер `drivers/video/mediatek/mtk_disp_g99.c`:
      конвейер OVL0 → RDMA0 → DITHER0 → DSI0, панель ILI7835F.
      Подробности: [docs/hardware/display.md](https://github.com/mnnoee/u-boot-mtk-g99/blob/mt6789-stable/docs/hardware/display.md)
- ✅ **USB fastboot** — устройство перечисляется на ПК по USB
- ✅ **USB CDC ACM** — консоль U-Boot по USB, UART-кабель не нужен
- ✅ **UMS / fastboot block по UFS** — память телефона видна на ПК (`ums 0 scsi 2`)
- ✅ **UFS storage** — чтение и запись работают
- ✅ **PHY probe** — `MUSB: PHY probed OK`, контроллер поднимается
- ✅ **Bootmenu** — с fastboot по умолчанию, навигация с зацикливанием
- ✅ **Serial по UART** — 921600 baud, testpoints рядом с fingerprint connector
- ✅ **Кнопки** — Volume Up / Down работают, управляют bootmenu.
      Самописный драйвер `drivers/button/button-mtk-kp.c` (в форке ellyq такого
      драйвера не было), узлы `volume-up` / `volume-down` в DTS.

### Что не работает

- ❌ **SD-карта (MSDC1)** — контроллер поднимается, но карта не отвечает:
      `CMD8` → `INT=0x400`, `RSP=0`. Журнал всех попыток v161…v219 —
      [docs/hardware/sd-card-msdc1.md](https://github.com/mnnoee/u-boot-mtk-g99/blob/mt6789-stable/docs/hardware/sd-card-msdc1.md).
      **Это текущий блокер проекта.**
- ❌ **Загрузка Android** — ядро стартует и паникует:
      `Attempted to kill init!` (не тот DTB)

## Репозитории

| | |
|---|---|
| **U-Boot (исходники)** | [github.com/mnnoee/u-boot-mtk-g99](https://github.com/mnnoee/u-boot-mtk-g99) — форк `ellyq`, ветка `mt6789-stable` |
| **Документация (этот репозиторий)** | [github.com/mnnoee/u-boot-x676b_docs](https://github.com/mnnoee/u-boot-x676b_docs) |
| **Образы и скрипты** | [github.com/mnnoee/u-boot-x676b](https://github.com/mnnoee/u-boot-x676b) |
| **Device Tree** | [github.com/mnnoee/u-boot-x676b_dtbo](https://github.com/mnnoee/u-boot-x676b_dtbo) |

## Быстрый старт

См. [BUILD.md](BUILD.md) для инструкций по сборке.

## Ссылки

- [TOOLS.md](TOOLS.md) - Инструменты и окружение
- [TROUBLESHOOTING.md](TROUBLESHOOTING.md) - Решение проблем
- [USB_ISSUES.md](USB_ISSUES.md) - Проблемы с USB
- [DEVICE_TREE.md](DEVICE_TREE.md) - Device Tree
