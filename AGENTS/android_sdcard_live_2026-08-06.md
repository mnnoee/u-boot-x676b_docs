# SD-карта в Android — живые данные с устройства (root/KernelSU)

**Дата:** 2026-08-06 (после полудня)
**Метод:** adb shell + su (KernelSU, контекст `u:r:ksu:s0`)
**Устройство:** Infinix X676B, MT6789. SD-карта вставлена, отформатирована vold'ом.

## ⭐ ГЛАВНАЯ НАХОДКА: SD-карта в Android = mmc0 на контроллере 0x11240000

- `/sys/class/mmc_host/mmc0` → `../../devices/platform/soc/11240000.mmc/mmc_host/mmc0`
- `/sys/class/mmc_host/` содержит ТОЛЬКО `mmc0` (второго слота нет)
- `/dev/block/mmcblk0` = SD-карта (179:0, 31166976 блоков ≈ 15.9 GB), `mmcblk0p1` = 179:1
- vold смонтировал карту: `/dev/block/vold/public:179,1` → `/mnt/media_rw/6662-1609` (FUSE)
- **Это тот же контроллер 0x11240000, который инициализируется в U-Boot!**
- UFS (основной накопитель) — это `sdc` (major 8/259), `sda`/`sdb` = boot0/boot1 UFS (4 MB)

Вывод: железо и контроллер полностью исправны. Проблема только в инициализации U-Boot.

## Рабочее состояние слота (debugfs /sys/kernel/debug/mmc0/ios)

```
clock:        50000000 Hz  (actual 47999984)
vdd:          17 (2.9 ~ 3.0 V)
bus mode:     2 (push-pull)
power mode:   2 (on)
bus width:    2 (4 bits)
timing spec:  7 (sd uhs DDR50)
signal voltage: 1 (1.80 V)
```

## ⭐ Питание: VMCH/VMC включены через PMIC MT6366

```
/sys/class/regulator/regulator.27  NAME=vmch  STATE=enabled  MICROVOLTS=2900000
/sys/class/regulator/regulator.36  NAME=vmc   STATE=enabled  MICROVOLTS=1800000
/sys/class/regulator/regulator.43  mt6358_vmch_eint_high  disabled
/sys/class/regulator/regulator.44  mt6358_vmch_eint_low   disabled
```

**В U-Boot эти регуляторы НЕ включены** — в DTS U-Boot vmmc/vqmmc объявлены
фейковыми `regulator-fixed` с `regulator-always-on` (mt6789.dtsi:381-396), т.е.
реально питание никто не включает. Нужно: VMCH enable 0x1CD8 (bit0), VMC enable 0x1CC4 (bit0),
напряжения: VMCH 2.9V = 0x1E48 [10:8]=2, VMC 1.8V = 0x1E4C [11:8]=4.

## ⭐ Пины MSDC1: GPIO71–76 (НЕ GPIO167, как думали раньше!)

pinmux-pins (функция func1, owner 11240000.mmc):
```
pin 71 (GPIO71): CLK   — input bias pull down, drive 3 mA, IES+SMT, output enabled
pin 72 (GPIO72): CMD   — input bias pull up,   drive 3 mA, IES+SMT, output enabled
pin 73 (GPIO73): DATA0 — input bias pull up,   drive 3 mA, IES+SMT, output enabled
pin 74 (GPIO74): DATA1 — input bias pull up,   drive 3 mA, IES+SMT, output enabled
pin 75 (GPIO75): DATA2 — input bias pull up,   drive 3 mA, IES+SMT, output enabled
pin 76 (GPIO76): DATA3 — input bias pull up,   drive 3 mA, IES+SMT, output enabled
```
pin 70 (GPIO70): pull down — сосед, не в pinmux msdc.
pin 7 (GPIO7):   pull up — CD pin (см. ниже).

### CD-пин
- DT: `cd-gpios = <phandle 0x49 GPIO 7 flags 0>` → GPIO7, активный низкий (pull-up в покое)
- В U-Boot DTS уже есть `cd-gpios = <&pio 7 GPIO_ACTIVE_HIGH>` — но MSDC_PS CDSTS=1
  и при вставленной, и при вынутой карте → hardware CDSTS не реагирует, надо полагаться
  на GPIO CD или принудительно.

## Клоки (clk_summary, debugfs)

```
msdcpll                      383999878 Hz  ← включён, 384 MHz
  msdcpll_ck                 383999878
    msdc50_0_sel             383999878
  msdcpll_d2                 191999939     ← 192 MHz
    msdc30_1_sel             191999939     ← источник MSDC1 = 192 MHz
      msdc30_1_ck            191999939
        ifrao_msdc1_clk      191999939
  msdcpll_d4                  95999969     ← 96 MHz
    msdc5hclk_sel            273000000     ← HCLK (другой источник)
```

- MSDCPLL = 384 MHz, `msdc30_1_sel` = `msdcpll_d2` (192 MHz) — тот же индекс 4,
  что и в U-Boot после загрузки. В U-Boot MSDCPLL тоже включён (CON2=0x80000001).
- Частоты НЕ проблема — U-Boot после ресета уже в правильном состоянии клоков.

## Как получали данные (устройство)

- `/dev/mem` НЕТ (mknod создаёт, но STRICT_DEVMEM → "No such device or address")
- debugfs монтируется: `mount -t debugfs none /sys/kernel/debug`
- `/sys/kernel/debug/mmc0/ios|clock|caps|caps2`
- `/sys/kernel/debug/clk/clk_summary`
- `/sys/kernel/debug/pinctrl/10005000.pinctrl-pinctrl_mtk_v2/{pinmux-pins,pinconf-pins}`
- `/sys/class/regulator/regulator.*/name|state|microvolts`
- `/proc/device-tree/soc/mmc@11240000/...` (od -An -tx1 для бинарных свойств)

## Выводы для U-Boot

1. Контроллер 0x11240000 — правильный, карта определяется там (в Android как mmc0)
2. **Нужно включить питание VMCH 2.9V + VMC 1.8V через pwrap/PMIC MT6366**
   (pwrap: VMCH=0x1CD8, VMC=0x1CC4; pwrap в U-Boot «умирает» — см.
   `pwrap_wacs2_live_debug_2026-08-06.md`, искать решение)
3. Пины GPIO71-76 — в DTS U-Boot конфиг уже правильный (mmc1_pins_default:
   0x4701=GPIO71 CLK, 0x4801=GPIO72 CMD, 0x4901-0x4c01=GPIO73-76 DATA, drive 3,
   pull-up на CMD/DATA, pull-down на CLK). Проверить, что pinctrl реально применяется.
4. Клоки и MSDCPLL уже в порядке (192 MHz) — драйвер форсит 26 MHz зря (workaround
   forced_26mhz); при рабочем питании правильнее оставить 192 MHz.
5. CD: не полагаться на CDSTS, использовать GPIO7 (или игнорировать).

## Файлы
- U-Boot DTS: `u-boot-mtk-g99/arch/arm/dts/mt6789.dtsi:381-425` (регуляторы + sdhc2 + пины)
- U-Boot pinctrl: `drivers/pinctrl/mediatek/pinctrl-mt6789.c` (GPIO71 есть, MTK_PIN(71))
