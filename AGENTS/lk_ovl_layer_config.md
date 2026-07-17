# OVL Layer Configuration — LK Decompile Analysis

## Source
`/workdir/git/heliog99/lk_unpack_v0/lk_a_lk.bin.c`

---

## 1. Component Index Mapping

From DTB strings and LK code:

| Index | Component | Base Address | DTB Node |
|-------|-----------|-------------|----------|
| 0 | MUTEX0 | 0x14001000 | disp_mutex@14001000 |
| 1 | RDMA0 | 0x14007000 | disp_rdma0@14007000 |
| 2 | COLOR0 | 0x14009000 | disp_color0@14009000 |
| 3 | CCORR0 | 0x1400b000 | disp_ccorr0@1400b000 |
| 4 | AAL0 | 0x1400c000 | disp_aal0@1400c000 |
| **5** | **OVL0** | **0x14005000** | **disp_ovl0@14005000** |
| **6** | **OVL0_2L** | **0x14006000** | **disp_ovl0_2l@14006000** |
| 7 | GAMMA0 | 0x1400d000 | disp_gamma0@1400d000 |
| 8 | POSTMASK0 | 0x1400e000 | disp_postmask0@1400e000 |
| 9 | DITHER0 | 0x1400f000 | disp_dither0@1400f000 |
| 10 | DSC_WRAP0 | 0x14012000 | disp_dsc_wrap0@14012000 |
| 16 | DSI0 | 0x14013000 | dsi@14013000 |
| 17 | DSI1 | 0x14016000 | dsi1 |
| 23 | WDMA0 | 0x14014000 | disp_wdma0@14014000 |

**OVL0 = component index 5, OVL0_2L = component index 6**

---

## 2. DAT_000ad2d8 — OVL Base Address

```c
int DAT_000ad2d8;  // ROM value = 0x14005000 (OVL0 base)
```

Used for indirect register access:
- For OVL0 (param_9=5): `offset = 0`, so `lVar1 = DAT_000ad2d8 + 0 = 0x14005000`
- For OVL0_2L (param_9=6): `offset = 0x14006000 - DAT_000ad2d8 = 0x1000`, so `lVar1 = 0x14006000`

---

## 3. FUN_00041b88 — OVL Layer Configuration (THE KEY FUNCTION)

```c
// Source: lk_a_lk.bin.c:46768-46997
// Purpose: Configure OVL layer (format, address, pitch, size)

undefined8 FUN_00041b88(
    ...,                     // various params
    int param_9,             // component ID: 5=OVL0, 6=OVL0_2L
    uint param_10,           // layer ID (0-3)
    uint param_11,           // color format flag (0=RGB, 1=YUV)
    int param_12,            // pixel format code (e.g., 0x1001020)
    int param_13,            // FRAMEBUFFER ADDRESS (physical)
    int param_14,            // stride multiplier
    int param_15,            // stride
    uint param_16,           // width
    uint param_17,           // offset X
    int param_18,            // offset Y
    uint param_19,           // src width
    uint param_20,           // src height
    uint param_21,           // flag
    undefined4 param_22,     // alpha
    uint param_23,           // flag
    byte param_24)           // flag
{
  // Validate component ID (must be 5 or 6)
  if (1 < param_9 - 5U) {
    // DEBUG_ASSERT_FAILED
  }

  // Select OVL base offset
  if (param_9 == 5) {
    iVar8 = 0;              // OVL0: offset 0
  } else {
    iVar8 = 0x14006000 - DAT_000ad2d8;  // OVL0_2L: offset 0x1000
  }

  // Calculate layer register base
  lVar1 = DAT_000ad2d8 + (iVar8 + param_10 * 0x20);

  // === WRITE OVL LAYER REGISTERS ===
  *(lVar1 + 0xc0) = 1;           // OVL_LAYER_CON = 1 (enable layer)
  *(lVar1 + 0x30) = uVar6;       // OVL_CFG (format config)
  *(lVar1 + 0x3c) = param_17 | param_18 << 0x10;  // OVL_OFFSET (X|Y)
  *(lVar1 + 0xf40) = param_13 + param_16 * param_15 + iVar7 * param_14;  // OVL_L0_ADDR
  *(lVar1 + 0x38) = param_19 | param_20 << 0x10;  // OVL_SRC_SIZE (W|H)
  *(lVar1 + 0x34) = param_22;    // OVL_PITCH
  *(lVar1 + 0x44) = uVar4;       // OVL_DMA_CTRL
}
```

### OVL Layer Register Map (from LK code)

| Offset | Register | Description |
|--------|----------|-------------|
| 0x030 | OVL_CFG | Format configuration (depends on pixel format) |
| 0x034 | OVL_PITCH | Layer pitch/stride |
| 0x038 | OVL_SRC_SIZE | Source size (width[15:0] | height[31:16]) |
| 0x03C | OVL_OFFSET | Offset (x[15:0] | y[31:16]) |
| 0x044 | OVL_DMA_CTRL | DMA control |
| 0x0C0 | OVL_LAYER_CON | Layer enable (bit 0) |
| **0xF40** | **OVL_L0_ADDR** | **Layer 0 framebuffer address** |
| 0xF60 | OVL_L1_ADDR | Layer 1 framebuffer address |
| 0xF80 | OVL_L2_ADDR | Layer 2 framebuffer address |
| 0xFA0 | OVL_L3_ADDR | Layer 3 framebuffer address |

### OVL_L0_ADDR Calculation
```
OVL_L0_ADDR = param_13 + (param_16 * param_15) + (iVar7 * param_14)
             = fb_addr + (width * stride) + (format_dependent * multiplier)
```

This is the physical address of the framebuffer that OVL reads from DRAM.

---

## 4. FUN_00041a30 — OVL0 Enable/Start

```c
// Source: lk_a_lk.bin.c:46700-46731
// Purpose: Enable OVL0 and configure source connection

undefined8 FUN_00041a30(..., int param_9)
{
  if (param_9 == 6) {
    uVar3 = 0x14006000 - DAT_000ad2d8;  // OVL0_2L offset
  } else {
    uVar3 = 0;                            // OVL0 offset
  }

  puVar1 = &DAT_ffff00001400500c + uVar3;  // OVL_EN
  lVar2 = DAT_000ad2d8 + uVar3;

  *puVar1 = *puVar1 | 1;              // OVL_EN |= bit0 (enable)
  *(lVar2 + 4) = 0x201e;              // OVL_CFG = 0x201e
  // MMSYS register |= 0x10000000     // Enable MMSYS clock/path
  *(lVar2 + 0x24) |= 1;              // OVL_SRC_CON |= bit0
  *(lVar2 + 0x24) |= 0x1000000;     // OVL_SRC_CON |= bit24
  *(lVar2 + 0x24) |= 0x4000000;     // OVL_SRC_CON |= bit26
  *puVar1 = *puVar1 | 0x400000;     // OVL_EN |= bit22
}
```

### OVL0 Enable Sequence
```
1. OVL_EN (0x1400500c) |= 0x000001     // Enable OVL
2. OVL_CFG (0x14005010) = 0x0000201e   // Configure format
3. MMSYS reg |= 0x10000000              // Enable MMSYS path
4. OVL_SRC_CON (0x14005024) |= 0x00000001  // Enable layer 0 source
5. OVL_SRC_CON (0x14005024) |= 0x01000000  // Enable layer 1 source
6. OVL_SRC_CON (0x14005024) |= 0x04000000  // Enable layer 2 source
7. OVL_EN (0x1400500c) |= 0x00400000   // Enable OVL trigger
```

---

## 5. FUN_00041f74 — OVL0 Reset/Disable

```c
// Source: lk_a_lk.bin.c:47023-47063
// Purpose: Reset OVL0 to initial state

undefined8 FUN_00041f74(..., int param_9)
{
  if (param_9 == 5) {
    _DAT_ffff00001400500c = 0;     // OVL0_EN = 0 (disable)
    uVar3 = 0;
  } else {
    uVar3 = 0x14006000 - DAT_000ad2d8;
    *(uVar3 + 0x1400500c) = 0;     // OVL0_2L_EN = 0 (disable)
  }

  lVar1 = DAT_000ad2d8 + uVar3;
  *(lVar1 + 0x14) = 1;            // Toggle reset
  *(lVar1 + 0x14) = 0;

  // Poll OVL_STATUS at offset 0x240 until ready
  if ((*(lVar1 + 0x240) & 3) == 0) {
    do {
      FUN_0005ae2c(10);            // 10us delay
      iVar4--;
    } while (iVar4 > 0 && (*(lVar1 + 0x240) & 3) == 0);
  }

  *(lVar1 + 4) = 0;               // OVL_CFG = 0
  *(lVar1 + 8) = 0;               // OVL_CLK_CFG = 0
}
```

---

## 6. FUN_00041644 — Display Pipeline Enable (Main Entry Point)

```c
// Source: lk_a_lk.bin.c:46560-46625
// Purpose: Enable full display pipeline

undefined8 FUN_00041644(..., long param_9)
{
  // Get pipeline component list
  piVar2 = FUN_00042de8(*(uint*)(param_9 + 0xe0));
  // → returns &DAT_000ad650 + pipeline_id * 10

  uVar1 = FUN_00042870(*(uint*)(param_9 + 0xe0));
  // → returns number of components in pipeline

  // Step 1: Pipeline routing reconfigure
  FUN_00043610(..., *(uint*)(param_9 + 0xe0));

  // Step 2: Pipeline validation
  FUN_00043b4c(..., *(uint*)(param_9 + 8), *(uint*)(param_9 + 0xe0));

  // Step 3: Debug dump
  FUN_00040064(..., 0);
  FUN_00040064(..., 1);

  // Step 4: Enable each component via vtable
  if (uVar1 >= 1) {
    for (i = 0; i < uVar1; i++) {
      // Debug dump for this component
      FUN_00040064(..., piVar2[i]);

      // Enable DSI if this is the DSI component
      if (i < 0x18 && ((1 << i) & 0xc10000) != 0) {
        if (DAT_00108eb0[i] != 0) {
          pcVar3 = DAT_00108eb0[i]->start;
          if (pcVar3 != Reset) {
            (*pcVar3)(i, 1);    // Call component start function
          }
        }
      }
    }
  }

  // Step 5: Final debug dump
  FUN_00040064(..., 0);
  FUN_00040064(..., 1);

  // Step 6: Enable MUTEX for each component
  if (uVar1 >= 1) {
    for (i = 0; i < uVar1; i++) {
      FUN_0003f93c(..., piVar2[i]);  // Enable MUTEX for this component
    }
  }

  // Step 7: Enable MUTEX0 and MUTEX1
  FUN_0003f93c(..., 0);  // MUTEX0
  FUN_0003f93c(..., 1);  // MUTEX1
}
```

---

## 7. DAT_00108eb0 — Driver Vtable (Component Array)

```c
// Source: lk_a_lk.bin.c:45777-45805
// Initialization: FUN_00040694()

void FUN_00040694(void)
{
  DAT_00108eb0[0]  = DAT_000acfe8;  // Component 0 driver
  DAT_00108eb0[1]  = DAT_000ad008;  // Component 1 driver
  DAT_00108eb0[2]  = DAT_000ad028;  // Component 2 driver
  DAT_00108eb0[3]  = DAT_000ad048;  // Component 3 driver
  DAT_00108eb0[4]  = DAT_000ad068;  // Component 4 driver
  DAT_00108eb0[5]  = DAT_000ad088;  // Component 5 driver (OVL0?)
  DAT_00108eb0[6]  = DAT_000ad0a8;  // Component 6 driver (OVL0_2L?)
  DAT_00108eb0[7]  = DAT_000ad0c8;  // Component 7 driver
  DAT_00108eb0[8]  = DAT_000ad0e8;  // Component 8 driver
  DAT_00108eb0[9]  = DAT_000ad108;  // Component 9 driver
  DAT_00108eb0[10] = DAT_000ad128;  // Component 10 driver
  DAT_00108eb0[11] = DAT_000ad148;  // Component 11 driver
  DAT_00108eb0[12] = DAT_000ad168;  // Component 12 driver
  DAT_00108eb0[13] = DAT_000ad188;  // Component 13 driver
  DAT_00108eb0[14] = DAT_000ad1a8;  // Component 14 driver (DSI0)
  DAT_00108eb0[15] = DAT_000ad1c8;  // Component 15 driver
  DAT_00108eb0[16] = DAT_000ad1e8;  // Component 16 driver
  DAT_00108eb0[17] = DAT_000ad208;  // Component 17 driver
  DAT_00108eb0[18] = DAT_000ad228;  // Component 18 driver
  DAT_00108eb0[19] = DAT_000ad248;  // Component 19 driver
  DAT_00108eb0[20] = DAT_000ad268;  // Component 20 driver
  DAT_00108eb0[21] = DAT_000ad288;  // Component 21 driver
  DAT_00108eb0[22] = DAT_000ad2a8;  // Component 22 driver
  DAT_00108eb0[23] = DAT_000ad2c8;  // Component 23 driver
}
```

### Vtable Entry Layout (0x20 bytes each)

| Offset | Function | Description |
|--------|----------|-------------|
| 0x00 | name_ptr | Pointer to component name string |
| 0x08 | ??? | Unknown |
| 0x10 | ??? | Unknown |
| 0x18 | init | Component initialization |
| 0x20 | disable | Component disable |
| 0x28 | enable | Component enable |
| 0x30 | ??? | Unknown |
| 0x38 | ??? | Unknown |
| 0x40 | ??? | Unknown |
| 0x48 | ??? | Unknown |
| 0x50 | ??? | Unknown |
| 0x58 | ??? | Unknown |
| 0x60 | start | Component start (called in pipeline enable) |
| 0x68 | ??? | Unknown |
| 0x70 | ??? | Unknown |
| 0x78 | ??? | Unknown |
| 0x80 | ??? | Unknown |
| 0x88 | ??? | Unknown |
| 0x90 | ??? | Unknown (called in other paths) |

---

## 8. FUN_0003f93c — MUTEX Enable (NOT OVL config!)

```c
// Source: lk_a_lk.bin.c:45333-45488
// Purpose: Enable MUTEX for a specific component

long FUN_0003f93c(..., int param_9)
{
  switch(param_9) {
  case 0:   // Debug: dump MMSYS registers as formatted strings
    // Reads MMSYS config registers (0x14000xxx)
    // Formats as "0x%03x" strings
    // Multiple loops over different register ranges
    break;
  case 1:   // Debug: dump more MMSYS registers
    break;
  case 5:   // OVL0
  case 6:   // OVL0_2L
    FUN_0004222c(..., param_9);
    break;
  case 7:   // Unknown
    FUN_000444e8();
    break;
  case 0xd: // Unknown
    FUN_00043f04();
    break;
  case 0x10: // DSI0
  case 0x16: // Unknown
  case 0x17: // Unknown
    FUN_0003a4bc(param_9, 1);
    break;
  }
}
```

**FUN_0003f93c is NOT the OVL config function** — it's a debug dump / MUTEX enable function. The actual OVL layer config is FUN_00041b88.

---

## 9. Complete Display Pipeline Sequence (from LK)

```
1. FUN_00041644 — Main pipeline enable
   ├── FUN_00043610 — Pipeline routing reconfigure
   ├── FUN_00043b4c — Pipeline validation
   ├── For each component:
   │   ├── FUN_00040064 — Debug dump
   │   └── vtable[+0x60](component_id, 1) — Component start
   │       ├── For OVL0 (id=5): FUN_00041a30
   │       │   ├── OVL_EN |= 1
   │       │   ├── OVL_CFG = 0x201e
   │       │   ├── MMSYS |= 0x10000000
   │       │   ├── OVL_SRC_CON |= 0x05000001
   │       │   └── OVL_EN |= 0x400000
   │       └── For DSI0 (id=16): DSI enable
   ├── FUN_00040064(0), FUN_00040064(1) — Final debug dump
   └── For each component: FUN_0003f93c — MUTEX enable

2. OVL Layer Config (called separately):
   FUN_00041b88(component_id=5, layer=0, fb_addr, width, height, ...)
   ├── OVL_LAYER_CON (0x140050c0) = 1
   ├── OVL_CFG (0x14005030) = format_config
   ├── OVL_OFFSET (0x1400503c) = x | (y << 16)
   ├── OVL_L0_ADDR (0x14005f40) = fb_address
   ├── OVL_SRC_SIZE (0x14005038) = width | (height << 16)
   ├── OVL_PITCH (0x14005034) = stride
   └── OVL_DMA_CTRL (0x14005044) = dma_config

3. Display Refresh:
   DSI 0x2C (Write Memory Start) via CMDQ
   └── FUN_0003f1bc → FUN_0003e108
       ├── Write 0x2c3909 to DSI_CMDQ (0x14013d00)
       ├── Write 0x8001 to DSI_CMDQ_SIZE (0x14013060)
       └── Toggle DSI_START (0x14013000)
```

---

## 10. What We Need for U-Boot

To display text on screen from U-Boot, we need:

### Step 1: OVL Layer Config
```c
// Set OVL layer 0 to read from framebuffer
writel(1, 0x140050c0);           // OVL_LAYER_CON = 1 (enable layer)
writel(format_config, 0x14005030); // OVL_CFG
writel(x | (y << 16), 0x1400503c); // OVL_OFFSET
writel(fb_addr, 0x14005f40);     // OVL_L0_ADDR ← THIS IS THE KEY!
writel(width | (height << 16), 0x14005038); // OVL_SRC_SIZE
writel(stride, 0x14005034);      // OVL_PITCH
writel(dma_config, 0x14005044);  // OVL_DMA_CTRL
```

### Step 2: OVL Enable
```c
writel(readl(0x1400500c) | 1, 0x1400500c);     // OVL_EN |= 1
writel(0x201e, 0x14005010);                      // OVL_CFG
writel(readl(0x14005024) | 0x05000001, 0x14005024); // OVL_SRC_CON
writel(readl(0x1400500c) | 0x400000, 0x1400500c);   // OVL_EN |= trigger
```

### Step 3: MUTEX Enable
```c
// Enable MUTEX0 for OVL0
writel(1, 0x14001020);  // MUTEX0_EN = 1
```

### Step 4: DSI Refresh
```c
writel(0x2c3909, 0x14013d00);  // DSI_CMDQ = Write Memory Start
writel(0x8001, 0x14013060);    // DSI_CMDQ_SIZE = trigger
writel(0, 0x14013000);         // DSI_START = 0
writel(1, 0x14013000);         // DSI_START = 1
```
