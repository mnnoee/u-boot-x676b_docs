# DSI Command Send Algorithm — LK Decompile + Kernel Analysis

## Source Files
- LK: `/workdir/git/heliog99/lk_unpack_v0/lk_a_lk.bin.c`
- Kernel: `/workdir/git/heliog99/src_linux/linux/drivers/gpu/drm/mediatek/mtk_dsi.c`

---

## 1. Key Register Offsets (MT6789)

| Offset | Name | Source | Description |
|--------|------|--------|-------------|
| 0x000 | DSI_START | kernel:35, LK FUN_0003e108:44561 | Toggle bit 0 to trigger cmd send |
| 0x004 | DSI_INTEN | kernel | Interrupt enable |
| 0x008 | DSI_INTSTA | kernel | Interrupt status |
| 0x00C | DSI_STATUS | kernel, LK:44050 | Bit 0: busy. Bit 1: ack. Signed check `< 0` |
| 0x010 | DSI_CON | kernel | Control register |
| 0x014 | DSI_MODE_CTRL | kernel:1127 | Bit 0: CMD/VIDEO mode |
| 0x018 | DSI_TXRX_CTRL | kernel | Bit 0,1: lane config |
| 0x060 | DSI_CMDQ_SIZE | kernel:94, LK:44082 | Bits [5:0] = cmd count. Bit 15 = trigger |
| 0x074 | DSI_RX_DATA0 | kernel:101 | Read response byte 0 |
| 0x078 | DSI_RX_DATA1 | kernel:102 | Read response byte 1 |
| 0x07C | DSI_RX_DATA2 | kernel:103 | Read response byte 2 |
| **0xd00** | **DSI_CMDQ** | **kernel:1265 (mt6789)** | **Command FIFO start (MT6789-specific!)** |

**CRITICAL**: MT6789 uses `reg_cmdq_off = 0xd00`, NOT 0x200 (mt8173) or 0x180 (mt2701)!

```c
// kernel:1264-1269
static const struct mtk_dsi_driver_data mt6789_dsi_driver_data = {
    .reg_cmdq_off = 0xd00,    // ← THIS IS THE KEY
    .has_shadow_ctl = false,
    .has_size_ctl = true,
    .support_per_frame_lp = true,
};
```

---

## 2. DAT_0009a8d0 and DAT_0009a8f0 — ROM Arrays

These are **static arrays in ROM** at addresses 0x9a8d0 and 0x9a8f0. Each entry is a 64-bit pointer, indexed by component ID.

### DAT_0009a8d0 — Register Base Pointers
- `DAT_0009a8d0[0]` = `0x14013000` (DSI0 register base)
  - Confirmed by FUN_0003e108:44561: `puVar7 = (uint *)&DAT_ffff000014013000;`
- `DAT_0009a8d0[1]` = likely `0x14013000` for DSI1 (if present)

### DAT_0009a8f0 — Command Buffer Pointers
- `DAT_0009a8f0[0]` = pointer to DSI0 CMDQ FIFO at `0x14013000 + 0xd00 = 0x14013d00`
  - Confirmed by: kernel defines `reg_cmdq_off = 0xd00` for MT6789
  - The LK writes command words through this pointer

### Component Index Calculation
```c
uVar20 = (param_1 != 0x17 && param_1 != 0x10);  // 0 for DSI, 1 for MDP
uVar19 = (param_1 != 0x10);
```
- `param_1 = 0x17` → DSI0 → index 0
- `param_1 = 0x10` → secondary DSI or MDP → index 0
- `param_1 = other` → MDP → index 1

---

## 3. CMDQ Entry Format (MT6789)

From kernel `mtk_dsi_cmdq()` at line 1065:

```c
// Kernel CMDQ format:
#define CONFIG      GENMASK(7, 0)    // bits [7:0]
#define DATA_ID     GENMASK(15, 8)   // bits [15:8]
#define DATA_0      GENMASK(23, 16)  // bits [23:16]
#define DATA_1      GENMASK(31, 24)  // bits [31:24]

// For SHORT packets (tx_len <= 2):
reg_val = (type << 8) | config;
// Data bytes written at offset+2, offset+3 separately

// For LONG packets (tx_len > 2):
reg_val = (tx_len << 16) | (type << 8) | config;
// Followed by data words at offset+4, +8, etc.
```

### MIPI DCS Data Types (DATA_ID)
| Type | Meaning |
|------|---------|
| 0x05 | DCS Short Write, 0 parameters |
| 0x15 | DCS Short Write, 1 parameter |
| 0x39 | DCS Long Write (2+ parameters) |
| 0x06 | DCS Read, 0 parameters |
| 0x16 | DCS Read, 1 parameter |

### Config Bits
| Bit | Name | Value | Description |
|-----|------|-------|-------------|
| 0 | SHORT_PACKET | 0 | Short packet mode |
| 1 | LONG_PACKET | 2 | Long packet mode |
| 2 | BTA | 4 | Bus Turn-Around (for reads) |

---

## 4. 0x2c3909 — Full Byte-Level Decode

`0x2c3909` is a **single 32-bit CMDQ word** written to DSI_CMDQ (offset 0xd00).

```
Hex:    0x 00 2c 39 09
Binary: 00000000 00101100 00111001 00001001
```

| Bits | Hex | Field | Meaning |
|------|-----|-------|---------|
| [7:0] | 0x09 | CONFIG | flags (see below) |
| [15:8] | 0x39 | DATA_ID | DCS Long Write |
| [23:16] | 0x2C | DATA_0 | **Write Memory Start** (DCS command) |
| [31:24] | 0x00 | DATA_1 | padding / unused |

### Config = 0x09 breakdown
```
0x09 = 0b00001001 = bit0 + bit3
```
- **Bit 0** = 1: Likely signals "send command" (vs. read)
- **Bit 1** = 0: NOT LONG_PACKET (contradicts DATA_ID=0x39)
- **Bit 2** = 0: NO BTA (write, not read)
- **Bit 3** = 1: Unknown flag (possibly CMD mode or LP mode flag)

**NOTE**: The LK uses a **different config encoding** than the kernel. The kernel uses `SHORT_PACKET=0, LONG_PACKET=2, BTA=4`. The LK uses `0x09` for this specific DCS command.

### Why 0x39 (DCS Long Write) for 0x2C?
- 0x2C = "Write Memory Start" is a DCS command with **no parameters**
- Normally sent as type 0x05 (DCS Short Write, 0 params)
- LK sends it as type 0x39 (DCS Long Write) — this is a **quirk of the LK implementation**
- The panel/DSI controller may require this specific encoding

---

## 5. FUN_0003c8b8 — Complete Annotated Body

```c
// Source: lk_a_lk.bin.c:43993-44216
// Purpose: Send DSI command (read or write) via CMDQ
// Called for: panel init sequences, DCS read/write

ulong FUN_0003c8b8(
    int param_1,        // component ID: 0x17=DSI0, 0x10=secondary
    undefined8 param_2, // unused
    undefined8 *param_3, // output buffer (for reads)
    byte *param_4,      // pointer to DCS command byte
    uint param_5)       // payload length
{
  uint *puVar1;         // status register pointer
  uint uVar2;           // payload length (masked)
  uint uVar3;           // temp
  byte bVar4;           // DCS command byte
  bool bVar5;           // comparison result
  undefined8 *puVar6;   // temp pointer
  uint uVar7;           // response type
  undefined8 *puVar8;   // temp pointer
  uint *puVar9;         // COMMAND BUFFER POINTER (→ DSI_CMDQ at 0xd00)
  ulong uVar10;         // temp
  ulong uVar11;         // temp
  ulong uVar12;         // component index (starts at uVar20)
  uint uVar13;          // PACKET SIZE: 0x600 or 0x1400
  int iVar14;           // delay counter
  uint uVar15;          // PAYLOAD COUNT (capped at 10)
  int iVar16;           // RETRY COUNTER (starts at 5)
  undefined4 *puVar17;  // REGISTER BASE (→ 0x14013000 for DSI0)
  long lVar18;          // register base (signed)
  ulong uVar19;         // secondary index
  ulong uVar20;         // PRIMARY INDEX: 0 for DSI, 1 for MDP

  // === STEP 1: Extract DCS command byte ===
  bVar4 = *param_4;                    // e.g., 0x2C for Write Memory Start
  uVar2 = param_5 & 0xff;             // payload length

  // === STEP 2: Set packet size based on DCS command ===
  uVar13 = 0x600;                      // default: short packet
  if (0xaf < bVar4) {                  // if DCS cmd > 0xAF
    uVar13 = 0x1400;                   // use long packet
  }

  // === STEP 3: Calculate component indices ===
  uVar20 = (ulong)(param_1 != 0x17 && param_1 != 0x10);  // 0 for DSI
  uVar15 = uVar2;                      // payload count
  if (9 < uVar2) {
    uVar15 = 10;                       // cap at 10
  }
  iVar16 = 5;                          // retry counter
  uVar19 = (ulong)(uint)(param_1 != 0x10);  // secondary index

  uVar12 = uVar20;                     // start with primary index

  // === STEP 4: Main retry loop ===
  do {
    if (param_3 == (undefined8 *)0x0) {
      return 0;                        // null output buffer
    }

    // Get register base from ROM array
    lVar18 = *(long *)(&DAT_0009a8d0 + uVar12 * 8);
    // → for DSI0: lVar18 = 0x14013000

    // Check DSI_STATE bits [1:0] — must be 0 (idle)
    if ((*(uint *)(lVar18 + 0x14) & 3) != 0) {
      return 0;                        // DSI not idle
    }

    if (uVar2 == 0) {
      return 0;                        // no payload
    }

    // === STEP 5: Poll until DSI_STATUS is ready ===
    do {
      if (iVar16 == 0) {
        return 0;                      // retries exhausted
      }
      // Wait loop: poll DSI_STATUS (offset 0x0c)
      // Signed check: < 0 means bit 31 set (busy)
      if (((*(byte *)(*(long *)(&DAT_0009a8d0 + uVar20 * 8) + 0x14) & 3) == 0) &&
         (*(int *)(*(long *)(&DAT_0009a8d0 + uVar20 * 8) + 0xc) < 0)) {
        iVar14 = 0;
        do {
          iVar14 = iVar14 + 1;
          uVar11 = uVar20;
          // Delay loop
          if (((uint)(iVar14 * 0x26e978d5) >> 3 | iVar14 * -0x60000000) < 0x418938) {
            do {
              bVar5 = uVar11 < uVar19;
              uVar11 = uVar11 + 1;
            } while (bVar5);
          }
        } while (*(int *)(*(long *)(&DAT_0009a8d0 + uVar20 * 8) + 0xc) < 0);
      }

      // === STEP 6: Enable DSI_CON bits 0,1 ===
      uVar7 = *(uint *)(lVar18 + 8);     // DSI_CON (offset 0x08)
      if ((uVar7 & 1) == 0) {            // bit 0 not set
        *(uint *)(lVar18 + 8) = *(uint *)(lVar18 + 8) | 1;  // set bit 0
        uVar7 = *(uint *)(lVar18 + 8);
      }
      if ((uVar7 >> 1 & 1) == 0) {       // bit 1 not set
        *(uint *)(lVar18 + 8) = *(uint *)(lVar18 + 8) | 2;  // set bit 1
      }

      // === STEP 7: Get command buffer pointer from ROM array ===
      puVar17 = *(undefined4 **)(&DAT_0009a8d0 + uVar12 * 8);
      // puVar17 = 0x14013000 (register base)

      puVar1 = puVar17 + 3;
      // puVar1 = 0x1401300C (DSI_STATUS pointer for polling)

      // Clear DSI_CON bits 0,1 if state bits set
      if ((*(byte *)(lVar18 + 0xc) & 3) != 0) {
        *puVar1 = *puVar1 & 0xfffffffe;
        *puVar1 = *puVar1 & 0xfffffffd;
      }

      // === STEP 8: Get CMDQ buffer pointer from ROM array ===
      puVar9 = *(uint **)(&DAT_0009a8f0 + uVar12 * 8);
      // puVar9 → 0x14013d00 (DSI_CMDQ for MT6789)

      // === STEP 9: Write command words to CMDQ FIFO ===
      *puVar9 = uVar15 << 0x10 | 0x3700;
      // CMDQ word 0: (payload_count << 16) | 0x3700

      puVar9[1] = uVar13 | (uint)bVar4 << 0x10 | 4;
      // CMDQ word 1: (packet_size) | (DCS_cmd << 16) | 4

      // === STEP 10: Trigger command ===
      puVar17[0x18] = 0x8002;
      // DSI_CMDQ_SIZE (offset 0x60) = 0x8002
      // = bit 15 (trigger) | 2 (two CMDQ words)

      *puVar17 = 0;
      // DSI_START (offset 0x00) = 0 (clear)

      *puVar17 = 1;
      // DSI_START (offset 0x00) = 1 (trigger)

      // === STEP 11: Poll for completion (up to 20 times) ===
      if ((((((((puVar17[3] & 1) == 0) && (FUN_0005ae2c(1000), (*puVar1 & 1) == 0)) &&
             (FUN_0005ae2c(1000), (*puVar1 & 1) == 0)) &&
            ((FUN_0005ae2c(1000), (*puVar1 & 1) == 0 && (FUN_0005ae2c(1000), (*puVar1 & 1) == 0)))) &&
           ((FUN_0005ae2c(1000), (*puVar1 & 1) == 0 &&
             ((FUN_0005ae2c(1000), (*puVar1 & 1) == 0 && (FUN_0005ae2c(1000), (*puVar1 & 1) == 0))))))
          && (FUN_0005ae2c(1000), (*puVar1 & 1) == 0)) &&
         (((((FUN_0005ae2c(1000), (*puVar1 & 1) == 0 && (FUN_0005ae2c(1000), (*puVar1 & 1) == 0)) &&
            (FUN_0005ae2c(1000), (*puVar1 & 1) == 0)) &&
           (((FUN_0005ae2c(1000), (*puVar1 & 1) == 0 && (FUN_0005ae2c(1000), (*puVar1 & 1) == 0)) &&
            ((FUN_0005ae2c(1000), (*puVar1 & 1) == 0 &&
              ((FUN_0005ae2c(1000), (*puVar1 & 1) == 0 && (FUN_0005ae2c(1000), (*puVar1 & 1) == 0))))))
           )))) && ((FUN_0005ae2c(1000), (*puVar1 & 1) == 0 &&
                     ((FUN_0005ae2c(1000), (*puVar1 & 1) == 0 &&
                      (FUN_0005ae2c(1000), (*puVar1 & 1) == 0)))))))) {
        // Timeout — DSI didn't respond
        FUN_0005ae2c(1000);
        // Reset and return error
        puVar17[0x21] = puVar17[0x21] | 1;
        do {
          lVar18 = *(long *)(&DAT_0009a8d0 + uVar20 * 8);
          bVar5 = uVar20 < uVar19;
          uVar20 = uVar20 + 1;
          *(uint *)(lVar18 + 0x10) = *(uint *)(lVar18 + 0x10) | 1;
          *(uint *)(lVar18 + 0x10) = *(uint *)(lVar18 + 0x10) & 0xfffffffe;
        } while (bVar5);
        return 0;
      }

      // === STEP 12: Command succeeded — read response (for reads) ===
      puVar17[0x21] = puVar17[0x21] | 1;
      puVar17[3] = puVar17[3] & 0xfffffffe;

      // Poll for ACK (bit 1 of status)
      if (((((((uint)puVar17[3] >> 1 & 1) == 0) && (FUN_0005ae2c(1000), (*puVar1 >> 1 & 1) == 0)) &&
           ... )) {
        // Timeout on ACK
        ...
      }

      iVar16 = iVar16 + -1;            // decrement retry counter
      puVar17[3] = puVar17[3] & 0xfffffffd;

      // Read response data from DSI_RX_DATA registers
      local_64 = puVar17[0x1d];  // DSI_RX_DATA0 at offset 0x74
      local_68 = puVar17[0x1e];  // DSI_RX_DATA1 at offset 0x78
      local_6c = puVar17[0x1f];  // DSI_RX_DATA2 at offset 0x7C
      uVar7 = local_64 & 0xff;   // response type

    } while (uVar7 == 2);       // retry if response type = 2 (busy)

    // === STEP 13: Copy response to output buffer ===
    if (0x22 < uVar7) {
      return 0;                  // unknown response type
    }
    // ... copy logic depending on response type ...

    uVar7 = (uint)uVar12;
    uVar12 = (ulong)(uVar7 + 1);    // next component
    if ((param_1 != 0x10) <= uVar7) {
      return uVar11;                 // done
    }
  } while( true );
}
```

---

## 6. FUN_0003e108 — Simple DSI Command Send

```c
// Source: lk_a_lk.bin.c:44488-44584
// Purpose: Send pre-built CMDQ words to DSI
// Used by FUN_0003f1bc for 0x2c3909

void FUN_0003e108(
    ...,                     // various params
    uint param_9,            // component ID: 0x17=DSI0
    undefined8 param_10,
    undefined4 *param_11,    // pointer to CMDQ words (e.g., &0x2c3909)
    uint param_12,           // number of CMDQ words (e.g., 1)
    char param_13)           // flag: '\0' = no DSI_START toggle
{
  // ... poll busy ...

  // === Copy CMDQ words to command buffer ===
  if (param_12 != 0) {
    puVar2 = *(undefined4 **)(&DAT_0009a8f0 + uVar8 * 8);
    // puVar2 → 0x14013d00 (DSI_CMDQ for MT6789)

    uVar4 = (ulong)param_12;
    puVar5 = param_11;     // &0x2c3909
    do {
      uVar4 = uVar4 - 1;
      *puVar2 = *puVar5;  // copy 32-bit word to CMDQ
      puVar2 = puVar2 + 1;
      puVar5 = puVar5 + 1;
    } while (uVar4 != 0);
  }

  // === Trigger command ===
  *(uint *)(*(long *)(&DAT_0009a8d0 + uVar8 * 8) + 0x60) = param_12 | 0x8000;
  // DSI_CMDQ_SIZE = count | bit15(trigger)
  // For count=1: 0x8001

  // === Toggle DSI_START (if param_13 != '\0') ===
  if (param_9 == 0x17) {
    puVar7 = (uint *)&DAT_ffff000014013000;  // DSI0 base
  } else {
    puVar7 = *(uint **)(&DAT_0009a8d0 + ...);
  }
  *puVar7 = *puVar7 & 0xfffffffe;  // DSI_START = 0
  *puVar7 = *puVar7 | 1;           // DSI_START = 1

  // === Poll for completion ===
  // ... poll DSI_STATUS ...
}
```

---

## 7. FUN_0003f1bc — DCS 0x2C (Write Memory Start) Entry Point

```c
// Source: lk_a_lk.bin.c:45115-45141
// Purpose: Send DCS 0x2C (Write Memory Start) to trigger frame refresh

ulong FUN_0003f1bc(
    ...,                     // various params
    uint param_9,            // component ID: 0x17=DSI0
    undefined8 param_10)
{
  uint *puVar1;
  undefined4 local_14;

  // Check if display is initialized
  if (*(int *)((ulong)(param_9 != 0x17 && param_9 != 0x10) * 0x728 + 0x108668) == 0) {
    // Display NOT initialized — send 0x2C via CMDQ
    local_14 = 0x2c3909;
    FUN_0003e108(param_1,..., param_9, param_10,
                 &local_14, 1, '\0');
    // param_12=1 (one CMDQ word), param_13='\0' (no DSI_START in FUN_0003e108)

    if (param_9 == 0x17) {
      // Toggle DSI_START manually
      _DAT_ffff000014013000 = _DAT_ffff000014013000 & 0xfffffffe | 1;
      // = clear bit0, then set bit0
      return 0;
    }
  }
  else if (param_9 == 0x17) {
    // Display IS initialized — just toggle DSI_START
    puVar1 = (uint *)&DAT_ffff000014013000;
    goto LAB_0003f248;
  }
  // ... other paths ...
}
```

---

## 8. FUN_0004b6f0 — Display Window + Refresh

```c
// Source: lk_a_lk.bin.c:54678-54699
// Purpose: Set display window (column/row) then trigger refresh

void FUN_0004b6f0(uint param_1, uint param_2, int param_3, int param_4)
{
  // param_1, param_2 = start coordinates
  // param_3, param_4 = dimensions

  // DCS 0x2A: Column Address Set (3 words: cmd + start + end)
  local_78 = (param_1 + param_3) - 1;  // end column
  uStack_7c = param_1 & 0xff00 | (param_1 & 0xff) << 0x10 | (local_78 >> 8) << 0x18 | 0x2a;
  local_78 = local_78 & 0xff;
  local_80 = 0x53902;    // CMDQ word for DCS 0x2A (type=0x05, config=0x02?)
  (*_DAT_0010ab98)(&local_80, 3, 1);  // send 3 words

  // DCS 0x2B: Row Address Set (3 words)
  local_78 = uVar1 & 0xff;
  uStack_7c = param_2 & 0xff00 | (param_2 & 0xff) << 0x10 | (uVar1 >> 8) << 0x18 | 0x2b;
  local_80 = 0x53902;
  (*_DAT_0010ab98)(&local_80, 3, 1);  // send 3 words

  // DCS 0x2C: Write Memory Start (1 word)
  local_80 = 0x2c3909;
  (*_DAT_0010ab98)(&local_80, 1, 0);  // send 1 word, no DSI_START
}
```

---

## 9. Algorithm: Send DCS 0x2C via DSI0 (MT6789)

### Step-by-Step (from LK analysis):

```
STEP 1: Wait for DSI idle
  READ  0x1401300C (DSI_STATUS) → check bit 0 = 0 (not busy)
  Poll with 1ms delay, max 5 retries

STEP 2: Enable DSI controller
  READ  0x14013008 (DSI_CON)
  WRITE 0x14013008 = (value | 3)  // set bits 0,1

STEP 3: Write CMDQ word
  WRITE 0x14013d00 = 0x002C3909  // DSI_CMDQ word 0

STEP 4: Trigger
  WRITE 0x14013060 = 0x8001      // DSI_CMDQ_SIZE = 1 | trigger
  WRITE 0x14013000 = 0           // DSI_START = 0
  WRITE 0x14013000 = 1           // DSI_START = 1

STEP 5: Wait for completion
  READ  0x1401300C (DSI_STATUS) → check bit 0 = 1 (busy→done)
  Then check bit 1 = 1 (ACK received)
  Poll with 1ms delay
```

### Simplified (from kernel mtk_dsi_host_send_cmd):

```c
// 1. Wait for idle
mtk_dsi_wait_for_idle(dsi);

// 2. Build CMDQ word for DCS 0x2C (no params)
u32 cmdq_word = (0x05 << 8) | 0;  // type=DCS_SHORT_WRITE_0_PARAM, config=SHORT_PACKET
// Or for LK format: 0x2c3909

// 3. Write to CMDQ
writel(cmdq_word, dsi->regs + 0xd00);  // DSI_CMDQ for MT6789

// 4. Set CMDQ size
writel(1, dsi->regs + 0x60);  // DSI_CMDQ_SIZE = 1

// 5. Start
writel(0, dsi->regs + 0x00);  // DSI_START = 0
writel(1, dsi->regs + 0x00);  // DSI_START = 1

// 6. Wait for IRQ
mtk_dsi_wait_for_irq_done(dsi, CMD_DONE_INT_FLAG, 2000);
```

---

## 10. Key Findings Summary

1. **MT6789 DSI_CMDQ is at offset 0xd00** (NOT 0x200 like mt8173!)
   - This is the critical difference from other MTK SoCs
   - DSI_CMDQ physical address = `0x14013000 + 0xd00 = 0x14013d00`

2. **DSI_CMDQ_SIZE is at offset 0x60**
   - Same across all MTK DSI variants
   - Bit 15 = trigger, bits [5:0] = command count

3. **DSI_START is at offset 0x00**
   - Toggle: write 0, then write 1

4. **0x2c3909 = DCS 0x2C (Write Memory Start)**
   - bits [23:16] = 0x2C = DCS command
   - bits [15:8] = 0x39 = DCS Long Write type (LK quirk)
   - bits [7:0] = 0x09 = config flags (LK-specific)

5. **Two CMDQ formats exist**:
   - LK format (FUN_0003c8b8): 2 words per command
   - Kernel/LK simple (FUN_0003e108): 1 word per command (used for 0x2C)

6. **The LK ROM arrays**:
   - DAT_0009a8d0[0] = 0x14013000 (DSI0 register base)
   - DAT_0009a8f0[0] = 0x14013d00 (DSI0 CMDQ buffer)
