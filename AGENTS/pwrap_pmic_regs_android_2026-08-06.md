# PWRAP/PMIC MT6366 — полный кэш регистров из Android (2026-08-06)

**Источник:** `/sys/kernel/debug/regmap/10026000.pwrap/registers` (root/KernelSU, debugfs)
**Что это:** regmap драйвера mt-pmic-pwrap — адреса 0x0000-0xFFFF = **адреса PMIC MT6366**,
значения = фактическое состояние регистров PMIC в рабочей Android-системе.
**Дамп целиком:** `docs/AGENTS/pwrap_pmic_regs_android_dump_2026-08-06.txt` (32768 строк, 16-бит слова)

## ⭐ Значения для SD-карты (проверено в рабочей системе)

| PMIC адрес | Значение | Назначение |
|---|---|---|
| 0x0008 | 0x6610 | SWCID / HW code (ID MT6366) |
| 0x1CC4 | 0x0001 | VMC (vqmmc) enable — бит 0 = 1 |
| 0x1CC6 | 0x0001 | (смежное поле, не разобрано) |
| 0x1CD8 | 0x0001 | VMCH (vmmc) enable — бит 0 = 1 |
| 0x1E48 | 0x0200 | VMCH напряжение: [10:8]=2 → 2.9V |
| 0x1E4C | 0x0400 | VMC напряжение: [11:8]=4 → 1.8V |

## Что нужно записать в PMIC из U-Boot (порядок)

```c
pwrap_write(0x1E48, 0x0200);  // VMCH = 2.9V
pwrap_write(0x1CD8, 0x0001);  // VMCH enable
pwrap_write(0x1E4C, 0x0400);  // VMC = 1.8V
pwrap_write(0x1CC4, 0x0001);  // VMC enable
```

## Про regmap debugfs (как снимать)

- `/dev/mem` в стоковом Android-ядре ОТСУТСТВУЕТ (`CONFIG_DEVMEM=n`) — снять регистры
  напрямую нельзя. Но debugfs regmap доступен:
  - `mount -t debugfs none /sys/kernel/debug`
  - `cat /sys/kernel/debug/regmap/10026000.pwrap/registers` — дамп всего PMIC
  - `cat /sys/kernel/debug/regmap/10026000.pwrap/access` — маска доступа по регистрам
  - `/proc/iomem` показывает `10026000-10026fff : 10026000.pwrap pwrap`
- Другие рабочие источники данных в Android:
  - `/sys/class/regulator/regulator.27` = vmch, `.36` = vmc (state/microvolts)
  - `/sys/kernel/debug/clk/clk_summary` — msdcpll 384 MHz, msdc30_1_sel 192 MHz
  - `/sys/kernel/debug/mmc0/ios` — 50 MHz, 4-bit, DDR50, 1.8V
  - `/sys/kernel/debug/pinctrl/10005000.pinctrl-pinctrl_mtk_v2/pinmux-pins|pinconf-pins`

## Сравнение с U-Boot

В U-Boot pwrap «умирает» при записи CMD (все регистры блока читаются как 0) — см.
`pwrap_wacs2_live_debug_2026-08-06.md`. При этом в Android (после того же U-Boot!) pwrap
работает и PMIC сконфигурирован. Значит «смерть» обратима — ядро оживляет pwrap при старте.

## Ссылки

- `docs/AGENTS/pwrap_wacs2_live_debug_2026-08-06.md` — протокол WACS2 и проблема «чтение убивает»
- `docs/AGENTS/pwrap_mt6366_pmic_access.md` — ранний анализ адресов регуляторов
- Ядро mainline: `src_linux/linux/drivers/soc/mediatek/mtk-pmic-wrap.c` (для справки, не Android)
- LK: `lk_unpack_v0/lk_a_lk.bin.c:53583` (FUN_0004a334)
