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

### Что работает
- ✅ PHY probe успешен (`MUSB: PHY probed OK`)
- ✅ MUSB инициализируется
- ✅ Bootmenu загружается (с Fastboot по умолчанию)
- ✅ UFS storage доступен
- ✅ USB stack стабилен
- ✅ Чтение UFS работает

### Проблемы
- ❌ PC не видит USB устройство (pullup toggle)
- ❌ Дисплей не работает (simple-framebuffer removed)

## Быстрый старт

См. [BUILD.md](BUILD.md) для инструкций по сборке.

## Ссылки

- [TOOLS.md](TOOLS.md) - Инструменты и окружение
- [TROUBLESHOOTING.md](TROUBLESHOOTING.md) - Решение проблем
- [USB_ISSUES.md](USB_ISSUES.md) - Проблемы с USB
- [DEVICE_TREE.md](DEVICE_TREE.md) - Device Tree
