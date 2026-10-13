# MSDC1 (SD-карта) — питание и клоки на MT6789 (X676B)

**Дата:** 2026-08-06 · **Источник:** анализ ядра (mtk-mmc.ko, clk-mt6789-*.c), стокового DT
и регуляторной подсистемы. Сводка того, что нужно для оживления SD-карты в U-Boot.

## Питание SD — только через PMIC (единственный путь)

**fixed-regulator / GPIO-питания для SD НЕ существует.** В стоковом DT (`mmc1` в `base_tree.dts`):

```
mmc1: mmc@11240000 {
    vmmc-supply  = <0x85>;  // ldo_vmch — VDD карты, 2.9–3.3V, НЕ always-on
    vqmmc-supply = <0x86>;  // ldo_vmc  — VccQ шины, 1.8–3.3V, НЕ always-on
    cd-gpios = <0x49 0x07 0x00>;   // GPIO 7, active-high (0x00), debounce 0
    ...
};
```

Оба LDO динамические, по умолчанию **выключены**. Включает их только ядро
(mt6358-regulator.ko → pwrap). Ни preloader, ни LK питание SD не включают
(в LK нет ни одного обращения к 0x1CD8/0x1CC4). Значит U-Boot обязан включить
VMCH/VMC сам через pwrap WACS2 (протокол — в `pwrap_wacs2_protocol_verified.md`).

### Как драйвер ядра подаёт питание (mtk-mmc.ko_detailed.c, msdc_ops_set_ios стр. 2937)

- power_mode=OFF: `mmc_regulator_set_ocr(vmmc, 0)` + `regulator_disable(vqmmc)`
- power_mode=ON:  `regulator_enable(vqmmc)` (vmmc уже включён)
- power_mode=UP:  `msdc_init_hw()` (reset) → `mmc_regulator_set_ocr(vmmc, ios->vdd)` → udelay ~100 мс
- Переключение pinctrl "default"/"state_uhs" при смене сигнального напряжения (для MSDC1 оба состояния идентичны)

### Регистры LDO (MT6366 = MT6358-совместимый)

| LDO | Enable (бит 0) | Напряжение (ANA_CON0) | Статус |
|---|---|---|---|
| **VMCH** | 0x1CD8 | 0x1E48, биты [10:8]: 2=2.9V, 3=3.0V, **5=3.3V** | 0x1CE6 (QI bit0) |
| **VMC** | 0x1CC4 | 0x1E4C, биты [11:8]: 4=1.8V, 0xA=2.9V, 0xB=3.0V, **0xD=3.3V** | 0x1CD2 (QI bit0) |

Последовательность включения для обычной SD-карты:
1. VMCH 3.3V: `WDATA=0x0500 → CMD=0x20001E48`, затем `WDATA=0x0001 → CMD=0x20001CD8`
2. VMC 3.3V: `WDATA=0x0D00 → CMD=0x20001E4C`, затем `WDATA=0x0001 → CMD=0x20001CC4`
3. Пауза ≥ 60 мкс (ramp) + ~100 мс (как драйвер)
4. Для UHS-I позже: VMC 1.8V (селектор 4)
5. Верификация: читать VMCH/VMC CON0 (бит 0 = 1) или QI-бит

## Клоки MSDC1 — полный список

### Topckgen (0x10000000) — `clk-mt6789-topckgen.c`

`msdc30_1_sel` (ID 116), MUX_GATE_CLR_SET_UPD:
- mux: **0x10000080** биты [10:8]; set=0x84; clr/gate=**0x88 бит 15**; update=0x04 бит 29
- Родители: 0=tck_26m (26 МГц — хватает для CMD0), 1=univpll_d6_d2, 2=mainpll_d6_d2,
  3=mainpll_d7_d2, **4=msdcpll_d2** (требует поднятой msdcpll: 0x1000C350 bit0 + PCW 0x354)
- Gate: enable = запись `1<<15` в 0x10000088 (CLR); состояние в 0x10000080 бит 15 (0=вкл)
- ⚠️ В ядре драйвер parent НЕ переключает (нет clk_set_rate в mtk-sd.c) — берёт от загрузчика.
  Для init на 26 МГц msdcpll не нужна.

### Infra AO (0x10001000) — `clk-mt6789-infra_ao.c`, группа IFRAO1

| Клок | Роль | Бит | set 0x88 / clr 0x8C / sta 0x94 |
|---|---|---|---|
| `ifrao_msdc1` (21) | hclk | **4** | включение = запись в CLR (0x8C) |
| `ifrao_msdc1_clk` (28) | source_cg | **16** | включение = запись в CLR (0x8C) |

(eMMC для сравнения: msdc0 биты 2/6/17.) Уже сделано в U-Boot: `writel(BIT(4)|BIT(16), 0x1000108C)`.

### Доказательство «клок доходит» — CKSTB

`msdc_set_mclk` ждёт **бит 7 (CKSTB)** регистра MSDC_CFG (0x11240000) —
`while ((readl(0x11240000) >> 7 & 1) == 0)`. Это «clock stable», НЕ DLL.
Проверка с железа: 0x11240000 = 0x02000099 → бит 7 = 1 ✓ (клок реально тикает).

## Пин-группы MSDC1 из стокового DT (пины 71–76)

- cmd/dat: GPIO 73,74,75,76,72 — mode 1, input, pull-up, drive 4 мА
- clk: GPIO 71 — mode 1, pull-down
- В U-Boot уже настроены вручную (v178/v179), формула pin mode: `base + (pin/8)*4`, бит (pin%8)*4

## MSDC1 vs MSDC0 — отличия

| | mmc0 (eMMC 0x11230000) | mmc1 (SD 0x11240000, 4-бит) |
|---|---|---|
| detect | non-removable | cd-gpios GPIO 7 active-high, **broken-cd нет** |
| vmmc | vemc | vmch |
| vqmmc | — | vmc |
| клоки | +crypto, bus_clk | только source, hclk, source_cg |
| скорость | hs400/hs200 | sdr12/25/50/104, ddr50, cap-sd-highspeed |
| ocr-voltage | — | 0x30000 (2.9–3.1V) |

## Чек-лист «оживить SD» (порядок)

1. ✅ Инфра-клоки MSDC1: CLR 0x1000108C = BIT(4)|BIT(16) — готово (v189)
2. ✅ Topckgen: gate 0x10000088 bit15 (enable), mux на 26 МГц (индекс 0) — готово (v180/v190)
3. ✅ Пины 71–76 mode 1 + IES/SMT/PUPD — готово (v178/v179)
4. ✅ CKSTB (бит 7 0x11240000) = 1 — подтверждено
5. ✅ pwrap WACS2 канал работает (SWCID 0x6610) — подтверждено 2026-08-06
6. ⛔ **Включить VMCH 3.3V + VMC 3.3V через pwrap** — СЛЕДУЮЩИЙ ШАГ (не сделано!)
7. ⛔ CD: GPIO 7 active-high (карта вставлена → 1)
8. Проверить ответ на CMD0

## Ссылки

- `unpack/modules_decompile/mtk-mmc.ko_detailed.c` — msdc_ops_set_ios (стр. 2937), msdc_init_hw (стр. 1568), msdc_set_mclk (стр. 3487)
- `src_linux/linux/drivers/clk/mediatek/clk-mt6789-topckgen.c` — msdc30_1_sel
- `src_linux/linux/drivers/clk/mediatek/clk-mt6789-infra_ao.c` — ifrao_msdc1 (бит 4), msdc1_clk (бит 16)
- `src_linux/linux/drivers/regulator/mt6358-regulator.c:616-623` — mt6366_regulators (vmch/vmc)
- `заметки/devtree/base_tree.dts` / `docs/AGENTS/stock_dtb_msd_analysis.md` — узел mmc1
- `docs/AGENTS/pwrap_wacs2_protocol_verified.md` — протокол доступа к PMIC
