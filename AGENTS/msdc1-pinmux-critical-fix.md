# MSDC1 Pin Mode Registers — Критическая ошибка и исправление

## Проблема
Pin mode registers для MSDC1 (GPIO 71-76) были по ошибке записаны в **неправильные адреса**.
Драйвер `pinctrl-mt6789.ko` не мог применить DTS pinctrl (нет group definitions в U-Boot).
В результате CMD/CLK/DAT линии оставались в GPIO mode (function 0) вместо MSDC1 (function 1).

## Ключевой инсайт: 8 пинов на 1 регистр, НЕ 1 пин на 1 регистр!

**НЕПРАВИЛЬНО** (наши первые попытки):
```
mw.l 0x10005770 0x1   # GPIO 71 mode — НЕПРАВИЛЬНЫЙ АДРЕС
mw.l 0x10005780 0x1   # GPIO 72 mode — НЕПРАВИЛЬНЫЙ АДРЕС
```
Формула `0x10005300 + N*0x10` — **неверна** для MT6789! Каждый 32-bit регистр содержит mode для **8 пинов** (4 бита на пин).

**ПРАВИЛЬНО**:
```
mw.l 0x10005380 0x11051100   # GPIO 71: bits [31:28] = 0x1 (MSDC1 CLK)
mw.l 0x10005390 0x11111111   # GPIO 72-76: bits [3:0]..[19:16] = 0x1 (MSDC1 CMD/DAT0-3)
```

## Правильная формула

```
register_addr = base + field_offset + (pin_within_range / 8) * 4
bit_position = (pin_within_range % 8) * 4
```

Где `base` и `field_offset` берутся из `mt6789_pin_mode_range[]` таблицы.

## Pin mode registers для MSDC1 (GPIO 71-76)

| GPIO | Функция | Mode Register | Bit Field |
|------|---------|---------------|-----------|
| 71 (CLK) | 1 | 0x10005380 | [31:28] |
| 72 (CMD) | 1 | 0x10005390 | [3:0] |
| 73 (DAT0) | 1 | 0x10005390 | [7:4] |
| 74 (DAT1) | 1 | 0x10005390 | [11:8] |
| 75 (DAT2) | 1 | 0x10005390 | [15:12] |
| 76 (DAT3) | 1 | 0x10005390 | [19:16] |

**Важно**:写入 через `mw.l` перезаписывает ВСЕ 8 пинов в регистре. Используй read-modify-write!

## I/O controller bank (iocfg_rt)

| Register | Address | Pins |
|----------|---------|------|
| IES | 0x11EA0080 | bits [1]-[6] for GPIO 71-76 |
| SMT | 0x11EA0140 | bits [16]-[21] |
| PUPD | 0x11EA00B0 | bits [0]-[5] |
| R0 | 0x11EA00E0 | bits [0]-[5] |
| R1 | 0x11EA00F0 | bits [0]-[5] |
| DRV | 0x11EA0030 | 3 bits each |

## Что было сделано

1. `mw.l 0x10005380 0x11051100` — set GPIO 71 to function 1 (MSDC1 CLK)
2. `mw.l 0x10005390 0x11111111` — set GPIO 72-76 to function 1 (MSDC1 CMD/DAT0-3)
3. `mw.l 0x11EA0080 0x7E` — enable IES for pins 71-76
4. `mw.l 0x11EA00B0 0x3F` — set PUPD for pins 71-76
5. `mw.l 0x11EA00E0 0x3F` — set R0 for pins 71-76
6. `mw.l 0x11EA00F0 0x00` — clear R1 for pins 71-76
7. `mw.l 0x11EA0140 0x3F0000` — enable SMT for pins 71-76

## Результат
После правильной настройки pin mode registers:
- `mmc rescan` проходит **без ошибок** (было "Card did not respond to voltage select")
- `mmc info` не выводит данных и не выводит ошибку
- SDC_STATUS = 0x00000145 (BUSY + CMD_INT + DAT0_INT + CMD_RDY)

## Текущая проблема
Контроллер отправляет команды, но ответ не читается. Возможные причины:
1. Source clock не настроен правильно (MSDC30_1_SEL mux)
2. Clock divider слишком быстрый/медленный
3. Timing issues с SD card

## Что нужно сделать дальше
1. Проверить source clock frequency — возможно F26M не настроен как parent
2. Проверить SDC_CFG clock divider
3. Попробовать отправить CMD0 вручную через регистры
4. Добавить debug output для SDC_STATUS после каждой команды

## Важные адреса для отладки
- MSDC_CFG: 0x11240000 (clock divider)
- SDC_CFG: 0x11240050
- SDC_STATUS: 0x11240054
- SDC_CMD: 0x11240064
- SDC_CMD_ARG: 0x11240068
- SDC_RESP0: 0x11240080
