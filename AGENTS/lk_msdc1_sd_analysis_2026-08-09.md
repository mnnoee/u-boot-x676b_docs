# LK + SD (MSDC1): выводы агентов-исследователей — 2026-08-09

**Источники:** два explore-агента (анализ lk_unpack_v0/ и docs/AGENTS/+заметки/),
без захода в u-boot-mtk-g99/ и src_linux/. Резюме для следующих сессий.

## 1. Есть ли в LK код для SD-карты (mmc1)? — ДА, полный

В декомпилированном стоковом LK есть готовый код инициализации MSDC1/SD:
- `lk_unpack_v0/lk_lk.bin.c:26585-27421` и `lk.img.c` (~32500-32710)
- Точка входа: `FUN_ffff000050f28344` → `_DAT_ffff000050fedfd0 = 0x708` (ID MSDC1)
  → `FUN_ffff000050f214c8` (выбор базы: host0 → 0x11230000, host1 → 0x11240000,
  top 0x11ef0000) → `FUN_ffff000050f222f0` (инициализация SD-карты)
- msdc_init: `FUN_ffff000050f20668` / `FUN_ffff000050f20868`

### Последовательность LK (по декомпиляции)
1. **Клоки** — таблица конфигурации (lk_lk.bin offset 0xa0a5c; MSDC1:
   `0x0300000001 / 0x0000030303 / 0x00000004 / 0x00000380`); mux msdc30_1_sel =
   индекс 4 (msdcpll_d2) = **192 МГц** (совпадает с Android)
2. **Пины** — `FUN_ffff000050f43ae0`: `0x11D40090 |= 0x01`, `0x11D400C0 |= 0x3F`,
   `0x11D400B0 &= ~0x3F`; GPIO pull: `0x10C00C28` bit14 = 0
3. **Контроллер** — SDC_CFG сброс; PATCH_BIT0(+0x70) |= 0x2000|2; ожидание сброса
   boot mode (+0x78 bit4); PATCH_BIT0 &= ~0x8000; **MSDC_CFG |= 0xF**
   (RST|FIFOCLR|DMASTART) + ожидание готовности; **FIFOCS |= 0x80000000**;
   SDC_CFG |= 0x80000 (SDIO), &= ~0x100000; **PATCH_BIT0/1/2 = 0x403c0007 /
   0xffe64309 / 0x14881803**; PAD_TUNE0/1 = 0
4. **Такты** — init на **100 кГц** (`FUN_ffff000050f207f0(param_9, 0, 100000)`)
   ← ВНИМАНИЕ: LK стартует на 100 кГц, а наш U-Boot на 400 кГц!
5. **Карта** — CMD0 → CMD8 (arg 0x1aa) → CMD55/ACMD41 (OCR|0x40000000, или
   |0x51000000 если S18A/1.8V), 5 ретраев
6. **Вольтаж** — CMD11 (rawcmd 0x8b) через `FUN_ffff000050f21cd0` (5 попыток);
   `FUN_ffff000050f21bf8` — аппаратный переключатель (в этом LK — пустая
   заглушка `FUN_ffff000050f43adc`!); реальный переключатель PAD: `FUN_ffff000050f43ae0`
   (host0 1.8V: `0x11d40090 &= ~0x3f | 1`; 3.3V: `0x11d40090 |= 0x3f`; pad
   `0x11e20120/0x11e20160/0x11e20170`: 1.8V → 0x401/0xbfe/0x401; 3.3V → 0x7ff)
7. **Питание LK НЕ включает** — VMCH/VMC включаются прелоадером (BL2) ДО LK;
   в LK нет обращений к 0x1CD8/0x1CC4/0x1E48/0x1E4C (это подтверждено и
   pwrap-анализом: LK ходит только к ADC 0xD88 и флагу батареи 0x152)

### Кодировка rawcmd в LK (полезно для сверки)
`0x80 | номер_команды`; ACMD: `0x100 | номер`. CMD8=0x88, CMD11=0x8b, CMD55=0x37,
ACMD41=0x129, ACMD51=0x133, CMD3=0x83, CMD6=0x86.

### Заглушки в LK (важно)
`FUN_ffff000050f43adc` (pad switch), `FUN_ffff000050f43bcc` (power),
`FUN_ffff000050f43bd0` — пустые. `FUN_ffff000050f43bd4` — реальный код.

## 2. Доказано ли, что LK РЕАЛЬНО поднимал SD на X676B? — НЕТ

- Ни одного лога/замера успешной работы SD в LK в доках нет (все живые тесты —
  U-Boot или Android).
- Известно: BL2_ext (прелоадер) инициализирует ТОЛЬКО MSDC0 (eMMC)/UFS;
  для MSDC1 — no-op (`msdc1_clock_bus_hang.md`).
- LK не настраивает шины/часы MSDC (это делает BL2): `msdc1_clock_bus_hang.md`.
- Питание SD к моменту LK подано прелоадером (vmc_en=1, vmch_en=1, vmch_volt=0x200 —
  `early_log_mmcdv0_2026-08-09.txt` строки 99-101).

## 3. Ключевые отличия от нашего U-Boot (кандидаты на проверку!)

1. **Частота init: LK = 100 кГц, мы = 400 кГц** ← НЕ ПРОВЕРЕНО на железе!
   (вендорское ядро тоже 400 кГц, но LK-загрузчик конкретно этого устройства
   стартует на 100 кГц)
2. **MSDC_CFG |= 0xF (RST|FIFOCLR|DMASTART) одной записью** и ожидание готовности;
   FIFOCS |= 0x80000000 (аппаратный сброс FIFO)
3. **PATCH_BIT0 |= 0x2000|2 (0x70)**, ожидание сброса boot mode (+0x78 bit4),
   PATCH_BIT0 &= ~0x8000 — шаги, которых нет ни у нас, ни в Android-ядре
4. **SDC_CFG &= ~0x100000** (у нас SDC_CFG=0x03080000 — bit20=1 стоит? 0x3080000:
   bit19=1 (SDIO), bit22=1? проверить 0x100000 = bit20)
5. Пин-конфигурация LK через `0x11D40090/C0/B0` и `0x10C00C28` bit14 (pull) —
   другие адреса, чем наш pinctrl (0x11EA0000)!

## 4. План проверки (по порядку)

1. **Тест частоты 100 кГц** (дешевле всего): попробовать на железе init на
   100 кГц — LK работает именно так (div при 192 МГц = 480)
2. Сверка MSDC_CFG |= 0xF / FIFOCS |= 0x80000000 / PATCH_BIT0 0x2000|2 /
   boot mode +0x78 bit4 / PATCH_BIT0 &= ~0x8000 — консолью до mmc dev 0
3. SDC_CFG bit20 = 0
4. pinctrl LK: 0x11D40090/C0/B0 + 0x10C00C28 bit14
5. Если CRC останется — прошивка стокового LK (u-boot-x676b/lk_a_dump.img)
   со вставленной картой и снятие живого дампа регистров MSDC1 при работе LK
   (но LK сразу бутает Android — нужен ранний перехват/логи)

## 6. Результаты проверки на железе (2026-08-09, v211/v212)

- **v211**: f_min=100000 форс НЕ сработал — MIN_BUS_CLK=200000 (mtk-sd.c:237)
  перебивал (лог: `msdc:f_min=200000`)
- **v212**: форс перенесён после MIN_BUS_CLK — лог: `msdc:f_min=100000`,
  реальный init на 100 кГц (div=480 при 192 МГц)
- **Результат**: `mmc dev 0` → молча fail, INT=0x480 (RSPCRCERR) на 100 кГц
- **ВЫВОД: частота НЕ виновата** — CRC-ошибка на 400/200/100 кГц одинакова.
  Кандидаты №2-5 из раздела 3 остаются: MSDC_CFG |= 0xF единой записью +
  FIFOCS |= 0x80000000; PATCH_BIT0 |= 0x2000|2 + boot mode +0x78 bit4 +
  PATCH_BIT0 &= ~0x8000; SDC_CFG &= ~0x100000 (bit20 — у нас стоит:
  sdc_cfg=0x3080000); pinctrl LK 0x11D40090/C0/B0 + 0x10C00C28 bit14

## 7. Ссылки (доки, где это уже частично расписано)
- docs/AGENTS/mt6789_msdc1_init_analysis.md — ГЛАВНЫЙ: полная инициализация MSDC1/SD из LK
- docs/AGENTS/msdc1_clock_bus_hang.md — LK не трогает шины/часы; BL2_ext: MSDC1 = no-op
- docs/AGENTS/msdc1_sd_power_and_clocks.md — питание включает прелоадер, не LK
- docs/AGENTS/mt6789_msdc1_crc_investigation_2026-08-08.md — PATCH_BIT LK vs вендор
- docs/AGENTS/pwrap_wacs2_protocol_verified.md + LK_PWRAP_INIT_ANALYSIS.md — LK не ходит к PMIC-регуляторам SD
- docs/AGENTS/mt6789_msdc1_clock_tree.md — таблица клоков LK (offset 0xa0a5c)
- docs/AGENTS/pinctrl-msdc1-pins.md — карта пинов GPIO71-76
- заметки/early_log_mmcdv0_2026-08-09.txt — лог probe (питание уже включено)
