# Display Init Sequence — MT6789 (ILI7835) — TESTED & WORKING

## Статус
- **Протестировано на устройстве**: экран переключается с logo BL2_ext в управляемый режим
- **Проблема**: кэш CPU — OVL читает из физической памяти, а `mw.l` пишет в кэш
- **Решение**: cache clean (DC CVAC) перед триггером DSI

---

## 🟢 МИНИМАЛЬНАЯ РАБОЧАЯ ПОСЛЕДОВАТЕЛЬНОСТЬ (2026-08-05, подтверждено: красный и зелёный экран)

**Контекст**: всё остальное уже настроено LK (клоки, MUTEX, routing, OVL L0/L1, DSI, пайплайн). Не требуется ничего перенастраивать — только обновить кадр.

### Один кадр = 4 шага:

```
# 1. dc off — mw.l пишет напрямую в память (иначе OVL увидит старые данные)
dc off

# 2. Залить FB (0x278d00 слов = 1080×2400×4 = весь буфер)
mw.l 0xfeabe000 0xff00ff00 0x278d00

# 3. Сброс DSI — ОБЯЗАТЕЛЕН перед КАЖДЫМ кадром (BUSY=1 после прошлого кадра)
mw.l 0x14013010 0x01000001; md.l 0x14013010 1; md.l 0x14013010 1; mw.l 0x14013010 0x01000001; md.l 0x1401300c 1
# → BUSY снимается: 0x80001016 → 0x00001016

# 4. Триггер кадра
mw.l 0x14013d00 0x2c3909; mw.l 0x14013060 0x0001; mw.l 0x14013010 0x01000002
```

**Модель**: «1 кадр = 1 сброс + CMDQ 0x2C + DSI_EN». После каждого кадра BUSY возвращается (0x80001016) — следующий кадр без сброса не уходит.

**Подтверждение из ядра**: `mtk_dsi_wait_for_idle()` (mtk_dsi.c:1027) при таймауте делает то же самое — `mtk_dsi_enable()` + `mtk_dsi_reset_engine()` (toggle bit0 CON_CTRL).

**Не работает**:
- Toggle DSI_START (0x0→0x1) без CMDQ — экран не меняется
- CMDQ+DSI_EN без предварительного сброса при BUSY=1 — кадр не уходит

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

Три записи в CLR-регистр (порядок не важен).

**Исправлено 2026-08-05** — прежние подписи битов были неверны. Раскладка по
`src_linux/linux/drivers/clk/mediatek/clk-mt6789-mm.c:32-52`:

| Значение | Биты | Что включает |
|----------|------|--------------|
| 0x420803 | 0,1,11,17,22 | MUTEX0, APB_BUS, SMI_INFRA, SMI_COMMON, SMI_GALS |
| 0x4 | 2 | **OVL0** clock (не RDMA0) |
| 0x80000 | 19 | **DSI0** clock (не MUTEX0) |
| 0x10000 | 16 | DITHER0 clock |

⚠️ На живом устройстве `md.l 0x14000100 1` → `0xfe848040`, то есть **все эти биты уже нулевые
(клоки включены LK)**. Весь Блок 1 фактически no-op; он оставлен как страховка, но не является
«инициализацией». Разбор — в `заметки/предложения_по_дисплею.md`, раздел 1.3.

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

---

## ⚠️ Критическое замечание: драйвер mtk_disp_g99 мешает ручным экспериментам (2026-08-05)

**Проблема**: если в U-Boot загружен драйвер `mtk_disp_g99.c`, каждая команда shell
вызывает `video_sync()`, который **полностью переконфигурирует пайплайн** и отправляет
свой кадр. Это перетирает любые ручные изменения регистров.

**Подтверждение**: `md.l 0xff4a1450 2` → `f00dface 00000002` — video_sync() отработал
после последней команды.

**Решение**: перед ручными экспериментами нужно либо:
1. Убрать vidconsole из stdout: `setenv stdout serial,usbacm` + `setenv stderr serial,usbacm`
2. Отключить драйвер (uboot-lk-*.img без mtk_disp_g99)
3. Использовать `md.l` для чтения регистров (не вызывает video_sync)

**Текущее состояние драйвера** (из свежей диагностики):
- DSI в command mode (`MODE_CTRL = 0`)
- DSI_BUSY застрял на 1 (`INTSTA = 0x80001016`)
- Layer 1 включён LK, Layer 0 выключен (`SRC_CON = 0x2`)
- Оба слоя смотрят на один FB (`0xfeabe000`)
- vidconsole в stdout → video_sync() вызывается на каждый символ

---

## Рабочая последовательность DSI trigger (2026-08-05)

**Открыто**: правильный порядок команд для обновления дисплея в command mode.

### Проблема
Ранее использовался DSI trigger с `CMDQ_SIZE = 0x8001` (trigger bit = bit 15) + toggle `DSI_START`.
Это **ломало** контроллер: DSI_BUSY застрял на 1, DSI_EN не держался.

### Решение
**CMDQ_SIZE = 0x0001** (без trigger-бита) + **DSI_EN = 1**. Контроллер отправляет pending
CMDQ автоматически при включении DSI_EN.

### Рабочая последовательность (однострочник)

```
mw.l 0x14013d00 0x2c3909; mw.l 0x14013060 0x0001; mw.l 0x14013010 0x01000002
```

| Шаг | Команда | Регистр | Значение | Описание |
|-----|---------|---------|----------|----------|
| 1 | `mw.l 0x14013d00 0x2c3909` | DSI_CMDQ | `0x2c3909` | DCS 0x2C (Write Memory Start) |
| 2 | `mw.l 0x14013060 0x0001` | DSI_CMDQ_SIZE | `0x0001` | count=1, **без trigger-бита** (бит 15 = 0) |
| 3 | `mw.l 0x14013010 0x01000002` | DSI_CON | `0x01000002` | DSI_EN=1 (бит 1) + bit 24 |

### Почему это работает
1. CMDQ загружается в очередь (шаг 1)
2. CMDQ_SIZE указывает количество команд (шаг 2)
3. При установке DSI_EN контроллер **автоматически** отправляет pending CMDQ (шаг 3)
4. Панель получает 0x2C и обновляет GRAM из текущего OVL/FB

### Почему старый способ ломал
- `CMDQ_SIZE = 0x8001` (бит 15 = trigger) + toggle `DSI_START` — это **двойной триггер**
- Контроллер пытается отправить CMDQ дважды → BUSY застряает
- Или: trigger при BUSY=1 → контроллер падает

### Важные замечания
1. **DSI_EN не держится** — CON читается как `0x01000000` (бит 1 = 0) после записи `0x01000002`
   Это нормально для command mode: контроллер отправляет CMDQ и автоматически отключается
2. **Bit 24 в CON** — обязателен. Если его нет (CON = 0x00000002), DSI_EN не работает
3. **DSI нужно сбросить перед каждой отправкой** — если BUSY=1, команда не уйдёт
4. **Только одна команда за раз** — нельзя загружать несколько CMDQ подряд

### Последовательность для вывода на экран

```
# 1. Сброс DSI (если BUSY=1)
mw.l 0x14013010 0x01000001; md.l 0x14013010 1; md.l 0x14013010 1; mw.l 0x14013010 0x01000001; md.l 0x1401300c 1

# 2. Записать данные в FB
mw.l 0xfeabe000 0x0000ff00 0x278d00

# 3. Отправить DSI trigger
mw.l 0x14013d00 0x2c3909; mw.l 0x14013060 0x0001; mw.l 0x14013010 0x01000002
```

### Что подтверждено на железе
- ✅ Текст консоли появился на экране (FB содержит вывод vidconsole)
- ✅ Экран обновился без toggle DSI_START
- ✅ Контроллер не падает (BUSY не застряает)
- ❌ Заливка цвета в FB + trigger не показала цвет (причина неясна — возможно cache или OVL)
