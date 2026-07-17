# MT6789 DISP Power Domain Status — Device Readings

## Initial Readings (from device via serial)

```
0x10006354 = 0x0000000d   (DISP_PWR_CON)
0x1000616c = 0xf83a1bc1   (PWR_STATUS)
0x10006170 = 0xf83a1bc1   (PWR_STATUS_2ND)
```

## Bit 21 Analysis

```
0xf83a1bc1 >> 21 & 1 = 1   ← Bit 21 IS SET in both registers
```

**Conclusion: DISP power domain is ON.**

## DISP_PWR_CON (0x10006354) = 0x0000000d

Binary: `0000 0000 0000 0000 0000 0000 0000 1101`

| Bit | Name | Value | Meaning |
|-----|------|-------|---------|
| 0 | PWR_RST_B | 1 | Reset released (active) |
| 1 | PWR_ISO | 0 | No isolation (good) |
| 2 | PWR_ON | 1 | Power on (good) |
| 3 | PWR_ON_2ND | 1 | Secondary power on (good) |
| 4 | PWR_CLK_DIS | 0 | Clocks enabled (good) |

**All power bits are correctly configured.** Domain is fully powered and out of reset.

## Implications

The DISP domain is powered on, so the inability to write MUTEX_EN/OVL_EN is NOT caused by power domain gating. The root cause must be something else — possibly:

1. MMSYS clock gating (CLK_MM_DISP_MUTEX0 = bit 0 of 0x14000100)
2. Bus protection still active
3. Some other module-level reset

## Other Readings

```
0x100063bc = 0x00000001   (DPM control — bit 4 = 0, DPM not started)
0x14000100 = 0xfe848040   (MMSYS clock gate status — MUTEX0 bit 0 = OFF)
```
