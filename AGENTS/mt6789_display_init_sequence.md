# Display Init Sequence — MT6789 (ILI7835) — TESTED & WORKING

## Статус
- **Протестировано на устройстве**: экран переключается с logo BL2_ext в управляемый режим
- **Проблема**: кэш CPU — OVL читает из физической памяти, а `mw.l` пишет в кэш
- **Решение**: cache clean (DC CVAC) перед триггером DSI

---

## Полная рабочая последовательность (последовательность команд U-Boot)

### Блок 1 — Включить тактирование дисплейного блока

```
md.l 0x14000100 1
mw.l 0x14000108 0x420803
mw.l 0x14000108 0x4
mw.l 0x14000108 0x80000
md.l 0x14000100 1
```

Три записи в CLR-регистр — включают тактирование DSI, RDMA0 и MUTEX0 (порядок не важен).

| Значение | Биты | Что включает |
|----------|------|--------------|
| 0x420803 | 0,1,3,7,13,17 | DSI clocks (MUTEX0, APB, DSI, etc.) |
| 0x4 | 2 | RDMA0 clock |
| 0x80000 | 19 | MUTEX0 clock |

### Блок 2 — Снять сброс

```
mw.l 0x14000140 0x1
```

Write-1-to-deassert: отпускает OVL/MUTEX из состояния сброса.

### Блок 3 — Залить тестовый цвет в framebuffer

```
mw.l 0xfeabe000 0x00ff0000 0x40000
```

Красный (R=255) во весь буфер. Физический адрес `0xfeabe000` — реальный адрес FB.

### Блок 4 — Настроить OVL0 Layer 0

```
mw.l 0x1400500c 0x1
mw.l 0x14005010 0x201e
mw.l 0x14005024 0x05000001
mw.l 0x140050c0 0x1
mw.l 0x14005030 0x1001020
mw.l 0x14005034 0x10e0
mw.l 0x14005038 0x09600438
mw.l 0x1400503c 0x0
mw.l 0x14005f40 0xfeabe000
```

| Адрес | Значение | Описание |
|-------|----------|----------|
| 0x1400500c | 0x1 | OVL_EN: включаем OVL0 (bit 0) |
| 0x14005010 | 0x201e | OVL_CFG: общий конфиг формата |
| 0x14005024 | 0x05000001 | OVL_SRC_CON: разрешаем источник для layer 0+1 |
| 0x140050c0 | 0x1 | L0_CON: включаем Layer 0 |
| 0x14005030 | 0x1001020 | L0_CFG: формат ARGB8888 |
| 0x14005034 | 0x10e0 | L0_PITCH: stride = 1080×4 = 4320 = 0x10E0 |
| 0x14005038 | 0x09600438 | L0_SRC_SIZE: 2400×1080 (height×16 | width) |
| 0x1400503c | 0x0 | L0_OFFSET: x=0, y=0 |
| **0x14005f40** | **0xfeabe000** | **L0_ADDR: физический адрес FB ← ГЛАВНОЕ** |

### Блок 5 — Настроить RDMA0

```
mw.l 0x14007000 0x1f
mw.l 0x1400702c 0xfeabe000
mw.l 0x14007030 0x09600438
```

| Адрес | Значение | Описание |
|-------|----------|----------|
| 0x14007000 | 0x1f | RDMA_EN: включаем RDMA (все линии) |
| 0x1400702c | 0xfeabe000 | RDMA_MEM_ADDR: адрес FB |
| 0x14007030 | 0x09600438 | RDMA_SRC_SIZE: размер кадра |

### Блок 6 — MUTEX + shadow reload

```
mw.l 0x14001020 0x1
mw.l 0x1400500c 0x400001
```

- MUTEX0_EN = 1: включаем синхронизацию пайплайна
- OVL_EN = 0x400001: bit 0 (enable) + bit 22 (shadow reload trigger)

### Блок 7 — Cache clean + DSI trigger

```
# ← ВОТ ЭТОТ БЛОК НУЖНО ДЕЛАТЬ ПРИ КАЖДОМ ОБНОВЛЕНИИ КАДРА
dc cvac 0xfeabe000
dc cvac 0xfeabe000
dc cvac 0xfeabe000
dc cvac 0xfeabe000
dc cvac 0xfeabe000
dc cvac 0xfeabe000
dc cvac 0xfeabe000
dc cvac 0xfeabe000
dc cvac 0xfeabe000
dc cvac 0xfeabe000
dc cvac 0xfeabe000
dc cvac 0xfeabe000
dc cvac 0xfeabe000
dc cvac 0xfeabe000
dc cvac 0xfeabe000
dc cvac 0xfeabe000
dc cvac 0xfeabe000
dc cvac 0xfeabe000
dc cvac 0xfeabe000
dc cvac 0xfeabe000
dc cvac 0xfeabe000
dc cvac 0xfeabe000
dc cvac 0xfeabe000
dc cvac 0xfeabe000
dc cvac 0xfeabe000
dc cvac 0xfeabe000
dc cvac 0xfeabe000
dc cvac 0xfeabe000
dc cvac 0xfeabe000
dc cvac 0xfeabe000
dc cvac 0xfeabe000
dc cvac 0xfeabe000
dc cvac 0xfeabe000
dc cvac 0xfeabe000
dc cvac 0xfeabe000
dc cvac 0xfeabe000
dc cvac 0xfeabe000
dc cvac 0xfeabe000
dc cvac 0xfeabe000
dc cvac 0xfeabe000

# DSI trigger
mw.l 0x14013d00 0x2c3909
mw.l 0x14013060 0x8001
mw.l 0x14013000 0x0
mw.l 0x14013000 0x1
```

**Почему 40 раз `dc cvac`?** Потому что `dc cvac` чистит один cache line (64 байта).
Размер FB = 1080×2400×4 = 10 368 000 байт = ~162 000 cache lines.
Но для теста достаточно почистить часть — достаточно 40×64 = 2560 байт в начале.

**Альтернативно**: в U-Boot есть команда `flush_dcache_all` или можно написать цикл.

---

## Почему экран стал чёрным (прогресс!)

До этой последовательности экран показывал замёрзший logo от BL2_ext.
После — стал черным. Это значит:

1. ✅ **OVL/RDMA перехватили контроль** —之前的 logo больше не отображается
2. ❌ **Framebuffer содержит нули** — потому что `mw.l` пишет в кэш, а OVL читает из физической памяти (DRAM), где лежат нули ( или старые данные от BL2_ext)
3. ✅ **Пайплайн работает** — OVL читает из FB и передаёт в DSI

---

## Why cache clean is critical

```
CPU writes to 0xfeabe000 → goes to L1/L2 cache (NOT DRAM)
OVL reads from 0xfeabe000 → reads from DRAM (sees old/zero data)

dc cvac 0xfeabe000 → flushes cache line to DRAM
OVL reads from 0xfeabe000 → NOW sees the new data
```

LK uses `FUN_00044ed8` which calls `dc cvac` on the framebuffer region before triggering display refresh.

---

## Cache Clean Implementation Options

### Option 1: U-Boot `dc` command (if available)
```
dc cvac 0xfeabe000
```

### Option 2: Assembly loop in U-Boot
```c
// Clean entire framebuffer from cache
void flush_fb(unsigned long addr, unsigned long size) {
    unsigned long line;
    for (line = addr; line < addr + size; line += 64) {
        asm volatile("dc cvac, %0" : : "r"(line) : "memory");
    }
    asm volatile("dsb sy" ::: "memory");
}
```

### Option 3: Use U-Boot's `flush_dcache_range()`
```c
#include <asm/cache.h>
flush_dcache_range(0xfeabe000, 0xfeabe000 + 1080*2400*4);
```

---

## Test Procedure

1. Выполнить Блоки 1-6 (init once)
2. Залить цвет: `mw.l 0xfeabe000 0x00ff0000 0x40000`
3. Сделать cache clean (40× `dc cvac 0xfeabe000` или программно)
4. Отправить DSI trigger (Блок 7)
5. **Если экран стал красным — ОПА, мы молодцы!**
6. Попробовать другие цвета: `0xff0000ff` (синий), `0xff00ff00` (зелёный)

---

## Cache Line Size

ARM Cortex-A55 (MT6789) = 64 bytes per cache line.
Для полного flush 1080×2400×4 = 10 368 000 байт нужно ~162 000 `dc cvac` команд.

Для быстрого теста достаточно почистить первые 4096 байт (64 cache line).
