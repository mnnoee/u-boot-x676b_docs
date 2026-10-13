# MT6789 MSDC1 SD Card Debug Session — 2026-08-06

## Цель
Исправить инициализацию SD карты в U-Boot на MT6789 MSDC1 (0x11240000). Проблема: CMD8 и CMD55 падают с timeout (ret=-5), карта не определяется.

## Что обнаружено

### 1. PAD_TUNE регистры имеют маски записи
**Адреса:**
- PAD_TUNE0: 0x1124009c
- PAD_TUNE1: 0x112400a0

**Целевые значения из LK:**
- PAD_TUNE0: 0x403c0007
- PAD_TUNE1: 0xffe64309

**Проблема:** При попытке записать эти значения через `writel()`, читаются совершенно другие значения.

**Тестирование маски (запись 0xFFFFFFFF):**
```
PAD_TUNE0: write 0xFFFFFFFF → read 0x00033306
PAD_TUNE1: write 0xFFFFFFFF → read 0x0000ffff
```

**Правильные значения для записи (после маскирования):**
```
PAD_TUNE0: 0x403c0007 & 0x00033306 = 0x00030006 ✓ записалось
PAD_TUNE1: 0xffe64309 & 0x0000ffff = 0x00004309 ✓ записалось
```

**Вывод:** Регистры имеют hardware write mask — только определённые биты записываемы. Код в U-Boot пытается писать полные значения, но записываются только младшие биты.

### 2. MSDC_CFG неправильно конфигурируется
**Адрес:** 0x11240000

**Текущее состояние в U-Boot:**
```
MSDC_CFG = 0x0200119b
  Bit 1 (CKPDN) = 1  ← clock power down ВКЛЮЧЁН (плохо!)
  Bit [15:8] (CKDIV) = 0x01
  Bit 7 (CKSTB) = 1
  Bit 3 (PIO) = 1
```

**Рабочее значение в LK:**
```
MSDC_CFG = 0x02000099
  Bit 1 (CKPDN) = 0  ← clock включён
  Bit [15:8] (CKDIV) = 0x00
  Bit 7 (CKSTB) = 1
  Bit 3 (PIO) = 1
```

**Тестирование:** Ручная запись 0x02000099 через `mw.l 0x11240000 0x02000099` изменила поведение:
- Было: CMD8 timeout (ret=-5)
- Стало: "TX/RX FIFO non-empty before start of IO. Reset" (прогресс, но всё ещё не работает)

**Проблема:** `msdc_set_mclk()` перезаписывает MSDC_CFG при каждом вызове, уничтожая ручные изменения.

### 3. builtin-cd не является корневой причиной
**Что пробовал:**
- v202: добавил `broken-cd;` в DT → не помогло
- v203: заменил на `builtin-cd = <0>;` → не помогло

**MSDC_PS регистр:**
```
MSDC_PS = 0x00000002
  Bit 1 (CDSTS) = 1  ← hardware считает, что карта не вставлена
```

Но карта физически вставлена. Hardware card detect не работает, НО это не мешает инициализации — проблема в PAD_TUNE и MSDC_CFG.

### 4. Источник тактирования уже исправлен
В probe логе видно:
```
[EARLY] msdc:base=0x11240000 src_clk=296999893
[EARLY] msdc1:forced_26mhz
[EARLY] msdc:f_min=400000 f_max=26000000
```

Workaround на строках 1770-1773 mtk-sd.c работает правильно. Это не проблема.

## Что сделано

### Версии U-Boot
- **v198**: Первая попытка MT6789 PAD_TUNE кода, но проверка `if (!host->top_base)` блокировала выполнение
- **v199**: Переместил проверку MT6789 перед top_base, код начал выполняться, но регистры не менялись
- **v200**: Писал напрямую в `&host->base->pad_tune0` вместо `tune_reg`
- **v201**: Добавил debug логи "pad_tune0:before" / "after" — подтвердил, что запись не работает
- **v202**: Добавил `broken-cd;` в DT
- **v203**: Заменил на `builtin-cd = <0>;`

### Тестирование через uart_cmd.py
Все эксперименты проводились на реальном железе через `/dev/ttyACM0`:
1. Проверка регистров через `md.l`
2. Ручная запись через `mw.l`
3. Тестирование write mask (запись 0xFFFFFFFF)
4. Подтверждение, что маскированные значения записываются
5. Попытка изменить MSDC_CFG → временный эффект до следующего `mmc rescan`

## Корневая причина

**Двойная проблема:**

1. **PAD_TUNE записывается неправильно** — код пишет полные 32-битные значения, но регистры принимают только маскированные биты
   - Текущий код (lines 1524-1527):
     ```c
     writel(0x403c0007, &host->base->pad_tune0);  // ✗ записывается только 0x7
     writel(0xffe64309, &host->base->pad_tune1);  // ✗ записывается 0x0
     ```
   - Нужно:
     ```c
     writel(0x00030006, &host->base->pad_tune0);  // ✓
     writel(0x00004309, &host->base->pad_tune1);  // ✓
     ```

2. **MSDC_CFG перезаписывается в msdc_set_mclk()** — функция устанавливает CKPDN=1 (clock power down) и неправильный CKDIV
   - Нужно найти, где именно это происходит, и исправить для MT6789 MSDC1

## План исправления

### Шаг 1: Исправить PAD_TUNE значения
Заменить в `msdc_init_hw()` (строки ~1524-1527):
```c
/* MT6789 MSDC1: Use working PAD_TUNE values from LK (masked for hardware) */
if (host->phys_base == 0x11240000) {
    if (host->top_base) {
        writel(0, &host->top_base->emmc_top_control);
        writel(0, &host->top_base->emmc_top_cmd);
    }
    /* Hardware write masks: PAD_TUNE0=0x00033306, PAD_TUNE1=0x0000ffff
     * LK values: 0x403c0007, 0xffe64309
     * Masked: 0x00030006, 0x00004309
     */
    writel(0x00030006, &host->base->pad_tune0);
    writel(0x00004309, &host->base->pad_tune1);
}
```

### Шаг 2: Исправить MSDC_CFG в msdc_set_mclk()
**TODO:** Прочитать код `msdc_set_mclk()` (начиная с ~line 870) и найти, где устанавливается:
- CKPDN (bit 1) — должен быть 0 (clock enabled)
- CKDIV (bits [15:8]) — должен быть 0 для 400kHz init

Вероятно, нужно добавить MT6789 MSDC1 workaround в эту функцию.

### Шаг 3: Проверить def_tune_para
Функция `msdc_ops_set_ios()` (lines ~960-968) восстанавливает значения из `def_tune_para`, которые инициализируются чтением текущих регистров (lines ~1668-1669).

**Возможная проблема:** Если PAD_TUNE записывается в init_hw(), но потом восстанавливается старое значение из def_tune_para.

**Решение:** После записи PAD_TUNE в init_hw(), обновить def_tune_para:
```c
writel(0x00030006, &host->base->pad_tune0);
writel(0x00004309, &host->base->pad_tune1);
host->def_tune_para.pad_tune = readl(&host->base->pad_tune0);
```

## Следующие действия

1. Прочитать код `msdc_set_mclk()` и найти, где устанавливается CKPDN и CKDIV
2. Добавить MT6789 MSDC1 workaround для правильных значений MSDC_CFG
3. Исправить PAD_TUNE значения на маскированные
4. Обновить def_tune_para после записи PAD_TUNE
5. Собрать v204 и протестировать на железе

## Файлы для изменения

- `/workdir/git/heliog99/u-boot-mtk-g99/drivers/mmc/mtk-sd.c`
  - Lines 1517-1530: `msdc_init_hw()` — исправить PAD_TUNE значения
  - Lines ~870+: `msdc_set_mclk()` — добавить MT6789 MSDC1 workaround для MSDC_CFG
  - Lines ~960-968: `msdc_ops_set_ios()` — проверить восстановление def_tune_para
  - Lines ~1668-1669: проверить инициализацию def_tune_para

## Логи для справки

### Early log с текущей версией (v203)
```
[EARLY] msdc:base=0x11240000 src_clk=296999893
[EARLY] msdc1:forced_26mhz
[EARLY] msdc:f_min=400000 f_max=26000000
[EARLY] msdc:init_hw:entry
[EARLY] msdc:pad_tune0:before=0x0
[EARLY] msdc:pad_tune0:after=0x7        ← записали 0x403c0007, прочитали 0x7
[EARLY] msdc:pad_tune1:before=0x0
[EARLY] msdc:pad_tune1:after=0x0        ← записали 0xffe64309, прочитали 0x0
[EARLY] msdc:init_hw:mt6789_pad_tune
[EARLY] msdc_set_mclk: hz=400000 timing=0 src_clk=26000000
[EARLY] msdc_send_cmd: CMD8 arg=0x1aa
[EARLY] msdc_send_cmd: CMD8 failed ret=-5
[EARLY] msdc_send_cmd: CMD55 arg=0x0
[EARLY] msdc_send_cmd: CMD55 failed ret=-5
```

### Регистры после boot
```
MSDC_CFG (0x11240000) = 0x0200119b  ← CKPDN=1 (плохо)
MSDC_PS (0x11240008) = 0x00000002   ← CDSTS=1 (карта не определена HW)
PAD_TUNE0 (0x1124009c) = 0x00000004 ← не наше значение
PAD_TUNE1 (0x112400a0) = 0x00000000 ← не наше значение
```

### Регистры после ручной записи
```bash
# PAD_TUNE
mw.l 0x1124009c 0x00030006; md.l 0x1124009c 1
→ 1124009c: 00030006 ✓

mw.l 0x112400a0 0x00004309; md.l 0x112400a0 1
→ 112400a0: 00004309 ✓

# MSDC_CFG
mw.l 0x11240000 0x02000099; md.l 0x11240000 1
→ 11240000: 02000099 ✓

# mmc rescan после этого
→ drivers/mmc/mtk-sd.c:600- msdc_start_command() TX/RX FIFO non-empty before start of IO. Reset
→ CMD8 всё ещё падает, но ошибка другая (FIFO вместо timeout)
```

## Ссылки на предыдущую работу

- `docs/AGENTS/mt6789_msdc1_live_diagnostics_2026-08-06.md` — диагностика MSDCPLL bypass mode
- `u-boot-x676b/PROJECT_STATE.md` — общее состояние проекта
- Версии U-Boot: `u-boot-x676b/uboot-lk-x676b-v{198-203}.img`

## Статус
🔴 **В работе** — найдена корневая причина (PAD_TUNE маски + MSDC_CFG CKPDN), готов план исправления
