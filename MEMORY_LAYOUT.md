# Memory Layout

## DRAM

Из `bdinfo`:
```
DRAM bank 0: 0x40000000 - 0x13FFFFFFF (4GB)
DRAM bank 1: 0x100000000 - 0x1FFFFFFFF (4GB)
```

## Reserved Regions (LMB)

```
- BL31:       0x48200000 - 0x483FFFFF
- TEE:        0x70000000 - 0x75FFFFFF
- SCP:        0xBFA00000 - 0xBFDFFFFF
- Framebuffer: 0xFE0C8000 - 0xFFFF6FFF
```

### Два адреса кадра — не путать

| Адрес | Что это |
|-------|---------|
| `0xFE0C8000` | начало **зарезервированной области** кадра (`reserved-memory`, 31 МиБ) |
| `0xFEABE000` | **адрес кадра, который поставил LK** — его читает драйвер из `OVL0_L0_ADDR`; дефолт `LK_DEFAULT_FB` в `mtk_disp_g99.c` |

`0xFEABE000` лежит внутри резерва. Размер кадра: `4320 × 2400 = 0x9E6000` байт.

### Early log

| Адрес | Что |
|-------|-----|
| `0x43FFF000` | 16 МБ буфер раннего лога (`include/early_log.h`), читается через ACM/UART или mtkclient |

## USB PHY Registers

Register bank 0x11f40300+ (НЕ 0x000+!):
- `0x318` — PHY control 1
- `0x31c` — PHY control 2
- `0x320` — PHY control 3
- `0x368` — PHY analog control
- `0x36c` — PHY calibration (eye-vrt, eye-term, discth, rx_sqth)

## Important Addresses

| Address | Description |
|---------|-------------|
| `0x11002000` | UART0 (ttyS0) |
| `0x11200000` | USB MUSB |
| `0x11240000` | MSDC1 (SD-карта) — **не работает, текущий блокер** |
| `0x11f40000` | USB PHY |
| `0x11012000` | Keypad controller (ADC) |
| `0x10010000` | Keypad controller KP (кнопки Volume Up/Down) |
| `0x14000000` | MMSYS (дисплей) |
| `0x14013000` | DSI0 — инициализирует LK |
| `0x43FFF000` | Буфер раннего лога |
| `0xFE0C8000` | Резерв кадра в `reserved-memory` |
| `0xFEABE000` | Кадр, установленный LK |

## UFS LUNs

- sda (4MB boot)
- sdb (4MB boot)
- sdc (238GB main)

## Kernel cmdline (from stock LK)

```
console=tty0 root=/dev/ram loglevel=8 8250.nr_uarts=4 initcall_debug=1 mtk_printk_ctrl.disable_uart=1 bootopt=64S3,32N2,64N2 PN=x676b_h891 arm64.nomte tkv=294930432 bootconfig
```

- `mtk_printk_ctrl.disable_uart=1` — UART disabled in kernel by default, must set to `0` for debug
- `bootopt=64S3,32N2,64N2` — boot partition sizes
- `PN=x676b_h891` — product name
- `arm64.nomte` — no MTE (memory tagging)
- `tkv=294930432` — TKV version
- `bootconfig` — uses Android Boot Configuration (bootconfig.txt in boot image)

## Boot Flow

```
Preloader → U-Boot (in lk_a/lk_b) → boot.img kernel → Android
```

- Preloader is proprietary (cannot be replaced)
- U-Boot replaces LK in lk_a/lk_b partitions
- `gimme-uboot.py` from `liblk` packages U-Boot binary into LK image format
- Flash via `fastboot flash lk_a/lk_b`
