# Session: Crash in `initr_caches` → `initr_mmc` (v148+)

## Problem
- v147 boots fine
- v148+ crashes — изменение: добавлен `obj-$(CONFIG_TARGET_MT6789) += clk-mt6789.o`
- **Серия экспериментов (v154-v159) показала:** MMU + cache включаются успешно. Краш в `initr_mmc` → `msdc_drv_probe()` → `clk_get_rate(&host->src_clk)`

## Что было сделано

### v154 — Маркеры в MMU установку
- `mmu_setup`/`setup_all_pgtables`/`create_table` — все маркеры.
- **Результат:** MMU включается успешно. `R:after_caches` достигнут. Краш ПОСЛЕ initr_caches.

### v155 — DEVICE entry для буфера 0x43FFF000
- `MT_DEVICE_NGNRNE` в mem_map для 0x43FFF000 (4KB).
- **Результат:** DEVICE entry увеличил число page tables → краш в `setup_all_pgtables` (emergency pass не вместился).
- **Вывод:** DEVICE entry ломает layout таблиц.

### v156 — flush_dcache_range после dcache_enable
- DEVICE entry откачен. Вместо него `flush_dcache_range(0x43FFF000, ...)` после `set_sctlr(CR_C)`.
- **Результат:** `dcache_enable:after_cache_on` виден, но `R:after_caches` — НЕТ из-за кэша.
- **Вывод:** Записи после dcache_enable идут только в L1 кэш, теряются при watchdog reset.

### v157 — `dc cvac` в `early_log_write` (ПРОРЫВ)
- В `early_log_write` добавлен `dc cvac` всей страницы 0x43FFF000 если CR_C установлен.
- Убран ручной `flush_dcache_range` из `dcache_enable`.
- **Результат:** **ВСЕ маркеры видны!** Полная цепочка до краша:
  ```
  R:ENTRY → R:before_caches → mmu_setup → dcache_enable →
  R:after_caches → R:before_dm → R:after_dm →
  R:before_dm_devices → R:before_serial → R:after_serial →
  R:before_mmc →
  clk:topckgen:probe → clk:apmixedsys:probe → clk:infracfg:probe →
  → КРАШ (R:after_mmc отсутствует)
  ```
- `ram_top=0xc0000000` (3GB) — отличается от v156 (5GB). Возможно другой бут или разный SKU.

### v158 — Маркеры в mtk-sd.c
- Добавлены `EARLY_W` в `msdc_drv_probe()`.
- **Результат:**
  ```
  msdc:probe:entry
  msdc:probe:before_clk_get_rate
  → КРАШ внутри clk_get_rate(&host->src_clk)
  ```
- **Точное место:** `host->src_clk_freq = clk_get_rate(&host->src_clk)` (строка ~1670 mtk-sd.c).
- `host->src_clk` = `CLK_TOP_MSDC30_1_SEL` (mux в topckgen, ID 116).

### v159 — Маркеры в clk-mtk.c
- Добавлены `EARLY_W` в `mtk_topckgen_get_rate` и `mtk_topckgen_get_mux_rate`.
- Ждём лог.

## Найденный баг (clock driver)

Краш в `clk_get_rate` для `CLK_TOP_MSDC30_1_SEL (ID=116)`:

1. `mtk_topckgen_get_mux_rate()` читает mux-регистр topckgen+0x88, биты [10:8]
2. Если mux выбран на parent index 0 (`CLK_TOP_F26M, ID=63`):
3. `mtk_topckgen_find_parent_rate()` → `mtk_clk_find_parent_rate()` → `clk_get_rate()` с ID=63
4. `mtk_topckgen_get_rate()` проверяет:
   - `id < fdivs_offs (1)`? НЕТ (63 >= 1)
   - `id < muxes_offs (95)`? ДА (63 < 95) → вызывает `mtk_topckgen_get_factor_rate(clk, 62)`
5. `fdivs[62]` — **ВЫХОД ЗА ГРАНИЦЫ МАССИВА** (только 54 элемента, индексы 0-53)!

**Корень:** ID `CLK_TOP_F26M=63` не является фактором (это fixed clock с `FIXED_CLK(CLK_TOP_F26M, CLK_XTAL, 26000000)`). Но т.к. `63 >= fdivs_offs (1)` и `63 < muxes_offs (95)`, код обрабатывает его как фактор.

**Доп. баг:** `mtk_apmixedsys_get_rate()` использует `pll = &priv->tree->plls[id]` — прямой доступ по ID как индексу массива. Но `CLK_APMIXED_MSDCPLL=6` — 6-й элемент массива, а реально MSDCPLL на позиции 2 (`apmixed_plls[2]`). Это даёт ТВДПЛЛ вместо MSDCPLL (но не крашит).

## Что дальше (v160)
- **Вариант A:** Добавить `CLK_TOP_F26M` (и др. fixed clocks) как FACTOR entry в `top_fixed_divs`:
  ```c
  FACTOR(CLK_TOP_F26M, CLK_XTAL, 1, 1, CLK_PARENT_XTAL),
  ```
- **Вариант B:** Модифицировать `mtk_topckgen_get_rate()` для проверки наличия ID в fclks/fdivs
- **Вариант C:** Заполнить все пропуски в fdivs (IDs 54-55, 63-94)

## Файлы изменены
- `include/early_log.h` — dc cvac после записи (v157)
- `arch/arm/cpu/armv8/cache_v8.c` — убран ручной flush (v157)
- `drivers/mmc/mtk-sd.c` — #include <early_log.h>, EARLY_W маркеры (v158)
- `drivers/clk/mediatek/clk-mtk.c` — #include <early_log.h>, EARLY_W маркеры (v159)
- `drivers/clk/mediatek/clk-mt6789.c` — (ещё не изменён — будет фикс)

## Адреса (v157)
- `tlb_addr=0xbffd0000 sz=0x2c000`
- `ram_top=0xc0000000 reloc=0xbfec2000`
- EARLY_W буфер: `0x43FFF000` (4KB, flushing)
- topckgen: `0x10000000`; apmixedsys: `0x1000c000`; infracfg_ao: `0x10001000`
- MSDC2 (SD карта): `0x11240000`

## Остаётся
- Определить точное значение mux-регистра topckgen+0x88
- Исправить out-of-bounds в fdivs (добавить F26M)
- Исправить PLL array lookup (по ID, не по индексу)
- Затем: MMC probe, UFS/UART/LK compat issues
