# Topckgen Bus Protection Release — MT6789

## Проблема
Writing to topckgen SET register `0x10000088` has no effect — value stays `0x00000000` after write.
The gate is already enabled (status bit 15=0), but we can't change the MUX or other gates.
Card doesn't respond to CMD0 because source clock (F26M) may not be running.

## Ключевой инсайт: SET/CLR registers are write-only

**Это критически важно:** на MediaTek topckgen регистры SET/CLR — **write-only**.
Чтение SET регистра ВСЕГДА возвращает 0. Это нормальное поведение!

Реальное состояние читается из **STATUS** регистра:
- `0x10000090` = GATE status (read-only)
- MUX state: читать напрямую из MUX регистра (`0x10000080`, `0x10000084`, etc.)

**Проверка:** Если `md.l 0x10000088` возвращает 0 — это НОРМАЛЬНО.
Нужно читать STATUS регистр или MUX регистры напрямую.

## Bus Protection Mechanism (INFRA domain)

Bus protection на MT6789 управляется через **infracfg_ao** (base `0x10001000`).
Это НЕ topckgen — это шинная защита, которая блокирует доступ от master к slave.

### Register Groups (все offsets от `0x10001000`)

| Group | SET | CLR | STA |
|-------|-----|-----|-----|
| TOPAXI_PROT_EN | `0x02A0` | `0x02A4` | `0x0228` |
| TOPAXI_PROT_EN_1 | `0x02A8` | `0x02AC` | `0x0258` |
| TOPAXI_PROT_EN_MM | `0x02D4` | `0x02D8` | `0x02EC` |
| TOPAXI_PROT_EN_2 | `0x0714` | `0x0718` | `0x0724` |
| TOPAXI_PROT_EN_MM_2 | `0x0DCC` | `0x0DD0` | `0x0DD8` |
| TOPAXI_PROT_EN_INFRA_VDNR | `0x0B84` | `0x0B88` | `0x0B90` |

### Как работает bus protection

1. **SET** register: write 1 to **enable** protection (блокирует доступ)
2. **CLR** register: write 1 to **disable** protection (разрешает доступ)
3. **STA** register: read to check current state (1 = protected)

### Формат BUS_PROT_WR_IGN
```c
BUS_PROT_WR_IGN(INFRA, mask, SET_ofs, CLR_ofs, STA_ofs)
```
- `WR_IGN` = "write ignore" — CLR операция НЕ ждёт acknowledgment
- Используется для power domain protection

### Release Sequence (из mtk-pm-domains.c)
```c
// Для release protection:
regmap_write(regmap, bpd->bus_prot_clr, bpd->bus_prot_set_clr_mask);
// BUS_PROT_IGNORE_CLR_ACK → return immediately (no poll)
// Без флага → poll STA until bits clear
```

## DISP Power Domain Bus Protection

Из `mt6789-pm-domains.h`:
```c
[MT6789_POWER_DOMAIN_DISP] = {
    .bp_cfg = {
        BUS_PROT_WR_IGN(INFRA, MT6789_TOP_AXI_PROT_EN_MM_DISP, 0x02D4, 0x02D8, 0x02EC),
        BUS_PROT_WR_IGN(INFRA, MT6789_TOP_AXI_PROT_EN_DISP, 0x02A0, 0x02A4, 0x0228),
    },
},
```

### Биты (из infracfg.h)
```c
MT6789_TOP_AXI_PROT_EN_MM_DISP  = BIT(10) | BIT(12)  // MM group
MT6789_TOP_AXI_PROT_EN_DISP     = BIT(6)  | BIT(23)  // TOPAXI group
```

### Release DISP bus protection (U-Boot команды)
```bash
# 1. Release MM_DISP protection (CLR register)
mw.l 0x100012D8 0x00001000   # BIT(10) | BIT(12) = 0x1000
# STA should clear: md.l 0x100012EC

# 2. Release DISP protection (TOPAXI CLR register)
mw.l 0x100012A4 0x00800040   # BIT(6) | BIT(23) = 0x00800040
# STA should clear: md.l 0x10001228
```

## Все MT6789 Bus Protection Entries

| Domain | Bit Mask | SET | CLR | STA |
|--------|----------|-----|-----|-----|
| MD | `1<<7` | 0x02A0 | 0x02A4 | 0x0228 |
| MD (VDNR) | `BIT(0)\|BIT(10)\|BIT(15)` | 0x0B84 | 0x0B88 | 0x0B90 |
| CONN | `BIT(13)\|BIT(18)` | 0x02A0 | 0x02A4 | 0x0228 |
| CONN_2ND | `BIT(14)` | 0x02A0 | 0x02A4 | 0x0228 |
| CONN_1 | `BIT(10)` | 0x02A8 | 0x02AC | 0x0258 |
| MFG1_1 | `BIT(21)` | 0x02A8 | 0x02AC | 0x0258 |
| MFG1_2 | `BIT(5)\|BIT(6)` | 0x0714 | 0x0718 | 0x0724 |
| MFG1 | `BIT(21)\|BIT(22)` | 0x02A0 | 0x02A4 | 0x0228 |
| MFG1_2ND | `BIT(7)` | 0x0714 | 0x0718 | 0x0724 |
| MM_2_ISP | `BIT(8)` | 0x0DCC | 0x0DD0 | 0x0DD8 |
| MM_2_ISP_2ND | `BIT(9)` | 0x0DCC | 0x0DD0 | 0x0DD8 |
| MM_IPE | `BIT(16)` | 0x02D4 | 0x02D8 | 0x02EC |
| MM_IPE_2ND | `BIT(17)` | 0x02D4 | 0x02D8 | 0x02EC |
| MM_VDEC | `BIT(24)` | 0x02D4 | 0x02D8 | 0x02EC |
| MM_VDEC_2ND | `BIT(25)` | 0x02D4 | 0x02D8 | 0x02EC |
| MM_VENC | `BIT(26)` | 0x02D4 | 0x02D8 | 0x02EC |
| MM_VENC_2ND | `BIT(27)` | 0x02D4 | 0x02D8 | 0x02EC |
| **MM_DISP** | **`BIT(10)\|BIT(12)`** | **0x02D4** | **0x02D8** | **0x02EC** |
| **DISP** | **`BIT(6)\|BIT(23)`** | **0x02A0** | **0x02A4** | **0x0228** |
| AUDIO_2 | `BIT(4)` | 0x0714 | 0x0718 | 0x0724 |
| MM_CAM | `BIT(0)\|BIT(2)` | 0x02D4 | 0x02D8 | 0x02EC |
| MM_CAM_2ND | `BIT(1)\|BIT(3)` | 0x02D4 | 0x02D8 | 0x02EC |

## Связь с MSDC1 (SD Card)

### Текущий статус (из msdc1_clock_bus_hang.md)
- Bus protection status `0x10001228` = `0xe60c1806`
- CLR register `0x1000108C` hang'ится при записи
- Bus protection registers `0x100012A0` вызывают crash

### Чтение status
```bash
# Все bus protection status registers
md.l 0x10001228 1   # TOPAXI_PROT_EN status
md.l 0x10001258 1   # TOPAXI_PROT_EN_1 status
md.l 0x100012EC 1   # TOPAXI_PROT_EN_MM status
md.l 0x10001724 1   # TOPAXI_PROT_EN_2 status (0x10001000+0x724)
md.l 0x10001DD8 1   # TOPAXI_PROT_EN_MM_2 status (0x10001000+0xDD8)
md.l 0x10001B90 1   # TOPAXI_PROT_EN_INFRA_VDNR status (0x10001000+0xB90)
```

### Release all protections (agnostic approach)
```bash
# Release ALL bus protections by writing to ALL CLR registers
# TOPAXI_PROT_EN
mw.l 0x100012A4 0xFFFFFFFF
# TOPAXI_PROT_EN_1
mw.l 0x100012AC 0xFFFFFFFF
# TOPAXI_PROT_EN_MM
mw.l 0x100012D8 0xFFFFFFFF
# TOPAXI_PROT_EN_2
mw.l 0x10001718 0xFFFFFFFF
# TOPAXI_PROT_EN_MM_2
mw.l 0x10001DD0 0xFFFFFFFF
# TOPAXI_PROT_EN_INFRA_VDNR
mw.l 0x10001B88 0xFFFFFFFF
```

## Topckgen Register Map (0x10000000)

| Offset | Address | Description |
|--------|---------|-------------|
| 0x000-0x07C | 0x10000000-0x1000007C | MUX registers (R/W, bits [10:8] = parent) |
| 0x080 | 0x10000080 | GATE_CLR (write-only, write 1 to enable) |
| 0x084 | 0x10000084 | GATE_SET (write-only, write 1 to disable) |
| 0x088 | 0x10000088 | GATE_CLR_1 (write-only) |
| 0x08C | 0x1000008C | GATE_SET_1 (write-only) |
| 0x090 | 0x10000090 | GATE_STATUS_0 (read-only) |
| 0x094 | 0x10000094 | GATE_STATUS_1 (read-only) |

### MSDC30_1_SEL MUX
- Offset: 0x088 (CLK_CFG_2_CLR)
- Bit [10:8] = parent select
- Parent 0 = F26M (26MHz)

### Проверка MUX state
```bash
# Читать MUX регистр напрямую (НЕ SET/CLR!)
md.l 0x10000088 1   # Это GATE_CLR, always reads 0!
md.l 0x10000028 1   # Это MUX register для MSDC30_1_SEL
md.l 0x10000090 1   # GATE status
```

## Выводы

1. **topckgen SET/CLR регистры — write-only**, чтение всегда = 0. Это нормально.
2. **Bus protection** блокирует шинные доступы, НЕ topckgen регистры напрямую.
3. **Для topckgen** нужно проверять MUX регистры (0x000-0x07C) и STATUS (0x090-0x094).
4. **CLR register hang** в infracfg_ao (0x1000108C) может быть вызван bus protection.
5. **Решение**: сначала release bus protection, потом включать clock gates.

## Источники
- `src_linux/linux/drivers/pmdomain/mediatek/mt6789-pm-domains.h` — bus protection entries
- `src_linux/linux/drivers/pmdomain/mediatek/mtk-pm-domains.c` — release implementation
- `src_linux/linux/include/linux/soc/mediatek/infracfg.h` — bit definitions
- `src_linux/linux/arch/arm64/boot/dts/mediatek/mt6789.dtsi` — device tree
- `docs/AGENTS/msdc1_clock_bus_hang.md` — existing investigation
- `firmware/*/extracted_dtb/main_tree.dts` — stock DTS
