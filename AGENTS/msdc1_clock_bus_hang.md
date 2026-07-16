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
