# MSDC1 (SD Card) Clock Gate — Bus Hang Investigation

## Статус: НЕ РАЗРЕШЕНО
- CLR register (0x1000108C) hang'ится при записи
- SET register (0x10001088) работает, но DISABLE gate (active-low)
- Bus protection registers (0x100012A0) вызывают crash при записи

## Что работает
- MMC subsystem: probe проходит, `mmc list` показывает `mmc@11240000: 0`
- `mmc info`, `mmc rescan`, `mmc part` — выполняются БЕЗ ошибок
- SD карта вставлена, но не детектится

## Что НЕ работает
- MSDC1 registers (0x11240000) читаются как все нули — контроллер не тактируется
- `mmc info` не выводит данные о карте

## Ключевые регистры

### IFRAO1 Clock Gate (base 0x10001000)
| Register | Offset | Address | Описание |
|----------|--------|---------|----------|
| SET | 0x88 | 0x10001088 | Write 1 = DISABLE gate (active-low!) |
| CLR | 0x8C | 0x1000108C | Write 1 = ENABLE gate (HANGS!) |
| Status | 0x94 | 0x10001094 | Read: 1 = gate disabled |

### MSDC1 Gate Bits
| Clock | Bit | SET Address | CLR Address |
|-------|-----|-------------|-------------|
| hclk (AHB bus) | 4 | 0x10001088 | 0x1000108C |
| source_cg | 16 | 0x10001088 | 0x1000108C |

### Topckgen Mux Gate (MSDC30_1_SEL)
| Register | Address | Bit |
|----------|---------|-----|
| Gate reg | 0x10000088 | 15 |
| CLR | 0x1000008C | 15 |

### Bus Protection (INCRACFG)
| Function | SET | CLR | Status |
|----------|-----|-----|--------|
| TOP_AXI_PROT_EN | 0x100012A0 | 0x100012A4 | 0x10001228 |
| TOP_AXI_PROT_EN_MM | 0x100012D4 | 0x100012D8 | 0x100012EC |

### MSDCPLL
| Register | Address | Status |
|----------|---------|--------|
| MSDCPLL_CON0 | 0x1000C350 | bit 0 = PLL enable (working) |

## Последовательность из explore agents

### Explore-1 (ядро + bus protection)
- MSDC1 **НЕ имеет** `power-domains` в DTS
- MSDC1 **НЕ требует** bus protection release
- INFRA bus path к MSDC1 должен быть всегда доступен
- Баг в драйвере: `mtk_gate_enable()` пишет в CLR (0x8C), но CLR hang'ится

### Explore-2 (LK + clock setup)
- LK **НЕ делает** явную настройку шин/часов перед MSDC
- Preloader (BL2) освобождает bus protection и включает infra часы
- LK пишет `0x8000000` (bit 27) в infra_ao1 только для display, не для MSDC

## Гипотезы
1. **Preloader не включает MSDC1 hclk** — BL2 может настраивать только eMMC (MSDC0), а не SD (MSDC1)
2. **CLR register защищён другим механизмом** — возможно нужен特殊的 sequence для в CLR
3. **INFRA bus clock (axi_ck) не включён** — все IFRAO clocks зависят от него

## Следующие шаги
1. Проверить статус bus protection: `md.l 0x10001228 1` (status = 0xe60c1806)
2. Попробовать включить axi_ck через другой механизм
3. Посмотреть что BL2 делает с MSDC1 clock — может быть MSDC1 hclk просто не включён в BL2
4. Проверить может ли `mw.l 0x1000108C 0x10` (только hclk, без source_cg) заработать

## Результат explore-3 (BL2_ext analysis)

**BL2_ext НЕ включает MSDC1 hclk (IFRAO1 bit 4).** 
- BL2_ext пишет в IFRAO1 только биты 27,28 (UFS storage)
- msdc_init вызывается ТОЛЬКО для MSDC0 (eMMC), не MSDC1
- MSDC1 pinmux код существует но НЕДОСТИЖИМ
- FUN_00039410 (вызывается для MSDC1) — пустая функция (no-op)

**Clock gates для MSDC1 (bits 4,16) были УЖЕ включены до我们的 записи!**
- Status register до SET write: bits 4,16 = 0 (enabled)
- После SET write: bits 4,16 = 1 (disabled — мы случайно ВЫКЛЮЧИЛИ!)

**Настоящая проблема: bus protection на MSDC1 address space (0x11240000)**
- BL2_ext снимает protection только для UFS и eMMC
- MSDC1 (SD card) НЕ защищён от release — шина к 0x11240000 блокирована
- CLR register (0x1000108C) hang'ится — это может быть признаком защищённой шины

## Прорыв (2026-07-17)

### Что сработало
1. **CLR register (0x1000108C) НЕ hang'ится!** Запись `mw.l 0x1000108C 0x10010` проходит успешно
2. **MSDC1 registers НЕ нули**: 0x11240000 = 0x02000099 (MSDC_CFG — контроллер жив!)
3. **Clock gates включены**: Status 0x18004600, bits 4,16 = 0 (enabled)
4. **Source clock mux**: 0x10000088 = 0x00000000 (parent 0 = F26M 26MHz)

### Оставшаяся проблема
- `mmc rescan` → "Card did not respond to voltage select! : -110 (ETIMEDOUT)"
- Контроллер доступен, но карта не отвечает на CMD0/CMD8
- Возможные причины:
  1. Питание SD карты не включено
  2. Card detect GPIO не настроен
  3. Source clock не запущен (gate включён, но PLL/parent не настроен)
  4. Пин-мукс не настроен

### Что нужно сделать дальше
1. Проверить питание SD карты (enable pin, voltage regulator)
2. Настроить card detect GPIO
3. Проверить что source clock реально тактирует (записать/прочитать MSDC_CFG)
4. Посмотреть пин-мукс для MSDC1 в BL2_ext (FUN_0003986c)

## MT6366 PMIC LDO Enable для SD карты (2026-07-17)

### PWRAP Bridge
- Base: 0x10026000
- WACS2_CMD: 0x10026880 (BIT(29)|addr для записи)
- WDATA: 0x10026884
- RDATA: 0x10026894
- VLDCLR: 0x100268A4

### LDO_VMCH (питание SD карты 3.3V)
- LDO_VMCH_CON0: 0x1CD8 (bit 0 = enable)
- VMCH_ANA_CON0: 0x1E48 (bits [10:8] = voltage: 2=2.9V, 3=3.0V, 5=3.3V)

### LDO_VMC (I/O питание 1.8-3.3V)
- LDO_VMC_CON0: 0x1CC4 (bit 0 = enable)
- VMC_ANA_CON0: 0x1E4C (bits [11:8] = voltage: 4=1.8V, 0xA=2.9V, 0xB=3.0V, 0xD=3.3V)

### Команды U-Boot
```
# VMCH = 3.3V
mw.l 0x10026884 0x0500; mw.l 0x10026880 0x20001E48  (set voltage)
mw.l 0x10026884 0x0001; mw.l 0x10026880 0x20001CD8  (enable)

# VMC = 3.3V
mw.l 0x10026884 0x0D00; mw.l 0x10026880 0x20001E4C  (set voltage)
mw.l 0x10026884 0x0001; mw.l 0x10026880 0x20001CC4  (enable)
```
