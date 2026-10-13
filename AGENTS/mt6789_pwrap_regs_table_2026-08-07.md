# MT6789 pwrap: таблица регистров и полная картина инициализации (2026-08-07)

**Источник 1 (сток, без данных)**: `unpack/vip/mtk-pmic-wrap.ko.c` (2183 стр.) — декомпиляция.
**Источник 2 (ПОЛНЫЙ исходник mainline)**: `src_linux/mtk_g99-mainline/drivers/soc/mediatek/mtk-pmic-wrap.c` — содержит `mt6789_regs`, `pwrap_mt6789`, `pwrap_init`, `pwrap_probe`. ИСПОЛЬЗОВАТЬ ЕГО КАК ОСНОВУ!

## mt6789_regs (68-элементная таблица из стокового .ko, .data+0x123c, 272 байта)

Только 8 не-нулевых записей, все остальные индексы = 0x0 (физически НЕ существуют):

| Индекс | Смещение | Абсолютный | Имя (mainline enum) |
|---|---|---|---|
| 0 | 0x0 | 0x10026000 | PWRAP_INIT_DONE2 |
| 34 | 0x880 | 0x10026880 | PWRAP_WACS2_CMD |
| 35 | 0x8A8 | 0x100268A8 | PWRAP_WACS2_RDATA |
| 36 | 0x8A4 | 0x100268A4 | PWRAP_WACS2_VLDCLR |
| 37 | 0x450 | 0x10026450 | PWRAP_INT_EN |
| 46 | 0x3E4 | 0x100263E4 | PWRAP_TIMER_EN |
| 66 | 0x884 | 0x10026884 | PWRAP_SWINF_2_WDATA_31_0 |
| 67 | 0x894 | 0x10026894 | PWRAP_SWINF_2_RDATA_31_0 |

mainline `mt6789_regs` подтверждает те же адреса (mtk-pmic-wrap.c:643):
`INIT_DONE2=0x0, TIMER_EN=0x3E4, INT_EN=0x450, WACS2_CMD=0x880, SWINF_2_WDATA=0x884, SWINF_2_RDATA=0x894, WACS2_VLDCLR=0x8A4, WACS2_RDATA=0x8A8`.

**ВАЖНО**: для MT6789 НЕТ регистров WRAP_EN / WACS2_EN / HIPRIO_ARB_EN / STAUPD / WACS0/1 / INIT_DONE0/1 / SIDLY / DIO. Их индексы в mt6789_regs = 0x0, т.е. ЛЮБАЯ запись по ним попадает в INIT_DONE2 (base+0)!

## pwrap_init для MT6789 (mainline pwrap_init:2067)

- НЕ вызывается, если INIT_DONE2 != 0 (probe: `if (!pwrap_readl(INIT_DONE2)) pwrap_init()`)
- reset_control_reset: только если caps имеет PWRAP_CAP_RESET (бит 1) — у MT6789 caps=0x20 (только ARB бит 5) → **reset НЕ делается** (сток тоже: бит1 caps=0 → не берёт rstc)
- Для MT6789 весь init сводится к: `pwrap_writel(wrp, 1, PWRAP_INIT_DONE2)` (все прочие записи WRAP_EN/ARB/WACS_EN/STAUPD/INIT_DONE0/1 попадают в тот же адрес 0x0)
- `pwrap_common_init_reg_clock` для MT6789 = **no-op** (default: break, стр.1789)
- slave caps SPI → pwrap_reset_spislave + pwrap_init_sidly (mt6366 в mainline НЕТ)

## Клоки pwrap (стоковый LK DT lk_main.dts:2270, 4 клока!)

```
clocks = <&infracfg_ao CLK_IFRAO_PMIC_AP>   // "spi"  = бит 1 IFRAO0
         <&infracfg_ao CLK_IFRAO_PMIC_TMR>  // "wrap" = бит 0 IFRAO0
         <&topckgen 0x18>                   // "ulposc"      = CLK_TOP_PWRAP_ULPOSC (mux!)
         <&topckgen 0x88>                   // "ulposc_osc"  = CLK_TOP_OSC_D10
```

- IFRAO0: status 0x10001090, SET 0x10001080, CLR 0x10001084. **SET=включить, CLR=выключить** (обратно старому документу!). Мы включили: SET 0x10001080 = 0x3 → status = 0x3 ✓
- topckgen mux CLK_TOP_PWRAP_ULPOSC_SEL: MUX_GATE_CLR_SET_UPD(0x90, 0x94, 0x98, shift 8, width 3, gate 15, UPD 0x08 bit 2). Родители: [0]=osc_d10, [1]=tck_26m_mx9_ck, [2]=osc_d4, [3]=osc_d8, [4]=osc_d16. В U-Boot: 0x10000090 биты[10:8]=0 (osc_d10 ✓), gate бит15=0 (открыт ✓). mainline DT assigned-parent = OSC_D10.
- CLK_TOP_OSC_D10 = FACTOR("ulposc", 1, 10). Источник "ulposc" — НЕ определён в mainline топкгене (SPM/внешний osc?). В СТОКЕ это клок из стокового топкгена (#0x18? #0x88?) — не развёрнут.

## Критерий готовности (mainline pwrap_probe)

```
INIT_DONE2 (0x10026000) != 0 → pwrap_init пропускается
WACS2_RDATA (0x100268A8) & BIT(15) == 0 → "initialization isn't finished" → -ENODEV
```
Для PWRAP_CAP_ARB: mask_done = PWRAP_STATE_INIT_DONE1 = BIT(15). Для не-ARB: INIT_DONE0 = BIT(21).

## ⚠️ ОШИБКА в старом файле pwrap_wacs2_live_debug_2026-08-06.md

Там написано «0x8441 → INIT_DONE1=1». **НЕВЕРНО**: 0x8441 & 0x8000 = 0 → бит 15 НЕ установлен.
0x28441 тоже НЕ имеет бита 15 (это 0x8441 | 0x20000).

## Протокол WACS2 (ARB, подтверждено mainline pwrap_read16:1469 / pwrap_write16:1531)

- Чтение: ждать FSM IDLE (биты[3:1]==0 в 0x8A8) → CMD[0x880] = adr → ждать FSM==6 (WFVLDCLR) → данные = RDATA[0x894] & 0xffff → VLDCLR[0x8A4] = 1
- Запись: ждать IDLE → WDATA[0x884] = wdata → CMD[0x880] = BIT(29)|adr (0x20000000)
- Лечение зависшего FSM: если FSM==6 → VLDCLR=1 (pwrap_leave_fsm_vldclr)

## Текущее состояние (2026-08-07, U-Boot v203, warm boot после fastboot)

- Весь блок 0x10026000..0x10026FFF читается как 0x00008441 константой
- Запись mw.l 0x10026000 1 НЕ применяется (чтение после — всё те же 0x8441)
- Клоки IFRAO0 включены, mux топкген верный — блок всё равно мёртв
- Сессия 1 (холодный старт после прошивки): pwrap был РАБОТАЮЩИМ (CMD=0xA → SWCID=0x6610). Умер после экспериментов + warm reboot.
- Гипотеза: прелоадер инициализирует pwrap только при ХОЛОДНОМ старте; при warm boot — НЕ инициализирует. Ядро при загрузке НЕ вызывает pwrap_init т.к. INIT_DONE2 читается как 0x8441 (≠0), но INIT_DONE1 (бит15) = 0 → ядро бы вернуло -ENODEV... ПРОТИВОРЕЧИЕ с работающим Android — НЕ РЕШЕНО. Нужен живой эталон из Android (devmem 0x10026000, 0x100268A8).
