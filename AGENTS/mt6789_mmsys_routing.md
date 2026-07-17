# MT6789 MMSYS Routing Table — OVL0 → RDMA0 → DSI0

## Source
`/workdir/git/heliog99/src_linux/linux/drivers/soc/mediatek/mt6789-mmsys.h`

---

## Полная routing-таблица MT6789

```c
static const struct mtk_mmsys_routes mmsys_mt6789_routing_table[] = {
    // OVL0 → RDMA0
    MMSYS_ROUTE(OVL0, RDMA0, 0xf18, BIT(0), BIT(0)),           // line 47
    MMSYS_ROUTE(OVL0, RDMA0, 0xf18, GENMASK(2,0), (0 << 0)),   // line 50

    // OVL0 → WDMA0
    MMSYS_ROUTE(OVL0, WDMA0, 0xf18, BIT(2), BIT(2)),           // line 49

    // OVL0_2L → RDMA0
    MMSYS_ROUTE(OVL_2L0, RDMA0, 0xf14, BIT(0), BIT(0)),        // line 52
    MMSYS_ROUTE(OVL_2L0, RDMA0, 0xf28, GENMASK(2,0), (2 << 0)),// line 55

    // RDMA0 → DSI0 (via RSZ0)
    MMSYS_ROUTE(RDMA0, DSI0, 0xf0c, GENMASK(1,0), (0 << 0)),   // line 57
    MMSYS_ROUTE(RDMA0, DSI0, 0xf30, GENMASK(2,0), (0 << 0)),   // line 58

    // RDMA0 → COLOR0
    MMSYS_ROUTE(RDMA0, COLOR0, 0xf0c, GENMASK(1,0), (1 << 0)), // line 59

    // DITHER0 → DSI0
    MMSYS_ROUTE(DITHER0, DSI0, 0xf20, BIT(0), BIT(0)),         // line 61
    MMSYS_ROUTE(DITHER0, DSI0, 0xf30, GENMASK(2,0), (1 << 0)), // line 62

    // DITHER0 → WDMA0
    MMSYS_ROUTE(DITHER0, WDMA0, 0xf20, BIT(1), BIT(1)),        // line 63

    // DITHER0 → DSC0
    MMSYS_ROUTE(DITHER0, DSC0, 0xf20, BIT(2), BIT(2)),         // line 64

    // DSC0 → WDMA0
    MMSYS_ROUTE(DSC0, WDMA0, 0xf38, BIT(0), BIT(0)),           // line 66

    // DSC0 → DSI0
    MMSYS_ROUTE(DSC0, DSI0, 0xf38, BIT(1), BIT(1)),            // line 67
    MMSYS_ROUTE(DSC0, DSI0, 0xf30, GENMASK(2,0), (2 << 0)),    // line 68
};
```

---

## Регистры routing (0xf0c-0xf38)

| Offset | Name | Description |
|--------|------|-------------|
| 0xf0c | RDMA0_RSZ0_SOUT | RDMA0 output select: 0=→DSI0, 1=→COLOR0 |
| 0xf14 | OVL0_2L_MOUT | OVL0_2L output mux: bit0=RDMA0, bit1=RSZ0, bit2=WDMA0 |
| 0xf18 | OVL0_MOUT | OVL0 output mux: bit0=RDMA0, bit1=RSZ0, bit2=WDMA0 |
| 0xf20 | DITHER0_MOUT | DITHER0 output mux: bit0=DSI0, bit1=WDMA0, bit2=DSC0 |
| 0xf28 | RDMA0_SEL_IN | RDMA0 input select: 0=OVL0, 1=RSZ0, 2=OVL0_2L |
| 0xf30 | DSI0_SEL_IN | DSI0 input select: 0=RDMA/RSZ0, 1=DITHER0, 2=DSC0 |
| 0xf38 | DSC0_MOUT | DSC0 output mux: bit0=WDMA0, bit1=DSI0 |

---

## Текущие значения на устройстве (после init)

```
0xf14 = 0x0  → OVL0_2L: no output
0xf18 = 0x1  → OVL0: bit0 = RDMA0 ✅
0xf1c = 0x0
0xf20 = 0x1  → DITHER0: bit0 = DSI0
0xf24 = 0x0
0xf28 = 0x0  → RDMA0 input: OVL0 ✅
0xf30 = 0x1  → DSI0 input: DITHER0 ⚠️
0xf34 = 0x0
0xf38 = 0x0  → DSC0: no output
```

**Текущий пайплайн**: OVL0 → RDMA0 → **DITHER0** → DSI0

---

## Нужные значения для прямого пути OVL0 → RDMA0 → DSI0

Для **прямого** пути (без DITHER0):

| Регистр | Оффiset | Нужное значение | Описание |
|---------|---------|-----------------|----------|
| OVL0_MOUT | 0xf18 | **0x1** (bit 0) | OVL0 → RDMA0 |
| RDMA0_SEL_IN | 0xf28 | **0x0** (bits[2:0]=0) | RDMA0 ← OVL0 |
| RDMA0_RSZ0_SOUT | 0xf0c | **0x0** (bits[1:0]=0) | RDMA0 → DSI0 |
| DSI0_SEL_IN | 0xf30 | **0x0** (bits[2:0]=0) | DSI0 ← RDMA/RSZ0 |

### Команды для U-Boot

```
# Прямой путь: OVL0 → RDMA0 → DSI0 (без DITHER0)
mw.l 0x14000f18 0x1
mw.l 0x14000f28 0x0
mw.l 0x14000f0c 0x0
mw.l 0x14000f30 0x0
```

### Альтернатива: через DITHER0 (как сейчас)

```
# Путь: OVL0 → RDMA0 → DITHER0 → DSI0
mw.l 0x14000f18 0x1
mw.l 0x14000f28 0x0
mw.l 0x14000f20 0x1
mw.l 0x14000f30 0x1
```

**Рекомендация**: использовать прямой путь (без DITHER0) — он проще и не требует настройки DITHER0.

---

## Визуальная схема

```
Прямой путь (нужен):
  OVL0 ──→ RDMA0 ──→ DSI0 ──→ Панель
  [0xf18=1] [0xf28=0] [0xf0c=0] [0xf30=0]

Через DITHER0 (текущий):
  OVL0 ──→ RDMA0 ──→ DITHER0 ──→ DSI0 ──→ Панель
  [0xf18=1] [0xf28=0]  [0xf20=1]  [0xf30=1]
```
