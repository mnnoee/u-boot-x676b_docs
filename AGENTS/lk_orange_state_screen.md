# LK Orange/Red State Warning Screen — Complete Analysis

## Source
`/workdir/git/heliog99/lk_unpack_v0/lk_a_lk.bin.c`

---

## Orange State Screen Function: `FUN_0004b1e4` (line 54331)

```c
undefined8 FUN_0004b1e4(...) {
  if (DAT_000add28 == 1 || DAT_000add28 == 3) {
    // RED STATE
    FUN_000362b4(...);                          // clear screen (black)
    iVar1 = FUN_000362a8();                     // get screen height
    auVar4 = FUN_00036194(iVar1 >> 1, 0);      // cursor to vertical center, column 0
    FUN_000360d0("Red State");                  // render title
    FUN_000360d0("OS Not Being Verified...");   // render subtitle
    pcVar2 = "Device Cannot Boot. Enter Repair...";
  }
  else if (DAT_000add28 == 2) {
    // ORANGE STATE
    FUN_000362b4(...);                          // clear screen (black)
    iVar1 = FUN_000362a8();                     // get screen height
    auVar4 = FUN_00036194(iVar1 >> 1, 0);      // cursor to vertical center, column 0
    FUN_000360d0("Orange State");               // render title
    FUN_000360d0("OS Not Being Verified Or Custom OS");
  }
  FUN_000360d0(pcVar2);                         // render description
  FUN_000360d0(pcVar3);                         // render extra text
  FUN_00036614();                               // kick watchdog
  FUN_0005ae2c(5000000);                        // sleep 5 seconds
  FUN_00036614();                               // kick watchdog again
  return 0;
}
```

**State variable:** `DAT_000add28`
- `0` = no warning
- `1` = Red State (verification failed)
- `2` = Orange State (device unlocked)
- `3` = Red State (custom OS)

**Strings** (lines 3332-3335):
- `s_Orange_State_0009d4b3` — "Orange State"
- `s_OS_Not_Being_Verified_Or_Custom_O_0009d4c2` — "OS Not Being Verified Or Custom O..."
- `s_Red_State_0009d500` — "Red State"
- `s_Device_Cannot_Boot._Enter_Repair_0009d50c` — "Device Cannot Boot. Enter Repair..."

---

## Text Rendering Pipeline

### FUN_000360d0 (line 39979) — STRING RENDERER
```c
void FUN_000360d0(..., char *param_9) {
  if (_DAT_00108010 != 0) {             // graphics context initialized?
    FUN_0005b950(abStack_150, param_9, ...);  // vsprintf
    uVar2 = FUN_0005d3ec(abStack_150);        // strlen
    do {
      FUN_000359c4(*pbVar3);            // render one character
      pbVar3++;
    } while (uVar1 != 0);
  }
}
```

### FUN_000359c4 (line 39669) — CHARACTER RENDERER (manages cursor)
- Character cells: **8 pixels wide x 16 pixels tall**
- Handles BS, TAB, LF, CR control characters
- For printable chars: calls `FUN_00035d48(cursor_x * 8, cursor_y * 16, char)`

### FUN_00035d48 (line 39824) — PIXEL WRITER (glyph bitmap to framebuffer)
```c
void FUN_00035d48(int param_1, int param_2, byte param_3) {
  // param_1 = X pixel position
  // param_2 = Y pixel position
  // param_3 = character code
  
  // Font bitmap: DAT_00095a44 + char * 16 (16 bytes per glyph)
  // For each of 16 rows:
  //   For each of 8 pixel pairs (2 pixels per byte, high/low nibble):
  //     pixel = color_lut[nibble] & (fg ^ bg) ^ bg | 0xFF000000
  //     write 4 bytes to backbuffer
  //   Advance to next row (stride = bpp * width)
}
```

### FUN_00044ed8 (line 49070) — FRAMEBUFFER FLUSH
```c
ulong FUN_00044ed8(...) {
  if (DAT_0010a260 & 1) {                // framebuffer initialized?
    FUN_000037dc(fb_va, fb_size);         // DC_CVAC cache clean loop
    FUN_00045bc8(...);                    // trigger DISP refresh
  }
}
```

### FUN_000037dc (line 4959) — DC_CVAC CACHE CLEAN
```c
void FUN_000037dc(ulong param_1, long param_2) {
  uVar1 = param_1 & ~0x3F;               // align to 64 bytes
  do {
    DC_CVAC(uVar1);                      // clean + invalidate data cache line
    uVar1 += 0x40;                       // next cache line (64 bytes)
  } while (uVar1 < param_1 + param_2);
  DataSynchronizationBarrier(3,3,0);     // DSB
}
```

---

## Key Globals

| Global | Address | Purpose |
|--------|---------|---------|
| `_DAT_00108010` | `0x108010` | Display info struct pointer |
| `_DAT_00108038` | `0x108038` | Framebuffer VA for glyph rendering |
| `_DAT_00108008` | `0x108008` | Cursor X (column) |
| `_DAT_00108018` | `0x108018` | Cursor Y (row) |
| `_DAT_0010801c` | `0x10801c` | Foreground pixel color |
| `_DAT_00108020` | `0x108020` | Background color (0 = black) |
| `_DAT_00108024` | `0x108024` | XOR mask (fg ^ bg) |
| `_DAT_00108048` | `0x108048` | Screen height in char cells |
| `_DAT_00108050` | `0x108050` | Screen width in char cells |
| `_DAT_0010a258` | `0x10a258` | Framebuffer PA |
| `_DAT_0010a268` | `0x10a268` | Framebuffer VA (from vm_framebuffer) |
| `_DAT_0010a270` | `0x10a270` | Framebuffer size (bytes) |
| `DAT_0010a260` | `0x10a260` | FB initialized flag (1=yes) |
| `DAT_000add28` | `0xadd28` | Orange/Red state variable |

---

## Display Context Structure (`_DAT_00108010`)

Initialized at `FUN_00036300` (line 40094):

| Offset | Field | Description |
|--------|-------|-------------|
| +0x00 | fb_ptr | Framebuffer VA pointer |
| +0x08 | fb_size | Total framebuffer size (bytes) |
| +0x10 | bpp_bytes | Bytes per pixel (4 for 32bpp) |
| +0x18 | height | Display height |
| +0x30 | bpp_mode | BPP mode index (3 = 32bpp ARGB8888) |
| +0x34 | width | Display width |

---

## Color Settings

Default foreground for 32bpp mode: `0xb6b6b6b6` (light gray)
- Set at init (line 40178)
- Can be changed via `FUN_000361d0(R, G, B)` (line 40028)
- For 32bpp: `_DAT_0010801c = (R << 16) | (G << 8) | B`

---

## Font Data

- **`DAT_00095a44`** — Glyph bitmap table (256 chars × 16 bytes each, 8×16 bitmap)
- **`DAT_00095944/948/94c/950`** — 4-entry ARGB color lookup tables for 32bpp mode

---

## Key Conclusion for U-Boot

The LK renders text by:
1. Writing pixels directly to the framebuffer VA (`_DAT_00108038`)
2. Doing a DC_CVAC cache clean (flush CPU cache to RAM)
3. Triggering DISP refresh

**There is NO separate OVL layer for the orange state text.** It overwrites the same framebuffer that contains the boot logo. The key to making updates visible is the **cache clean (DC_CVAC) + DISP refresh trigger** — without this, pixel writes stay in CPU cache and never reach the display.

For U-Boot, the equivalent would be:
1. Write pixels to framebuffer address (0xfeabe000)
2. Flush D-cache: `flush_dcache_range(fb_addr, fb_size)`
3. Trigger display refresh (if the display pipeline is running)
