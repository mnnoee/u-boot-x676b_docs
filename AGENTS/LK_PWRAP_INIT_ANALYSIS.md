# Анализ: инициализация pwrap в стоковом LK (MT6789, X676B)

Дата: 2026-08-07. Анализ выполнен ТОЛЬКО по файлам `lk_unpack_v0/`.

## ГЛАВНЫЙ ВЫВОД

**LK НЕ инициализирует pwrap вообще.** Ни в lk_lk.bin, ни в bl2_ext, ни в aee (ни в
образе lk.img, ни в стоковом lk_a) НЕТ ни одного обращения к регистрам инициализации:

| Регистр | Назначение (из kernel) | Найдено в LK |
|---|---|---|
| 0x10026000 | PWRAP_INIT_DONE2 | НЕТ (0 ссылок) |
| 0x100263E4 | PWRAP_TIMER_EN | НЕТ (0 ссылок) |
| 0x10026450 | PWRAP_INT_EN | НЕТ (0 ссылок) |
| 0x10001080/84/90 | IFRAO0 клоки pmic_tmr/pmic_ap | НЕТ (0 ссылок) |
| 0x10000090/94/98 | topckgen CLK_CFG_8 (pwrap_ulposc_sel mux) | НЕТ (0 ссылок) |
| 0x777f | arb_en_all | НЕТ |
| 0x6610 | SWCID (MT6366) | НЕТ |

Единственные адреса блока 0x1002xxxx в LK — WACS2 (0x10026880…0x100268A8) плюс
не-связанный блок 0x10022030-0x10022058 (clk_buf/SOI, см. ниже).

Вывод: **pwrap инициализируется прелоадером (BL2/preloader), который запускается ДО LK.
LK просто использует уже готовый WACS2.** В нашем U-Boot pwrap-инициализацию нужно
писать с нуля по kernel-документации, НЕ копируя из LK — в LK её нет.

## Единственная pwrap-функция в LK: доступ через WACS2

Идентична (скопирована) в 3 местах образа и в стоковом lk_a:
- `lk_lk.bin.c:54027` — FUN_ffff000050f475dc (дизасм `lk_full_disasm.txt:71411`, адрес 0x50f475dc)
- `lk.img.c:66769` — FUN_ffff000050f477dc
- `bl2_ext_disasm.txt:0x398c8`, `aee_disasm.txt:0x3b648`
- сток: `old/lk_a_lk_full_disasm_64_v3.txt` fcn.0004a334; bl2_ext: fcn.00039a00

Псевдокод (lk_lk.bin.c:54027-54048):

```c
undefined8 FUN_ffff000050f475dc(uint param_1,uint param_2,uint param_3,undefined2 *param_4)
{
  if (1 < param_1) return 2;              // тип >2 недопустим
  do {} while ((_DAT_ffff0000100268a8 & 0xe) != 0);   // ждём WACS2_FSM(0x100268A8) биты[3:1]==0
  if (param_1 == 1) {                     // WRITE
    _DAT_ffff000010026884 = param_3 & 0xffff;          // WACS2_WDATA(0x10026884)
    _DAT_ffff000010026880 = param_2 & 0xffff | 0x20000000; // WACS2_CMD(0x10026880): addr + WACS2_WRITE(29)
    return 0;
  }
  _DAT_ffff000010026880 = param_2 & 0xffff | param_1 << 0x1d; // READ: cmd = addr | (тип<<29)
  if (param_1 != 0) return 0;             // тип 2 = "записать без ожидания"
  do {} while ((_DAT_ffff0000100268a8 & 0xe) != 0xc); // ждём FSM==0xC (RD_DONE)
  *param_4 = (short)_DAT_ffff000010026894;            // WACS2_RDATA(0x10026894)
  _DAT_ffff0000100268a4 = 1;              // WACS2_VLDCLR(0x100268A4)=1 — сброс признака валидности
  return 0;
}
```

Порядок внутри транзакции:
1. poll 0x100268A8 & 0xE == 0
2. WRITE: сначала WDATA(0x10026884), потом CMD(0x10026880)
3. READ: CMD(0x10026880) = addr; poll 0x100268A8 & 0xE == 0xC; чтение RDATA(0x10026894);
   затем 0x100268A4 = 1 (VLDCLR).

## Вызывающие WACS2 (сигнатура и порядок)

Типы: FUN(write_flag, pmic_reg, value, *result); write_flag=1 → запись, 0 → чтение.

1. **gauge (первый доступ в boot)** — `lk_lk.bin.c:4212-4252` (FUN_ffff000050f02974):
```c
  puVar4 = FUN_ffff000050f14740();
  uVar5 = FUN_ffff000050f475dc(0,0xd88,0,local_14);   // читает PMIC 0xd88 (MT6358/MT6366 AUXADC/battery)
  ...                                              // при ошибке ASSERT (не возвращается)
  DAT_ffff000050fd201c = 0 < local_14[0];          // признак "battery present"
```
   Вызывается из FUN_ffff000050f145d4/f145c8 (инициализация battery-gauge драйвера,
   lk_lk.bin.c:17760-17805), один раз при первом обращении.

2. **RMW-обёртка (read-modify-write)** — `lk_lk.bin.c:4256-4281` (FUN_ffff000050f02af8):
```c
  uVar2 = FUN_ffff000050f475dc(0,param_9,0,local_24);            // read
  if ((int)uVar2 == 0) {
    uVar1 = local_24[0] & ((param_11 & 0xffff) << (param_12 & 0x1f) ^ 0xffffffff) |
            (param_10 & 0xffff) << (param_12 & 0x1f);            // mask/set битов
    local_24[0] = (ushort)uVar1;
    FUN_ffff000050f475dc(1,param_9,uVar1,(undefined2 *)0x0);     // write
  }
```
   (дубликат: FUN_ffff000050f02da0, lk_lk.bin.c:4298-4303)

3. **"battery removed"** — `lk_lk.bin.c:4330-4357` (FUN_ffff000050f02c88):
```c
  uVar2 = FUN_ffff000050f475dc(0,0x152,0,local_4);   // читает PMIC 0x152
  if ((int)uVar2 == 0) {
    pcVar1 = "has_battery_removed=0";
    if (((local_4[0] ^ 0xffff) & 7) != 0) pcVar1 = "has_battery_removed=1";
    FUN_ffff000050f0f9a0(...,pcVar1);                // в env
    FUN_ffff000050f475dc(1,0x152,0x4f,0);            // пишет 0x4f в PMIC 0x152 (сброс флага)
  }
```

4. **ptim-драйвер (polling)** — `lk_lk.bin.c:29933-29960` (FUN_ffff000050f2614c):
```c
  if (param_1 < 0x10) {
    plVar3 = (long *)(&DAT_ffff000050f9ed38 + uVar2 * 0x18);   // таблица {reg,bit,delay,sta_reg}
    FUN_ffff000050f475dc(1,*(ushort*)*plVar3, 1 << (...[1] & 0x1f), 0);  // set bit в PMIC
    FUN_ffff000050f580d4(delay*10);                              // задержка
    FUN_ffff000050f475dc(0,*(ushort*)(*plVar3+4), 0, local_44);  // read status reg
    ... цикл до 0x140 итераций: delay(100), read status
  }
```

## Что LK делает ПЕРЕД первым WACS2-доступом

Ничего специфичного для pwrap: устанавливает MMU/таймеры, затем сразу идёт в
gauge-чтение 0xd88. Ни клоков, ни mux-ов, ни power-доменов для pwrap LK не включает.

## Клоки/power, которые LK реально трогает (для полноты)

- **IFRAO0 0x10001088/0x1000108C** (EXT_SET/EXT_CLR, НЕ pmic-клоки!):
  `lk_lk.bin.c:80996,81005,81034,81671-81709,82080-82118` — refcount DMA-клока:
  `0x1000108C = 0x18000000` (enable, биты 27,28), `0x10001088 = 0x8000000` (disable, бит 27).
  Это SSE/SSPM-канал (0x10210000), не pwrap.
- **0x10022030/34/38/54/58** (`lk_lk.bin.c:42294-42326`, FUN_ffff000050f3656c/365d4) —
  clk_buf/SOI-контроль (включение при 0x10022030|=0x10000003 и т.п.), вызывается из
  FUN_ffff000050f35fe0 (clk_buf driver). Не pwrap.
- Строка "pwrap_pmic" (old/lk_a_lk.bin, оффсет 0x8bdc5) — только log-тег в таблице строк
  (рядом: "dlpt_info", "mediatek,pmic-auxadc"); кода pwrap-драйвера в LK нет.

## Вывод для U-Boot

Инициализацию pwrap (INIT_DONE2, TIMER_EN, INT_EN, arb_en_all, SWCID, клоки IFRAO0,
mux pwrap_ulposc_sel) нужно делать по образцу kernel `drivers/soc/mediatek/mtk-pmic-wrap.c`
и прелоадера, а НЕ по LK — в LK этой логики нет. LK делает лишь:
1. ждать FSM idle (0x100268A8 & 0xE == 0),
2. WDATA затем CMD (запись) / CMD (чтение),
3. ждать FSM==0xC, читать RDATA, VLDCLR=1.
