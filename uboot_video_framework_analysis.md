# Анализ U-Boot Video Framework — Архитектурный шаблон для MTK MT6789

## Содержание

1. [Video Uclass Framework](#1-video-uclass-framework)
2. [Путь vidconsole → video_sync() → driver](#2-путь-vidconsole--video_sync--driver)
3. [Примеры SoC display drivers](#3-примеры-soc-display-drivers)
4. [DSI command-mode в U-Boot](#4-dsi-command-mode-v-u-boot)
5. [Наш MTK driver vs framework](#5-наш-mtk-driver-vs-framework)
6. [Выводы и рекомендации](#6-выводы-и-рекомендации)

---

## 1. Video Uclass Framework

### 1.1 struct video_ops — Единственный hardware callback

```c
// include/video.h:148
struct video_ops {
    int (*video_sync)(struct udevice *vid);
};
```

**Это ВСЁ что framework даёт driver'у.** Нет `.probe`, `.bind`, `.fill`, `.blit` — ничего. Единственный хук — `video_sync`.

### 1.2 struct video_priv — Всё состояние дисплея

```c
// include/video.h
struct video_priv {
    /* Устанавливается driver'ом в probe(): */
    ushort xsize;                   // Ширина в пикселях (1080)
    ushort ysize;                   // Высота в пикселях (2400)
    ushort rot;                     // Поворот: 0/1/2/3
    enum video_log2_bpp bpix;       // Битность: VIDEO_BPP32
    enum video_format format;       // VIDEO_ARGB8888
    const char *vidconsole_drv_name;// Имя console driver'а (NULL = авто)
    int font_size;                  // Размер шрифта (0 = по умолчанию)
    int y_start_offset;             // Y-смещение для текста (пропуск нотча)

    /* Управляется uclass (НЕ ставить в driver): */
    void *fb;                       // Виртуальный адрес framebuffer
    int fb_size;                    // Размер FB в байтах
    struct { int xstart, ystart, xend, yend; } damage; // Грязный прямоугольник
    int line_length;                // Байт на строку (4320 = 1080*4)
    u32 colour_fg;                  // Текущий цвет текста
    u32 colour_bg;                  // Текущий фон
    bool flush_dcache;              // Включён ли dcache flush при sync
    ulong last_sync;                // Время последнего sync
};
```

### 1.3 video_sync() — Что делает uclass

```c
// video-uclass.c:500
int video_sync(struct udevice *vid, bool force)
{
    // 1. Если VIDEO_COPY — копирует грязные регионы из fb в copy_fb
    video_flush_copy(vid);

    // 2. Вызывает ops->video_sync() driver'а (НАШ ХУК!)
    if (ops && ops->video_sync)
        ops->video_sync(vid);

    // 3. Throttle: если прошло < CONFIG_VIDEO_SYNC_MS — выходим
    if (!force && get_timer(priv->last_sync) < CONFIG_VIDEO_SYNC_MS)
        return 0;

    // 4. Flush dcache (грязные регионы если VIDEO_DAMAGE, иначе весь FB)
    video_flush_dcache(vid, false);

    // 5. Обновляем время последнего sync
    priv->last_sync = get_timer(0);

    // 6. Сбрасываем damage-rect
    priv->damage = {MAX, MAX, 0, 0};
}
```

### 1.4 Кто вызывает video_sync()

| Место вызова | Когда |
|---|---|
| `vidconsole_putc()` | После **каждого** выведенного символа |
| `vidconsole_puts()` | После вывода строки (один раз) |
| `vidconsole_newline()` | При переводе строки (scroll) |
| `vidconsole_back()` | При нажатии Backspace |
| `ESC[2J` handler | При очистке экрана |
| `video_fill()` | При заливке цветом |
| `video_clear()` | При очистке экрана |
| `video_bmp_display()` | При показе BMP |
| `video_idle()` (cyclic) | Каждые ~10 мс (периодический refresh) |

### 1.5 Видео uclass driver definition

```c
// video-uclass.c:806
UCLASS_DRIVER(video) = {
    .id                 = UCLASS_VIDEO,
    .name               = "video",
    .post_bind          = video_post_bind,     // Аллокация FB памяти
    .post_probe         = video_post_probe,    // Настройка FB + создание vidconsole
    .priv_auto          = sizeof(struct video_uc_priv),
    .per_device_auto    = sizeof(struct video_priv),
    .per_device_plat_auto = sizeof(struct video_uc_plat),
};
```

### 1.6 video_post_probe() — Порядок действий

1. `priv->fb = map_sysmem(plat->base, plat->size)` — маппинг FB
2. Вычисление `line_length` если driver не задал
3. Установка цветов по умолчанию (чёрный фон, белый текст)
4. `video_clear(dev)` — заливка фона + sync
5. **Создание vidconsole child device** — привязка vidconsole0 driver'а
6. Установка `vc->ycur = priv->y_start_offset` — если задано
7. Показ U-Boot логотипа (если включено)
8. Регистрация cyclic handler для периодического sync

---

## 2. Путь vidconsole → video_sync() → driver

### 2.1 Полная цепочка вызовов

```
stdio_dev->putc(ch)                           // U-Boot console subsystem
  └─ vidconsole_putc(sdev, ch)                // vidconsole-uclass.c:537
       ├─ vidconsole_put_char(dev, ch)         // Обработка символа
       │    ├─ [ESC] vidconsole_escape_char()   // Накопление ANSI-последовательности
       │    ├─ [\n]  vidconsole_newline()        // Перевод строки + scroll
       │    │    ├─ vidconsole_move_rows()       // memmove FB для скроллинга
       │    │    ├─ vidconsole_set_row()         // Очистка новой строки
       │    │    └─ video_sync(parent, false)    // <-- SYNC!
       │    ├─ [\b]  vidconsole_back()           // Backspace
       │    └─ [printable] vidconsole_output_glyph()
       │         ├─ vidconsole_putc_xy()         // Вызов driver callback
       │         │    └─ console_putc_xy()       // console_normal.c:71
       │         │         ├─ lookup font bitmap
       │         │         ├─ compute FB start: fb + y * line_length + x * pbytes
       │         │         ├─ fill_char_vertically()   // Рисование глифа в FB
       │         │         └─ video_damage(parent, x, y, w, h)  // Пометка dirty
       │         └─ [если строка полна → vidconsole_newline()]
       └─ video_sync(parent, false)             // vidconsole-uclass.c:551 ← ГЛАВНЫЙ SYNC!
            ├─ video_flush_copy()               // Копия в copy_fb (если VIDEO_COPY)
            ├─ ops->video_sync(vid)             // НАШ CALLBACK!
            │    └─ mtk_disp_video_sync()
            │         ├─ flush_dcache_range(fb, fb+size)  // CPU cache → DRAM
            │         └─ mtk_disp_dsi_trigger()           // DSI CMDQ → панель
            ├─ video_flush_dcache(vid, false)   // uclass flush (повторный)
            └─ damage = {MAX, MAX, 0, 0}        // Сброс dirty rect
```

### 2.2 vidconsole_priv — Состояние консоли

```c
// include/video_console.h
struct vidconsole_priv {
    struct stdio_dev sdev;      // stdio device
    int xcur_frac;              // Текущий X (дробные единицы = пиксели * 256)
    int ycur;                   // Текущий Y в пикселях
    int rows, cols;             // Размеры консоли в символах
    int x_charsize, y_charsize; // Размер символа в пикселях (8x16)
    int xstart_frac;            // Левое поле (дробные единицы)
    int tab_width_frac;         // Ширина табуляции
    int xsize_frac;             // Ширина экрана (дробные единицы)
    int escape;                 // 1 = накопление ANSI escape
    int escape_len;             // Длина накопленной последовательности
    char escape_buf[32];        // Буфер escape-последовательности
    bool quiet;                 // Подавление вывода
};
```

### 2.3 vidconsole_ops — Callbacks console driver'а

```c
// include/video_console.h
struct vidconsole_ops {
    int (*putc_xy)(struct udevice *dev, uint x_frac, uint y, int cp);      // Рисование глифа
    int (*move_rows)(struct udevice *dev, uint rowdst, uint rowsrc, uint count); // Скроллинг
    int (*set_row)(struct udevice *dev, uint row, int clr);                 // Очистка строки
    int (*entry_start)(struct udevice *dev);                                // Начало строки
    int (*backspace)(struct udevice *dev);                                  // Backspace
    int (*get_font)(...);                                                   // Получение шрифта
    const char *(*get_font_size)(...);                                      // Размер шрифта
    int (*select_font)(...);                                                // Выбор шрифта
    int (*set_cursor_visible)(...);                                         // Видимость курсора
};
```

### 2.4 console_normal.c — Единственный vidconsole driver

```c
// drivers/video/console_normal.c:146
U_BOOT_DRIVER(vidconsole_normal) = {
    .name       = "vidconsole0",
    .id         = UCLASS_VIDEO_CONSOLE,
    .ops        = &console_ops,
    .probe      = console_probe,
    .priv_auto  = sizeof(struct console_simple_priv),
};

struct vidconsole_ops console_ops = {
    .putc_xy           = console_putc_xy,     // Рисование символа в FB
    .move_rows         = console_move_rows,   // Скроллинг через memmove
    .set_row           = console_set_row,     // Очистка строки заливкой
    .get_font_size     = console_simple_get_font_size,
    .get_font          = console_simple_get_font,
    .select_font       = console_simple_select_font,
    .set_cursor_visible = console_set_cursor_visible,
};
```

### 2.5 console_putc_xy() — Рисование символа

```c
// console_normal.c:71
static int console_putc_xy(struct udevice *dev, uint x_frac, uint y, int cp)
{
    // 1. Конвертация UTF-32 → CP437
    ch = console_utf_to_cp437(cp);

    // 2. Поиск битмапа шрифта
    pfont = fontdata->video_fontdata + ch * fontdata->char_pixel_bytes;

    // 3. Вычисление адреса в FB
    start = vid_priv->fb + linenum * line_length + x * pbytes;

    // 4. Заполнение пикселей
    fill_char_vertically(pfont, &line, vid_priv, fontdata, NORMAL_DIRECTION);

    // 5. Пометка как dirty
    video_damage(dev->parent, x, y, fontdata->width, fontdata->height);

    return VID_TO_POS(fontdata->width);
}
```

---

## 3. Примеры SoC display drivers

### 3.1 Rockchip VOP (rk_vop.c)

**Тип**: Continuous DMA scanout (video mode)

**probe():**
```c
int rk_vop_probe(struct udevice *dev)
{
    // 1. AHB bus reset assert/deassert (20us)
    // 2. Map MMIO registers
    // 3. Iterate DTS "port" subnodes, find display output
    // 4. rk_display_init() — program timing, enable Win0
    // 5. video_set_flush_dcache(dev, 1);
}
```

**video_ops**: ПУСТОЙ! Нет video_sync callback.
**Причина**: VOP непрерывно сканирует FB через DMA — trigger не нужен.

```c
static const struct video_ops rk3288_vop_ops = {};  // Пусто!
```

**Ключевой код** — `rkvop_enable()`:
```c
writel(fbbase, &win_regs->win0_yrgb_mst);  // FB адрес для DMA
writel(0x01, &regs->reg_cfg_done);          // Trigger register update (один раз)
```

### 3.2 Sunxi DE2 (sunxi_de2.c)

**Тип**: Continuous DMA scanout с double-buffer

**probe():**
```c
static int sunxi_de2_probe(struct udevice *dev)
{
    // 1. Try LCD display, then HDMI
    // 2. sunxi_de2_init() — полная инициализация пайплайна
    // 3. video_set_flush_dcache(dev, 1);
}
```

**video_ops**: ПУСТОЙ!

```c
static const struct video_ops sunxi_de2_ops = {};  // Пусто!
```

**Ключевой код** — double-buffer trigger:
```c
writel(address, &de_ui_regs->cfg[0].top_laddr);  // FB адрес
writel(1, &de_glb_regs->dbuff);                   // Double-buffer swap
```

### 3.3 Meson VPU (meson_vpu.c)

**Тип**: Canvas-based DMA scanout

**probe():**
```c
static int meson_vpu_probe(struct udevice *dev)
{
    // 1. Map 3 register regions (VPU, HHI, DMC)
    // 2. meson_vpu_init() — 561 строк HW-инициализации!
    // 3. meson_vpu_setup_plane() — OSD plane + canvas setup
    // 4. meson_vpu_setup_venc() — Video encoder
    // 5. meson_vpu_setup_vclk() — Pixel clock
    // 6. video_set_flush_dcache(dev, 1);
}
```

**video_ops**: НЕТ определения в U_BOOT_DRIVER!

**Canvas-based DMA**:
```c
meson_canvas_setup(priv, MESON_CANVAS_ID_OSD1, uc_plat->base,
                   xsize * 4, ysize, MESON_CANVAS_WRAP_NONE,
                   MESON_CANVAS_BLKMODE_LINEAR);
```

### 3.4 i.MX8MP LCDIF v3 (imx/lcdif.c)

**Тип**: Continuous DMA с shadow-load trigger

**video_sync**: ✅ Реализован!
```c
static int lcdifv3_video_sync(struct udevice *dev)
{
    reg = readl(priv->base + LCDIFV3_CTRLDESCL0_5);
    reg |= CTRLDESCL0_5_SHADOW_LOAD_EN;  // bit 30
    writel(reg, priv->base + LCDIFV3_CTRLDESCL0_5);
    return 0;
}
```
**Механик**: Устанавливает бит shadow load — HW загружает новый FB адрес из shadow-регистров при следующем VSYNC. Одноразовый trigger для обновления buffer pointer.

### 3.5 Intel MCDE (mcde_simple.c)

**Тип**: Software-triggered display (ST-Ericsson)

**video_sync**: ✅ Реализован!
```c
static int mcde_simple_video_sync(struct udevice *dev)
{
    // Только для software sync mode
    if (priv->src_synch != SRC_SYNCH_SOFTWARE) return 0;
    // 1. Enable flow (FLOEN)
    val |= MCDE_CRA0_FLOEN;
    writel(val, priv->base + MCDE_CRA0);
    // 2. Software trigger
    writel(MCDE_CHNL0SYNCHSW_SW_TRIG, priv->base + MCDE_CHNL0SYNCHSW);
    // 3. Disable flow
    val &= ~MCDE_CRA0_FLOEN;
    writel(val, priv->base + MCDE_CRA0);
    // 4. Poll completion
    return readl_poll_timeout(priv->base + MCDE_CRA0, val, !(val & MCDE_CRA0_FLOEN), ...);
}
```
**Механик**: Enable flow → SW trigger → disable flow → poll. Полный программный trigger на каждый кадр.

### 3.6 Samsung Exynos FIMD (exynos/exynos_fb.c)

**Тип**: Continuous DMA (RGB) + I80 SW trigger

**video_ops**: Пустой!
```c
static const struct video_ops exynos_fb_ops = {};
```
**Но есть I80 trigger** (для CPU-interface панелей):
```c
// exynos_set_trigger() — для I80 mode:
cfg |= (EXYNOS_I80SOFT_TRIG_EN | EXYNOS_I80START_TRIG);
writel(cfg, &reg->trigcon);
// Poll: EXYNOS_I80STATUS_TRIG_DONE
```
**Важно**: Exynos FIMD поддерживает I80 (CPU) interface с программным триггером — концептуально похоже на command-mode.

### 3.7 Intel Broadwell IGD (broadwell_igd.c)

**Тип**: GPU continuous scanout (x86 PCI device)

**video_ops**: Пустой! Использует VESA BIOS (VBE) для настройки. MTRR write-combining вместо dcache flush.

### 3.8 Tegra DC (tegra/dc.c)

**Тип**: Continuous DMA с double-buffered window update

**Trigger**: ✅ Есть — `state_ctrl` update flip:
```c
writel(GENERAL_ACT_REQ | WIN_A_ACT_REQ | GENERAL_UPDATE | WIN_A_UPDATE,
       &dc->cmd.state_ctrl);
```
**Но это НЕ command-mode trigger** — это обновление конфигурации окна (новый адрес, размер). DMA сканирует непрерывно.

**Кеш**: `mmu_set_region_dcache_behaviour(fb, size, DCACHE_WRITETHROUGH)` —.writethrough вместо flush.

### 3.9 i.MX IPU (imx/ipu_disp.c + ipu_common.c)

**Тип**: Continuous DMA с template microcode engine

**Триггер**: DI counter release + template microcode. `ipu_clear_buffer_ready()` для buffer swap. Double-buffering через CPMEM (Channel Parameter Memory).

**Уникальность**: IPU хранит FB адрес в CPMEM (per-channel DMA descriptor memory), а не в普通 registers.

### 3.10 TI DSS AM625 (tidss/tidss_drv.c)

**Тип**: Pure continuous — VP enable starts scanout. Без trigger.

### 3.11 TI LCDC AM33xx (ti/tilcdc.c)

**Тип**: Pure continuous — raster enable starts DMA. Без trigger.

### 3.12 ZynqMP DP (zynqmp/zynqmp_dpsub.c)

**Тип**: Descriptor-based DMA с VSYNC re-arm

**Trigger**: ✅ Есть — `dma_trigger()`:
```c
writel(DPDMA_GBL_TRG_CH3_MASK, priv->dpdma + DPDMA_GBL);
```
+ VSYNC interrupt re-arms DMA. Self-looping descriptor chain.

**Кеш**: `flush_dcache_range()` на DMA descriptor — критично, HW читает descriptor из DRAM.

### 3.13 Nexell DPC + MIPI (nexell/)

**Триггеры**: ✅ Три уникальных mechanism:
1. **`nx_dpc_set_cmd_buffer_flush()`** — bit 1 of `dpcdataflush` → flush command buffer к DSI
2. **`nx_dpc_set_reg_flush()`** — bit 4 of `dpcdataflush` → flush register updates
3. **`nx_mipi_dsi_remote_reset_trigger()`** — bit 4 of `dsim_escmode` → auto-clearing HW trigger

### 3.14 Сводная таблица ВСЕХ driver'ов

| Driver | Тип | video_sync | Trigger mechanism | Кеш | Platform |
|--------|-----|:----------:|-------------------|:---:|----------|
| Rockchip VOP | Cont. DMA | ❌ Пустой | reg_cfg_done (1 раз) | uclass | ARM SoC |
| Sunxi DE | Cont. DMA | ❌ Пустой | LOAD_REGS + MODE_START | uclass | ARM SoC |
| Sunxi DE2 | Cont. DMA | ❌ Пустой | dbuff (double-buffer) | uclass | ARM SoC |
| Meson VPU | Canvas DMA | ❌ Нет ops | Canvas auto-scan | uclass | ARM SoC |
| Exynos FIMD | Cont. DMA + I80 | ❌ Пустой | I80 SW trigger (опционально) | uclass | ARM SoC |
| Broadwell IGD | GPU scanout | ❌ Пустой | VBE BIOS | MTRR WC | x86 |
| **i.MX8 LCDIF** | Cont. DMA | **✅ Shadow load** | VSYNC-latched buffer swap | uclass + MMU | ARM64 |
| **Intel MCDE** | SW-triggered | **✅ SW trigger** | Enable→Trigger→Disable→Poll | uclass | ARM |
| Tegra DC | Cont. DMA | ❌ Нет ops | state_ctrl update flip | writethrough | ARM |
| i.MX IPU | Cont. DMA | ❌ Нет ops | Template microcode + DI counter | Нет | ARM |
| TI DSS | Cont. DMA | ❌ Нет ops | Нет | uclass | ARM64 |
| TI LCDC | Cont. DMA | ❌ Нет ops | Нет | Нет | ARM |
| **ZynqMP DP** | Desc. DMA | ❌ Нет ops | dma_trigger() + VSYNC re-arm | uclass + flush desc | ARM64 |
| **Nexell DPC** | CMD buffer | ❌ Нет ops | cmd_buffer_flush + reg_flush | uclass | ARM |
| ****MTK G99** | **CMD mode** | **✅ DSI trigger** | **DSI CMDQ 0x2C + DSI_START** | **flush in driver** | **ARM** |

**ВЫВОД**: Из 15 driver'ов только 3 реализуют video_sync:
1. **i.MX8 LCDIF** — shadow load (buffer swap при VSYNC)
2. **Intel MCDE** — полный SW trigger sequence
3. **MTK G99** — DSI CMDQ trigger (наш)

Все остальные — continuous DMA scanout, trigger не нужен. **Наш MTK driver — единственный command-mode driver в U-Boot с video_sync.**

---

## 4. DSI command-mode в U-Boot

### 4.1 Есть ли generic DSI command-mode abstraction?

**НЕТ.** В U-Boot нет абстракции "trigger frame update for command-mode panel".

Существующие DSI-файлы:
- `include/mipi_dsi.h` — API для отправки DCS-команд (`mipi_dsi_dcs_write()`)
- `drivers/video/mipi_dsi.c` — Протокольная логика (создание пакетов)
- `drivers/video/dsi-host-uclass.c` — Минимальный uclass (`init` + `enable`)
- `drivers/video/dw_mipi_dsi.c` — DesignWare DSI host (Rockchip/STM32)
- `drivers/video/mediatek/mtk_disp_g99.c` — **Наш единственный MTK-файл**

### 4.2 DesignWare DSI — command-mode config

```c
// dw_mipi_dsi.c
static void dw_mipi_dsi_command_mode_config(struct dw_mipi_dsi *dsi)
{
    dsi_write(dsi, DSI_TO_CNT_CFG, HSTX_TO_CNT(1000) | LPRX_TO_CNT(1000));
    dsi_write(dsi, DSI_BTA_TO_CNT, 0xd00);
    dsi_write(dsi, DSI_MODE_CFG, ENABLE_CMD_MODE);  // bit 0 = 1
}
```

**Отправка команд** через DesignWare:
```c
// 1. Записать payload в DSI_GEN_PLD_DATA (0x70)
// 2. Записать header в DSI_GEN_HDR (0x6c) — ЭТО ТРИГГЕР!
dw_mipi_dsi_gen_pkt_hdr_write(dsi, hdr_val);
// 3. Ждать пока FIFO опустеет
readl_poll_timeout(dsi->base + DSI_CMD_PKT_STATUS, ...);
```

**Но MT6789 НЕ использует DesignWare!** У MT6789 свой CMDQ-движок:
- CMDQ FIFO: `0x14013d00` (offset 0xd00, а не 0x70/0x6c)
- Trigger: toggle `DSI_START` (0x14013000), а не запись в `DSI_GEN_HDR`

### 4.3 Panel drivers с Write Memory Start

| Panel | Тип | Отправляет 0x2C? | Где? |
|-------|-----|-------------------|------|
| orisetech_otm8009a | Video mode | Один раз при init | `dcs_write_seq(dev, MIPI_DCS_WRITE_MEMORY_START)` |
| lg-lh400wv3 | Video mode | Один раз при init | `dsi_generic_write_seq(dsi, 0x2C)` |
| sharp-lq101r1sx01 | **Command mode** | **НЕТ** | Полагается на Tegra DC hardware |
| mot-panel | **Command mode** | **НЕТ** | Полагается на hardware |

**ВЫВОД**: Panel driver'ы в U-Boot НЕ отправляют 0x2C на каждый кадр. Command-mode панели полагаются на hardware DC+DSI (Tegra). Для MT6789 (где нет hardware auto-trigger) — наш подход с ручным CMDQ правильный.

### 4.4 Tegra DSI — command-mode (единственный пример)

```c
// tegra/dsi.c
static const u32 pkt_seq_command_mode[NUM_PKT_SEQ] = {
    [ 6] = PKT_ID0(MIPI_DSI_DCS_LONG_WRITE) | PKT_LEN0(3) | PKT_LP,
    [10] = PKT_ID0(MIPI_DSI_DCS_LONG_WRITE) | PKT_LEN0(5) | PKT_LP,
};

// Hardware auto-sends 0x2C at each frame start:
value = MIPI_DCS_WRITE_MEMORY_START << 8 | MIPI_DCS_WRITE_MEMORY_CONTINUE;
writel(value, &len->dsi_dcs_cmds);
```

Tegra DSI hardware автоматически отправляет 0x2C когда DC подаёт данные. Это аппаратная поддержка command-mode, которой нет у MT6789.

### 4.5 MIPI_DSI_MODE_VIDEO флаг

```c
// include/mipi_dsi.h
#define MIPI_DSI_MODE_VIDEO          BIT(0)   // отсутствие = command mode!
#define MIPI_DSI_MODE_VIDEO_BURST    BIT(1)
#define MIPI_DSI_MODE_VIDEO_SYNC_PULSE BIT(2)
#define MIPI_DSI_MODE_LPM            BIT(11)  // Low Power Mode
```

Command mode определяется **отсутствием** `MIPI_DSI_MODE_VIDEO` в `mode_flags`.

### 4.6 Nexell DPC+MIPI — единственные с command-buffer flush

```c
// nexell/soc/s5pxx18_soc_dpc.c
// Flush command buffer к DSI:
void nx_dpc_set_cmd_buffer_flush(u32 module_index) {
    reg = readl(&pregister->dpcdataflush);
    writel(reg | (1 << 1), &pregister->dpcdataflush);  // bit 1 = cmd flush
}

// Nexell MIPI — auto-clearing HW trigger:
void nx_mipi_dsi_remote_reset_trigger(u32 module_index) {
    writel(1 << 4, &pregister->dsim_escmode);  // trigger
    while (readl(&pregister->dsim_escmode) & (1 << 4));  // wait auto-clear
}
```

**Паттерн**: Write command → flush buffer → HW auto-sends. Ближе всего к тому, что нужно MTK.

### 4.7 Полный путь DSI command send (DesignWare reference)

```c
// dw_mipi_dsi.c —dw_mipi_dsi_gen_pkt_hdr_write()
// 1. Ждать FIFO не полон:
readl_poll_timeout(dsi->base + DSI_CMD_PKT_STATUS, val, !(val & GEN_CMD_FULL), ...);
// 2. Записать header — ЭТО ТРИГГЕР!
dsi_write(dsi, DSI_GEN_HDR, hdr_val);
// 3. Ждать FIFO пуст:
readl_poll_timeout(dsi->base + DSI_CMD_PKT_STATUS, val, (val & GEN_CMD_EMPTY | GEN_PLD_W_EMPTY), ...);
```

**Ключевой инсайт**: Запись в `DSI_GEN_HDR` (0x6c) ЕСТЬ trigger. Для MT6789 аналог — запись в `DSI0_CMDQ` (0x14013d00) + toggle `DSI0_START`.

---

## 5. Наш MTK driver vs framework

### 5.1 Что мы делаем правильно

| Требование framework | Статус | Реализация |
|----------------------|--------|------------|
| Регистрация в UCLASS_VIDEO | ✅ | `U_BOOT_DRIVER(mtk_disp_g99)` с `.id = UCLASS_VIDEO` |
| video_ops.video_sync | ✅ | `mtk_disp_video_sync()` — единственный доступный callback |
| bind() callback | ✅ | `plat->size = 0` — пропуск аллокации (используем LK FB) |
| probe() callback | ✅ | MMIO check → read FB → pipeline init → register |
| xsize/ysize/bpix/format | ✅ | 1080x2400, VIDEO_BPP32, VIDEO_ARGB8888 |
| line_length | ✅ | 4320 (1080 * 4) |
| y_start_offset | ✅ | 200 (пропуск нотча) |
| flush_dcache | ✅ | `video_set_flush_dcache(dev, true)` |
| VIDEO_DAMAGE | ✅ | `imply VIDEO_DAMAGE` в Kconfig |
| DTS binding | ✅ | `compatible = "mediatek,mt6789-disp"` |

### 5.2 Что мы делаем «неправильно» (но это нормально)

| Аспект | Статус | Почему это OK |
|--------|--------|---------------|
| Нет MTK DSI host driver | ⚠️ | DSI уже инициализирован LK; наш driver только trigger'ит |
| Прямые MMIO-записи вместо clock framework | ⚠️ | Clock framework в U-Boot не всегда доступен для MTK |
| Нет panel driver | ⚠️ | Panel уже настроен LK; нам не нужен отдельный driver |
| Двойной flush (driver + uclass) | ⚠️ | Безвредно; uclass flush после我们的 video_sync |

### 5.3 Полный pipeline: где наш хук

```
                    U-Boot Video Framework
                    ┌─────────────────────────────────────┐
                    │  video_post_probe()                  │
                    │    → video_clear()                   │
                    │    → создание vidconsole0            │
                    │    → периодический video_idle()      │
                    └──────────────┬──────────────────────┘
                                   │
                    ┌──────────────▼──────────────────────┐
                    │  vidconsole_putc(ch)                 │
                    │    → console_putc_xy()               │
                    │    → fill_char_vertically() → FB     │
                    │    → video_damage(x, y, w, h)       │
                    └──────────────┬──────────────────────┘
                                   │
                    ┌──────────────▼──────────────────────┐
                    │  video_sync(vid, false)              │  ← FRAMEWORK
                    │    → ops->video_sync(vid)            │  ← НАШ ХУК
                    │    → video_flush_dcache()            │
                    └──────────────┬──────────────────────┘
                                   │
                    ┌──────────────▼──────────────────────┐
                    │  mtk_disp_video_sync()               │  ← НАШ DRIVER
                    │    1. flush_dcache_range(fb)         │     CPU cache → DRAM
                    │    2. mtk_disp_dsi_trigger()         │     DRAM → Panel
                    │       → writel(0x2c3909, DSI0_CMDQ)  │
                    │       → writel(0x8001, CMDQ_SIZE)    │
                    │       → toggle DSI_START 0→1         │
                    └─────────────────────────────────────┘
                                   │
                    ┌──────────────▼──────────────────────┐
                    │  Hardware Pipeline                   │
                    │  OVL0 → RDMA0 → DITHER0 → DSI0      │
                    │  (DRAM)    (DMA)              (CMDQ) │→ Panel ILI7835F
                    └─────────────────────────────────────┘
```

### 5.4 Регистры MT6789 — полная карта

| Блок | Базовый адрес | Ключевые регистры |
|------|--------------|-------------------|
| MMSYS | `0x14000000` | CLK_CLR `+0x108`, SW_RST `+0x140`, Routing `+0xf18/0xf28/0xf20/0xf30` |
| OVL0 | `0x14005000` | EN `+0x00c`, SRC_CON `+0x024`, L0_CFG `+0x030`, L0_ADDR `+0xf40` |
| RDMA0 | `0x14007000` | EN `+0x000`, MEM_ADDR `+0x02c`, SRC_SIZE `+0x030` |
| MUTEX0 | `0x14001000` | EN `+0x020` |
| DSI0 | `0x14013000` | START `+0x000`, CON `+0x008`, STATUS `+0x00c`, CMDQ_SIZE `+0x060`, **CMDQ `+0xd00`** |

**ВАЖНО**: CMDQ на MT6789 по offset `0xd00`, а НЕ `0x200`!

---

## 6. Выводы и рекомендации

### 6.1 Полная картина: все display driver'ы в U-Boot

Проанализировано **15 SoC display driver'ов** + **6 DSI host/bridge driver'ов** + **8 panel driver'ов**:

**Категория 1: video_sync НЕ нужен (continuous DMA)** — 12 driver'ов:
- Rockchip VOP (rk3288/3328/3399), Sunxi DE/DE2, Meson VPU, Exynos FIMD, Broadwell IGD, Tegra DC, i.MX IPU, TI DSS, TI LCDC
- DMA сканирует непрерывно, trigger не нужен

**Категория 2: video_sync реализован** — 3 driver'а:
- **i.MX8 LCDIF** — shadow load (VSYNC-latched buffer swap)
- **Intel MCDE** — SW trigger sequence (enable→trigger→disable→poll)
- **MTK G99** — DSI CMDQ trigger (наш)

**Категория 3: DSI command-mode support** — 4 driver'а:
- **DesignWare DSI** — write payload→write header→poll status (Rockchip/STM32)
- **Tegra DSI** — hardware auto-sends 0x2C (аппаратная поддержка)
- **Nexell MIPI** — command buffer flush + remote reset trigger
- **MTK G99** — CMDQ FIFO + DSI_START toggle (наш)

### 6.2 Наш driver — единственный command-mode video driver в U-Boot

**Ключевой факт**: Ни один другой U-Boot driver НЕ делает то, что делаем мы:
1. CPU пишет в framebuffer (через vidconsole)
2. Flush dcache (CPU cache → DRAM)
3. Триггер DSI command (0x2C Write Memory Start) для отправки кадра на панель

Все остальные command-mode panel driver'ы (sharp, mot-panel) полагаются на hardware DC+DSI auto-trigger (Tegra). Наш MT6789 не имеет такой аппаратной поддержки — CMDQ trigger делается программно.

### 6.3 Архитектурный шаблон — где наш хук

```
Прямые MMIO-записи → это НОРМАЛЬНО для MT6789
(нет upstream MTK DSI host, нет clock framework support)

Наш driver:
├── UCLASS_VIDEO registration     ← правильно
├── video_ops.video_sync          ← единственный доступный callback
├── flush_dcache_range()          ← CPU cache → DRAM (DMA needs clean data)
├── DSI CMDQ trigger              ← MT6789-specific (0x2c3909 + DSI_START)
└── y_start_offset = 200          ← пропуск нотча
```

### 6.4 Главная проблема — НЕ архитектура, а clock gating

Из предыдущих сессий известно, что:
- DSI0 clock (бит 19 в `MMSYS_CLK_STA` `0x14000100`) **gated** (выключен)
- Запись `0x80000` в `MMSYS_CLK_CLR` не помогает — нужна запись `0xFFFFFFFF` для включения ВСЕХ клоков
- DISP_26M (бит 10 в mm1 `0x140001a0`) тоже может быть выключен

**Решение**: В `mtk_disp_pipeline_init()` заменить покомандные `writel` на:
```c
writel(0xFFFFFFFF, MMSYS_CLK_CLR);   // Включить ВСЕ клоки mm0
writel(0xFFFFFFFF, 0x140001A8);       // Включить ВСЕ клоки mm1
```

### 6.5 Что НЕ нужно менять

- **Архитектура driver'а** — она уже правильная и уникальная (command-mode video driver)
- **video_ops** — мы реализуем единственный доступный callback
- **vidconsole интеграция** — framework сам создаёт vidconsole и вызывает video_sync
- **Прямые MMIO-записи** — для MT6789 это нормально (нет upstream support)
- **CMDQ trigger** — наш подход с 0x2c3909 + DSI_START правильный

### 6.6 Что нужно исправить

1. **Clock gating** — `writel(0xFFFFFFFF, MMSYS_CLK_CLR)` вместо покомандных записей
2. **DSI0 clock** — убедиться что бит 19 включается и остаётся включённым
3. **DISP_26M** — возможно нужно включить mm1 клоки тоже
4. **DSI_CON** — возможно нужно включить DSI_CON (биты 0+1 в `0x14013008`) перед trigger
