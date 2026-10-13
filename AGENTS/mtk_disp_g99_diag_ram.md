# mtk_disp_g99 — RAM Diagnostic Buffer

## Все адреса
| Адрес | Блок | Описание |
|-------|------|----------|
| 0xff4a1400 | probe() | Блок 0x00: fb/plat_base снимок |
| 0xff4a1420 | probe() + 1st sync | Блок 0x20: vidconsole state |
| 0xff4a1450 | video_sync() | Блок 0x50: sync entry marker |

## Адрес
```
0xff4a1400  (сразу после framebuffer: LK_DEFAULT_FB + DISP_FB_SIZE)
```

## Формат (5 × u32)
| offset | поле | probe() | video_sync() 1-й вызов |
|--------|------|---------|------------------------|
| +0x00  | magic | `0xDEADBEEF` | — |
| +0x04  | uc_priv->fb | `0x00000000` (ещё не установлен) | `0xfeabe000` |
| +0x08  | plat->base | `0xfeabe000` | `0xfeabe000` |
| +0x0c  | uc_priv->fb_size | `0` (ещё не установлен) | `0x009e3400` |
| +0x10  | plat->size | `0x009e3400` | `0x009e3400` |

## Чтение через ACM shell
```bash
# После boot, в U-Boot shell (через USB ACM консоль):
md.l 0xff4a1400 5
```

Ожидаемый вывод:
```
ff4a1400: deadbeef 00000000 feabe000 00000000    ................
ff4a1410: 009e3400                                ..4.
```

Если `magic != 0xDEADBEEF` — probe() не выполнился (драйвер не загрузился, или MMIO недоступен).

## Где пишется
- В `mtk_disp_probe()` — однократно, в конце, после настройки pipeline.
- `flush_dcache_range()` после записи.
- В `video_sync()` — блок 0x50: `0xF00DFACE` при входе, статус в +0x54
  (0x1 = early return null, 0x2 = полный успех).

## Как читать
```bash
# Полный дамп всех блоков:
md.l 0xff4a1400 12

# Только probe():
md.l 0xff4a1400 5

# Только vidconsole state:
md.l 0xff4a1420 8

# Только video_sync entry:
md.l 0xff4a1450 2
```

## Интерпретация video_sync (0xff4a1450)
| +0x50 | +0x54 | Значение |
|-------|-------|----------|
| F00DFACE | — | video_sync вызван |
| F00DFACE | 0x1 | early return: uc_priv/fb/fb_size = NULL |
| F00DFACE | 0x2 | полный успех: flush + DSI trigger выполнен |
| — | — | ops->video_sync не вызывается вообще |

## Интерпретация vidconsole (0xff4a1420)
| +0x20 | Значение | Что означает |
|-------|----------|--------------|
| C0FFEE00 | probe() записал до lcdputs | Первый снимок |
| C0FFEE01 | video_sync() записал при первом вызове | Второй снимок |
| C0FFEE00 |Probe перезаписал (перезагрузка) | |

## Свежая диагностика (2026-08-05, после перезагрузки)

Пример вывода `md.l 0xff4a1400 24`:
```
ff4a1400: deadbeef feabe000 feabe000 009e3400  .............4..
ff4a1410: 009e3400 00000000 00000000 00000000  .4..............
ff4a1420: c0ffee00 ffffffff ffffffff ffffffff  ................
ff4a1430: ffffffff ffffffff ffffffff 00000000  ................
ff4a1450: f00dface 00000002 00000000 00000000  ................
```

Интерпретация:
- `deadbeef` — probe() отработал
- `fb = feabe000`, `plat->base = feabe000`, `fb_size = 9e3400` (10 368 000 байт = 1080×2400×4)
- `c0ffee00` — vidconsole snapshot от probe()
- `ffffffff` в +0x24..+0x3c — vidconsole не найден в момент probe() (ещё не создан)
- `f00dface 00000002` — video_sync() отработал успешно (полный flush + DSI trigger)

**Критическое наблюдение**: video_sync() отработал (status=0x2), значит драйвер переконфигурировал
пайплайн и отправил кадр. Любая ручная команда shell до этого момента была перетёрта.

## Важно: vidconsole мешает ручным экспериментам

`stdout=serial,usbacm,vidconsole` → каждый `printf`/`md.l` в shell вызывает `video_sync()`,
который:
1. Отключает OVL
2. Заново настраивает MMSYS routing, OVL, MUTEX
3. Заливает свой кадр
4. Триггерит DSI

Поэтому **никакой ручной эксперимент не является чистым**, пока vidconsole в stdout.
Решение: либо убрать vidconsole из stdout (`setenv stdout serial,usbacm`), либо отключить
драйвер entirely.
