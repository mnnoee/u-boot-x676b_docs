# MT6789 MSDC1 — расследование CRC-ошибок ответов (2026-08-08)

## Цель
SD-карта (mmc1, 0x11240000) не инициализируется в U-Boot: CMD8/CMD55 отвечают,
но с CRC-ошибкой. Найти правильную конфигурацию контроллера по фактам с
работающего устройства (Android) и декомпилированного LK.

## Симптомы (U-Boot, fresh log после rescan)
- CMD0: events=0x180 (CMDRDY+0x80), ret=0 — OK (команда без ответа)
- CMD8/CMD55: events=**0x480**, ret=-5 (EIO), r0=0x0, CMDRDY (bit8) НЕ приходит
- Такты init: 400 кГц = 192 МГц /4 /120 (div=120, sclk=400000) — корректно после форса 192МГц
- Питание: VMCH=0x200 (2.9V), VMC=0x400 (1.8V), оба EN=1 — совпадает с Android
- Карта физически есть: ps=0x10002 (DAT0=1, CDSTS=1)

## Расшифровка событий INT (0x480 = биты 7 + 10)
Источник: `src_linux/g99_android_kernel/android_kernel_gigaset_mt6789/drivers/mmc/host/mtk-mmc.h`
```
bit 0 MMCIRQ, 1 CDSC, 3 ACMDRDY, 4 ACMDTMO, 5 ACMDCRCERR, 6 DMAQ_EMPTY,
bit 7 SDIOIRQ, 8 CMDRDY, 9 CMDTMO, 10 RSPCRCERR, 11 CSTA,
12 XFER_COMPL, 13 DXFER_DONE, 14 DATTMO, 15 DATCRCERR, 16 ACMD19_DONE ...
```
→ 0x480 = **SDIOIRQ + RSPCRCERR**: ответ карты приходит, но CRC битый.
При CRC-ошибке CMDRDY не выставляется (аппаратно), драйвер корректно даёт -EIO.

## PATCH_BIT2: RESPWAIT — (0x3 << 2), НЕ <<16!
`mtk-mmc.h`: `MSDC_PB2_RESPWAIT (0x3 << 2)`, `MSDC_PB2_RESPSTSENSEL (0x7 << 16)`,
`MSDC_PB2_CRCSTSENSEL (0x7 << 29)`, `MSDC_PATCH_BIT2_CFGCRCSTS (0x1 << 28)`,
`MSDC_PB2_SUPPORT_64G (0x1 << 1)`. (В u-boot mtk-sd.c — то же: 0x0c/биты 3:2.)

## Что пишет LK (декомпиляция)
Источник: `lk_unpack_v0/lk.img.c` строки ~32510-32710 (инициализация MSDC,
функция выбирает контроллер: `*param_9 == 0 → 0x11230000 (MSDC0), иначе 0x11240000 (MSDC1)`):
```
+0xb0 (PATCH_BIT0) = 0x403c0007
+0xb4 (PATCH_BIT1) = 0xffe64309
+0xb8 (PATCH_BIT2) = 0x14881803, затем |= 2 → 0x14881805
+0xf0/+0xf4 (PAD_TUNE0/1) = 0  ← LK ОБНУЛЯЕТ (не ставит задержки!)
+0xf8..0x104 (dat_rd_dly[0..3]) = 0
+0x70/+0x78 (emmc_cfg0/emmc_sts) = 0
+0x4 (IOCON) = 0 (+RSPL/DSPL из параметра)
+0x30 (SDC_CFG) |= 0x80000 (SDIO bit19!), &= ~0x100000 (SDIOIDE bit20)
+0x14 (FIFOCS) |= 0x80000000 (CLR)
+0x228 (SDC_FIFO_CFG) &= ~0x1000000, &= ~0x2000000
```
ВАЖНО: значения 0x403c0007/0xffe64309 — это PATCH_BIT0/B1, а НЕ PAD_TUNE0/1!
Раньше (v191) мы ошибочно записали их в 0xf0/0xf4 — запись в 0xf0 даёт только
низкие биты (0x7) — старшие 24 бита не пишутся (write-mask).
Эксперимент с LK-значениями (0x403c0007/0xffe64309/0x14881805) НЕ помог: CRC та же.

## Вендорный драйвер — эталон (mtk-mmc.c)
Источник: `src_linux/g99_android_kernel/android_kernel_gigaset_mt6789/drivers/mmc/host/mtk-mmc.c`
Это ЕДИНСТВЕННЫЙ драйвер с `compatible = "mediatek,mt6789-mmc"` в имеющихся
исходниках — именно он работает на X676B (в остальных деревьях mtk-sd.c нет mt6789;
gki-kernel вообще не знает MT6789).

### mt6789_compat (строки 160-175)
clk_div_bits=12, async_fifo=true, data_tune=true, busy_check=true,
stop_clk_fix=true, **enhance_rx=true**, support_64g=true, need_gate_cg=true,
set_crypto_enable_in_sw=true.

### init_hw (строки 1450-1565) — последовательность для MT6789
1. `reset_control_assert/deassert` — аппаратный reset через reset-controller (У НАС НЕТ!)
2. MSDC_CFG MODE, msdc_reset_hw
3. INTEN=0, INT clear
4. TOP_CONTROL=0, TOP_CMD=0 (0x11EF0000)
5. IOCON=0, DDLSEL=0
6. `PATCH_BIT0 = 0x403c0046` + `CKGEN_MSDC_DLY_SEL (0x1f<<10) = 1` → **0x403c0446**
7. `PATCH_BIT1 = 0xffff4089`; stop_clk_fix: STOP_DLY=3 (→0x300); busy_check: clr bit7 → **0xffff4309**
8. EMMC50_CFG0 |= CFCSTS_SEL (bit4)
9. async_fifo: RESPWAIT=3 (→0xC)
10. **enhance_rx=true: TOP_CONTROL |= SDC_RX_ENH_EN (bit15)** ← мы не ставили!
    (иначе: RESPSTSENSEL=2, CRCSTSENSEL=2)
11. CFGRESP (bit15) = 0; **CFGCRCSTS (bit28) = 1** ← мы сбрасывали!
12. SUPPORT_64G (bit1) = 1
13. SDC_FIFO_CFG: clr WRVALIDSEL/RDVALIDSEL

### Наши ошибки в u-boot mtk-sd.c (v191)
- dev_comp = mt8183_compat (нет "mediatek,mt6789-mmc" в msdc_ids!) —
  `enhance_rx` отсутствует → false. MT6789 требует enhance_rx!
- `SDC_RX_ENH_EN` (top 0x11EF0000 bit15) не включали
- `CFGCRCSTS` (patch_bit2 bit28) сбрасывали (async_fifo_crcsts=false)
- `CKGEN_MSDC_DLY_SEL` (patch_bit0 биты14:10) = 0, вендор ставит 1
- Нет аппаратного reset через reset-controller (вендор делает!)
- Нет SUPPORT_64G

### Эксперимент (консоль U-Boot, без пересборки)
```
mw.l 0x112400b0 0x403c0446
mw.l 0x112400b4 0xffff4309
mw.l 0x112400b8 0x1000000e   ; RESPWAIT=3, CFGCRCSTS, SUPPORT_64G
mw.l 0x11ef0000 0x8000       ; SDC_RX_ENH_EN
```
РЕЗУЛЬТАТ: CMD8/CMD55 — та же ошибка 0x480 (RSPCRCERR). Вендорные значения
patch_bit/top НЕ решают проблему.

## Проверено и ОТКЛОНЕНО
- **SDIO-бит (sdc_cfg bit19)**: вендор его ВКЛЮЧАЕТ сам ("it's must otherwise
  sdio cmd5 failed", mtk-mmc.c:1726) — не причина
- **PAD_TUNE0/1 задержки**: LK обнуляет; вендор при enhance_rx не тюнится на init
- **Питание 2.9/1.8V**: совпадает с Android (docs/AGENTS/mt6789_msdc1_android_live_state_2026-08-08.md)
- **Pinctrl GPIO71-76**: mode 1, совпадает с Android (тот же док)
- **Герцовка**: 400кГц init теперь корректна (div=120 от 192МГц, sclk=400000)
- **Аппаратный reset через reset-controller — ОПРОВЕРГНУТ**:
  - Вендор получает reset через `devm_reset_control_get_optional_exclusive`
    (mtk-mmc.c:2861) и делает assert/deassert ТОЛЬКО если `host->reset` ненулевой
    (mtk-mmc.c:1457-1461)
  - В DT X676B у mmc@11240000 НЕТ свойства `resets` ни в Android
    (main_tree.dts:4295-4324), ни в нашем u-boot dtsi (mt6789.dtsi:399-419),
    ни в dtbo-оверлеях → host->reset = NULL → вендор на X676B аппаратный
    reset НЕ выполняет (наш код идентичен вендорному здесь)
  - Отдельного INFRA reset-бита для MSDC1 на MT6789 не существует
    (проверено по вендорным spm_reg.h: биты MSDC1 есть только для SPM-запросов
    клоков, не для reset)
  - Наш софт-сброс MSDC_CFG RST (msdc_reset_hw) — то же, что у вендора
- **PATCH_BIT/ENH_RX (вендорные значения)**: PATCH_BIT0=0x403c0446,
  PATCH_BIT1=0xffff4309, PATCH_BIT2=0x1000000e, TOP_CONTROL|=SDC_RX_ENH_EN(0x8000)
  — установлены в консоли, CRC та же (0x480) → не причина
- **LK-значения PATCH_BIT** (0x403c0007/0xffe64309/0x14881805): не причина

## Оставшиеся гипотезы
1. Сигнал CMD-линии: осциллограф (пин 71) — смотреть форму ответа карты
2. Сверить INT-регистр во время команды (после CMD55: int=0x480 — SDIOIRQ
   поднимается ВМЕСТЕ с RSPCRCERR: возможно, требуется маска/обработка SDIOIRQ)
3. **RSPL (iocon bit1)** — сэмплирование ответа на rising-кромке: эксперимент
   начат (mw.l 0x11240004 2 + mmc rescan), НЕ ЗАВЕРШЁН — сессия зависла на
   uart-скрипте (таймаут 240s), результат неизвестен, требуется повтор
4. **CFGRESP (patch_bit2 bit15, 0x8000)** — принудительное пересэмплирование
   ответа по CRCSTS (старый MTK-фолбек при битых ответах; LK/вендор держат 0,
   но на нашем железе ответы реально битые) — НЕ ПРОБОВАЛИ

## Файлы, использованные при расследовании
- `src_linux/g99_android_kernel/android_kernel_gigaset_mt6789/drivers/mmc/host/mtk-mmc.c`
  (вендорный драйвер MT6789 — ЭТАЛОН: mt6789_compat, msdc_init_hw;
  reset_control: 1457-1461, 2861-2864)
- `src_linux/g99_android_kernel/android_kernel_gigaset_mt6789/drivers/mmc/host/mtk-mmc.h`
  (раскладка INT, PATCH_BIT*, SDC_RX_ENH_EN, RESPWAIT=(3<<2)!)
- `src_linux/g99_android_kernel/android_kernel_gigaset_mt6789/arch/arm64/boot/dts/mediatek/mt6789.dts`
  (mmc1@11240000: compatible mt6789-mmc, top 0x11ef0000)
- `src_linux/g99_android_kernel/android_kernel_gigaset_mt6789/drivers/misc/mediatek/lpm/modules/include/mt6789/spm_reg.h`
  (MSDC1-биты: только SPM-запросы клоков 658-667, reset-битов нет)
- `u-boot-mtk-g99/arch/arm/dts/mt6789.dtsi` (mmc@11240000: compatible
  mt8183-mmc, БЕЗ resets)
- `src_linux/gki-kernel/drivers/mmc/host/mtk-sd.c` (mt6779_compat: enhance_rx=true)
- `lk_unpack_v0/lk.img.c` (строки 32510-32710: LK init MSDC1: PATCH_BIT=0x403c0007,
  PATCH_BIT1=0xffe64309, PATCH_BIT2=0x14881805, PAD_TUNE0/1=0)
- `firmware/Infinix_Note_12_Pro_X676B_MT6789_V2385_221208_MXML/Firmware/extracted_dtb/main_tree.dts`
  (mmc1@11240000 Android DT: cd-gpios GPIO7, no-mmc/no-sdio)
- `u-boot-mtk-g99/drivers/mmc/mtk-sd.c` (наш драйвер: mt8183_compat — БЕЗ enhance_rx)
- `docs/AGENTS/mt6789_msdc1_android_live_state_2026-08-08.md` (питание/клоки Android)

## Прочие факты сессии
- **/dev/mem на X676B (Android, KernelSU root)**: MMIO читается НУЛЯМИ даже для
  работающего eMMC (0x11230000) — снятие регистров MSDC из Android невозможно
  (MTK mem-протекция). dd read() → "Bad address" (STRICT_DEVMEM);
  mmap O_RDONLY → EACCES; работает только O_RDWR|O_SYNC и отдаёт нули.
  SELinux нужно в Permissive; `su -c` нестабилен (иногда uid=2000 shell,
  надёжнее `su 0 <cmd>`).
- /proc/iomem скрывает адреса (все 0).
- memdump (static arm64, mmap /dev/mem): /tmp/opencode/memdump.c
