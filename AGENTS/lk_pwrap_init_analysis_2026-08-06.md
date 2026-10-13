# LK и pwrap — итог анализа декомпиляции (2026-08-06)

**Метод:** агент анализировал ТОЛЬКО `/workdir/git/heliog99/lk_unpack_v0/`
(lk_lk.bin.c, lk.img.c, lk.img_full.c, lk_full_disasm.txt, bl2_ext_disasm.txt, old/*).

## Главный вывод: LK НЕ инициализирует pwrap вообще

- **Ноль обращений** к: `0x10026000` (INIT_DONE2), `0x100263E4` (TIMER_EN),
  `0x10026450` (INT_EN), `0x10001080/84/90` (IFRAO0 pmic_tmr/pmic_ap),
  `0x10000090` (topckgen mux pwrap_ulposc_sel), `0x777f` (arb_en_all), `0x6610` (SWCID)
- Единственные адреса блока 0x1002xxxx в LK — WACS2 (0x10026880/A4/A8/94/84) +
  несвязанный clk_buf (0x10022030)
- **pwrap инициализируется прелоадером до LK**; LK только использует готовый WACS2.
  Инициализацию надо брать из ядра `mtk-pmic-wrap.c`, не из LK.

## Единственная pwrap-функция LK — WACS2-транзакция (`lk_lk.bin.c:54027`)

```c
undefined8 FUN_ffff000050f475dc(uint param_1,uint param_2,uint param_3,undefined2 *param_4)
{
  if (1 < param_1) return 2;
  do {} while ((_DAT_ffff0000100268a8 & 0xe) != 0);   // poll WACS2_FSM, биты[3:1]==0 (IDLE)
  if (param_1 == 1) {                                 // WRITE
    _DAT_ffff000010026884 = param_3 & 0xffff;         // WDATA(0x10026884) — СНАЧАЛА данные
    _DAT_ffff000010026880 = param_2 & 0xffff | 0x20000000; // CMD(0x10026880): addr | WACS2_WRITE
    return 0;
  }
  _DAT_ffff000010026880 = param_2 & 0xffff | param_1 << 0x1d; // READ cmd
  if (param_1 != 0) return 0;
  do {} while ((_DAT_ffff0000100268a8 & 0xe) != 0xc); // ждём FSM==0xC (WFVLDCLR)
  *param_4 = (short)_DAT_ffff000010026894;            // RDATA(0x10026894)
  _DAT_ffff0000100268a4 = 1;                          // VLDCLR(0x100268A4)
  return 0;
}
```

Порядок записи: **WDATA(0x884) → CMD(0x880)**. Чтение: CMD → ждать FSM==0xC → RDATA(0x894) → VLDCLR=1.

## Первый WACS2-доступ LK — сразу, без инициализации (gauge, `lk_lk.bin.c:4212-4252`)

```c
puVar4 = FUN_ffff000050f14740();
uVar5 = FUN_ffff000050f475dc(0,0xd88,0,local_14);   // ПЕРВЫЙ доступ — сразу, без инициализации
if ((int)uVar5 != 0) { /* ASSERT */ }
```

Прочие потребители: RMW-обёртка (4256-4281), «battery removed» чтение 0x152/запись 0x4f (4330-4357),
ptim-драйвер (29933-29960).

## IFRAO0 клоки pwrap LK НЕ включает

LK не пишет в 0x10001080/84/90. Ближайшее — 0x10001088/8C (EXT_SET/EXT_CLR) с 0x18000000/0x8000000
(lk_lk.bin.c:80996-81034) — это refcount DMA-клока для SSPM, не pwrap.

## Следствия для нашей задачи

1. В U-Boot после прелоадера pwrap «мёртв» (все регистры 0x8441, записи игнорируются) — но
   прелоадер его инициализировал. Значит при холодном старте он был живой, а «умер» позже
   (возможно при переходе LK→U-Boot или из-за выключенных клоков).
2. Рецепт оживления нужно искать в **стоковом Android-ядре** `mtk-pmic-wrap.ko` — декомпиляция
   в `unpack/modules_decompile/mtk-pmic-wrap.ko_detailed.*` (это НЕ mainline, это реальное
   стоковое ядро устройства).
3. Симптом «все регистры 0x8441» — похоже на выключенный clock/power pwrap-домена. В ядре
   pwrap_init первым делом делает: reset_control_reset, WRAP_EN=1, HIPRIO_ARB_EN=0x777f,
   WACS2_EN=1, WACS0/1/2_EN, STAUPD_PRD=5, STAUPD_GRPEN=0xff, INIT_DONE0/1/2=1.

## Ссылки
- `lk_unpack_v0/LK_PWRAP_INIT_ANALYSIS.md` — полный отчёт агента
- Следующий шаг: анализ `unpack/modules_decompile/mtk-pmic-wrap.ko_detailed.*`
