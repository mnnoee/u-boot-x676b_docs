# MT6789 MSDC1 — ЖИВОЕ СОСТОЯНИЕ ANDROID (adb + KernelSU root) — 2026-08-08

**Источник:** работающий Android на X676B, `adb shell su -c` (KernelSU, uid=0)
**Метод:** debugfs (/sys/kernel/debug), regmap pwrap, pinctrl debugfs
**Сырые данные:** папка `docs/AGENTS/android_live_dump_2026-08-08/` (clk_summary.txt,
pwrap_registers.txt, regulator_summary.txt, pins.txt)

## 🎯 ГЛАВНОЕ: правильные герцовка и питание — ФАКТЫ с работающего устройства

### 1. Клоки MSDC1 в Android (clk_summary.txt)

```
msdcpll          rate = 383999878  (≈384 МГц)  ← PLL ВКЛЮЧЁН в Android!
   msdcpll_d2    rate = 191999939  (≈192 МГц)
      msdc30_1_sel rate = 191999939  ← mux = msdcpll_d2 (индекс 4), source = 192 МГц
         msdc30_1_ck / ifrao_msdc1_clk = 192 МГц
   msdcpll_d4    rate = 95999969   (≈96 МГц)
mainpll_d4_d2   rate = 273000000   ← msdc5hclk_sel (HCLK) = 273 МГц
```

- **Реальный source MSDC1 в Android = 192 МГц** (msdcpll_d2 при ВКЛЮЧЁННОМ PLL 384 МГц)
- В U-Boot mux тот же (индекс 4), но PLL выключен → bypass 26 МГц (не соответствует Android)
- **Карта работает на шине 48 МГц = 192/4** (см. ios ниже) — при 26 МГц это физически
  невозможно, подтверждение, что в Android source реально 192 МГц

### 2. Состояние карты (/sys/kernel/debug/mmc0/ios)

```
clock:          50000000 Hz
actual clock:   47999984 Hz   ← 192 МГц / 4
vdd:            17 (2.9 ~ 3.0 V)
bus mode:       2 (push-pull)
bus width:      2 (4 bits)
timing spec:    7 (sd uhs DDR50)
signal voltage: 1 (1.80 V)    ← VMC = 1.8V, ПОСТОЯННО (не только после UHS!)
power mode:     2 (on)
```

### 3. Питание SD (pwrap regmap, /sys/kernel/debug/regmap/10026000.pwrap/registers)

| Регистр | Android (факт) | Назначение |
|---|---|---|
| 0x1E48 | **0x0200** | VMCH (vmmc) = 2.9V, [10:8]=2 |
| 0x1E4C | **0x0400** | VMC (vqmmc) = 1.8V, [11:8]=4 |
| 0x1CD8 | **0x0001** | VMCH enable |
| 0x1CC4 | **0x0001** | VMC enable |
| 0x1CE6 | 0xC000 | VMCH статус (биты 14,15) |
| 0x1CD2 | 0xC000 | VMC статус (биты 14,15) |
| 0x1E42 | 0x0005 | VMCH ANA |
| 0x1E46 | 0x000D | VMC ANA |

Регуляторы (regulator_summary.txt): `vmch` 2900mV enabled ← 11240000.mmc-vmmc
(2900–3000mV); `vmc` **1800mV** enabled ← 11240000.mmc-vqmmc (**1800–1950mV** — в
стоковом DT vqmmc допускает ТОЛЬКО 1.8V!)

**ВЫВОД: Android держит VMC=1.8V с самого начала (constraint регулятора!), а не
3.3V при старте.** Наш первый вариант питания (1E48=0x200, 1E4C=0x400) — ЕДИНСТВЕННО
правильный. Вариант 3.3V сигнала (0xD00) НЕ соответствует Android.

### 4. Pinctrl MSDC1 (pins.txt + pinmux-pins)

```
pin 71: func1 → 11240000.mmc
pin 72-76: func1 → 11240000.mmc
```

GPIO 71–76 в Android = **mode 1** (MSDC), совпадает с нашей настройкой в U-Boot
(mode регистры 0x10005380/0x10005390, IES/SMT/PUPD iocfg_rt). Пины НЕ причина.

## ❌ Открытый вопрос

## ✅ ОБНОВЛЕНИЕ (после сверки с вендорным драйвером g99_android_kernel)

### MSDCPLL: Android и U-Boot ИДЕНТИЧНЫ (побайтово)

Вендорный драйвер `clk-mt6789.c:1602-1607` (дословно):
```c
PLL(CLK_APMIXED_MSDCPLL, "msdcpll", MSDCPLL_CON0/*base*/,
	MSDCPLL_CON0, 0, 0/*en*/,
	MSDCPLL_CON3/*pwr*/, 0, BIT(0)/*rstb*/,
	MSDCPLL_CON1, 24/*pd*/,
	0, 0, 0/*tuner*/,
	MSDCPLL_CON1, 0, 22/*pcw*/),
#define MSDCPLL_CON0 0x350  MSDCPLL_CON1 0x354  MSDCPLL_CON2 0x358  MSDCPLL_CON3 0x35C
```

Свежие сырые чтения (Android regmap и U-Boot md.l, оба подтверждены отдельными чтениями):

| Регистр | Android | U-Boot | Бит | Значение |
|---|---|---|---|---|
| CON0 0x350 | 0x000003E1 | 0x000003E1 | [0] EN | **1 (включён)** |
| CON1 0x354 | 0x031D89D8 | 0x031D89D8 | [21:0] PCW | 0x1D89D8 (1935832) |
| CON1 0x354 | — | — | [26:24] POSDIV | 3 |
| CON2 0x358 | 0x001D89D8 | 0x001D89D8 | — | не используется драйвером, но РЕАЛЬНО читается (access y y y n) |
| CON3 0x35C | 0x80000001 | 0x80000001 | [0] PWR_ON | **1 (запитан)** |
| CON3 0x35C | — | — | [1] ISO | **0 (не изолирован)** |

**PLL включён и запитан в ОБОИХ. Никакого «bypass 26МГц» нет ни в Android, ни в U-Boot.**
f_pll = 26e6 * (1935832/2^22) * 2^5 = 384 МГц (сходится с clk_summary 383999878),
msdcpll_d2 = 192 МГц — реальный source MSDC1 (mux idx 4).

ПРОШЛЫЙ ОТЧЁТ (350=3E0, 35C=2 у Android) — НЕ ВОСПРОИЗВЁЛСЯ при повторном снятии:
вероятно, было переходное состояние PLL. Не является надёжным фактом.

### КОРЕНЬ ПРОБЛЕМЫ (новая версия)

Форс `26МГц` в mtk-sd.c был НЕВЕРЕН: при реальном source 192 МГц делитель 68
даёт 2.8 МГц на линии вместо 400 кГц init → карта не успевает ответить на CMD8.
Android считает делители от 192 МГц → 400 кГц → карта работает.

**Фикс (v191): форс 192000000 вместо 26000000 для MSDC1** — сделано в mtk-sd.c.

### Прочие факты с Android (повторно подтверждены)

- 1E48=0200 (2.9V), 1E4C=0400 (1.8V), 1CD8/1CC4=0001 — наш код питания совпадает
- clk debugfs: msdc30_1_sel parent=msdcpll_d2 rate=191999939, enable_count=0
  (не доказывает ни включения, ни выключения PLL — только CCF-счётчики)
- pwrap regmap дамп полный (0x0000-0xFFFE) — сохранён

## ❌ Открытый вопрос

В U-Boot (после включения питания 2.9/1.8V): CDSTS=1, DAT0=1 (ps=0x10002) — карта
жива физически, но CMD8/CMD55 = таймаут. Не помогают:
- питание 2.9/1.8 (как Android) — карта видна (DAT0=1), CMD8 молчит
- пины mode 1 (идентичны Android) — не причина

Гипотезы:
1. **Bypass-клок не равен 26 МГц или нестабилен** — CKSTB=1, но частота/форма не
   проверены. Возможно, стоит включить MSDCPLL (CON2[0]=1) и работать от 192 МГц
   (как Android), тогда драйвер сам выставит делитель для 400 кГц init.
2. **Не тот источник 400 кГц init** — проверить, что в set_mclk делитель даёт
   100–400 кГц при форсированном 26 МГц (было: div=17 → sclk=382 кГц — в норме).
3. CD GPIO7 active-high — проверить.

## План

1. Вернуть питание 2.9/1.8 (первый вариант) — уже сделано в msdc_pwrap_sd_power.
2. Попробовать включить MSDCPLL в U-Boot (writel CON2[0]=1, PCW как в LK:
   CON0=0x3E1, CON1=0x31D89D8) и убрать форс 26 МГц — чтобы source был 192 МГц,
   как в Android. Драйвер тогда посчитает делитель для 400 кГц от 192 МГц.
3. Если PLL-включение не даст ответа — снять осциллографом CLK-линию (пин 71).

## Ссылки

- `android_live_dump_2026-08-08/clk_summary.txt` — полное дерево клоков (365 строк)
- `android_live_dump_2026-08-08/pwrap_registers.txt` — полный дамп PMIC через regmap
- `android_live_dump_2026-08-08/regulator_summary.txt` — регуляторы
- `android_live_dump_2026-08-08/pins.txt` — pinctrl pins (все пины)
- `msdc1_sd_power_and_clocks.md` — прошлый док (правка: vqmmc constraint = 1.8V!)
- `mt6789_msdc1_live_diagnostics_2026-08-06.md` — PLL выключен в U-Boot/LK
