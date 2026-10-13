# PWRAP WACS2 — живая отладка 2026-08-06: «чтение убивает канал»

**Статус:** НЕ РЕШЕНО. Запись CMD в WACS2_CMD на живом устройстве после reboot'а
приводит к «смерти» pwrap (все регистры блока читаются как 0x00000000).
В первой сессии (до первого reboot'а) CMD=0xA сработал (SWCID=0x6610). См. конец файла.

## Контекст

Цель: включить VMCH/VMC (питание SD) через pwrap на MT6789 (X676B). PMIC MT6366.
Доступ: U-Boot shell по /dev/ttyACM0, `uart_cmd.py -b 921600 -t 1.5 "cmd"`.
Команды слать ПО ОДНОЙ с паузой (пакетная отправка теряет/перемешивает вывод).

## Регистры WACS2 (MT6789, база 0x10026000) — ВЕРИФИЦИРОВАНО ядром и LK

| Регистр | Смещение | Абсолютный |
|---|---|---|
| PWRAP_INIT_DONE2 | 0x0 | 0x10026000 |
| PWRAP_TIMER_EN | 0x3E4 | 0x100263E4 |
| PWRAP_INT_EN | 0x450 | 0x10026450 |
| PWRAP_WACS2_CMD | 0x880 | 0x10026880 |
| PWRAP_SWINF_2_WDATA_31_0 | 0x884 | 0x10026884 |
| PWRAP_SWINF_2_RDATA_31_0 | 0x894 | 0x10026894 |
| PWRAP_WACS2_VLDCLR | 0x8A4 | 0x100268A4 |
| PWRAP_WACS2_RDATA | 0x8A8 | 0x100268A8 |

Источники:
- ядро: `src_linux/linux/drivers/soc/mediatek/mtk-pmic-wrap.c:643` (mt6789_regs)
- LK: `lk_unpack_v0/lk_a_lk.bin.c:53583` (FUN_0004a334) — идентичный набор адресов

### FSM (для ARB, PWRAP_GET_WACS_ARB_FSM)

```
PWRAP_GET_WACS_ARB_FSM(x) = ((x) >> 1) & 7     // биты [3:1] регистра 0x100268A8
```

| Значение | Состояние |
|---|---|
| 0 (0x0) | IDLE |
| 2 (0x4) | REQ |
| 4 (0x8) | WFDLE |
| 6 (0xC) | WFVLDCLR — данные готовы |

LK ждёт: `while ((0x100268A8 & 0xe) != 0)` для IDLE и `!= 0xc` для готовности.
INIT_DONE1 = BIT(15) в 0x100268A8 (0x8441 → INIT_DONE1=1, FSM=0 → IDLE).

ВАЖНО: биты [3:1] — это БИТЫ [3:1], НЕ [18:16] (последние — для НЕ-ARB платформ,
PWRAP_GET_WACS_FSM). Не перепутать!

## Протоколы (из ядра, pwrap_read16/pwrap_write16 — строки 1459-1552)

### Чтение (read16):
```
1. ждать FSM == IDLE (0); при таймауте: если FSM==6, записать 1 в VLDCLR (восстановление)
2. CMD[0x10026880] = adr              // для ARB адрес КАК ЕСТЬ (без >>1)
3. ждать FSM == WFVLDCLR (6)
4. данные = readl(0x10026894) & 0xffff   // SWINF_2_RDATA
5. VLDCLR[0x100268A4] = 1              // ОБЯЗАТЕЛЬНО после чтения!
```

### Запись (write16):
```
1. ждать FSM == IDLE
2. WDATA[0x10026884] = wdata & 0xffff
3. CMD[0x10026880] = 0x20000000 | adr    // BIT(29) — признак записи
```
(завершения не ждут, VLDCLR после записи НЕ пишут)

LK (FUN_0004a334) — точно так же: чтение CMD=адрес, данные из 0x894, VLDCLR=1;
запись WDATA + CMD|0x20000000.

## Комментарий ядра про таймаут (строки 1432-1439)

Если после чтения не записать VLDCLR — FSM застревает в WFVLDCLR(6), и СЛЕДУЮЩАЯ
команда не принимается (таймаут). Лечение: записать 1 в VLDCLR (pwrap_leave_fsm_vldclr).

## ⚠️ НАБЛЮДЕНИЕ: чтение «убивает» pwrap

### Сессия 1 (ДО reboot) — pwrap РАБОТАЛ:
- 0x100268A8 = 0x00028441 (INIT_DONE1=1, FSM биты [3:1]=0 → IDLE)
- CMD=0xA → статус 0x0002844D (FSM=6 WFVLDCLR ✓), данные 0x10026894 = 0x6610 (SWCID MT6366 ✓)
- Затем CMD=0x1CD8 (БЕЗ закрытия VLDCLR после 0xA!) → канал завис: все регистры 0x6610

### Сессия 2+ (после U-Boot reset) — pwrap читается, но СРАЗУ «умирает» от CMD:
- После reboot: 0x100268A8 = 0x00008441 (INIT_DONE1=1, FSM=0 IDLE) — выглядит живым
- Запись CMD=0x1CD8 → ВСЕ регистры pwrap-блока (0x10026000, 0x100263E4, 0x10026450,
  0x10026880, 0x10026894, 0x100268A8) читаются как 0x00000000
- reboot, запись CMD=0xA → та же «смерть» (все 0)
- reboot, VLDCLR=1 → CMD=0xA → та же «смерть»
- Только чтение без записи CMD — pwrap остаётся живым (0x8441)

Проверено, что дело НЕ в uart_cmd.py: mw.l/readback на topckgen (0x10000080)
работают идеально (запись 0x01000401 → чтение 0x01000401).

### Симптомы «смерти»:
- 0x100268A8 = 0x00000000 (вместо 0x8441) — INIT_DONE1 пропал
- 0x10026000 (INIT_DONE2) = 0 — не инициализирован
- Все остальные регистры блока тоже 0
- U-Boot при этом ЖИВ (topckgen/MSDC1 читаются нормально)

### Гипотезы (не подтверждены):
1. Запись CMD при не-инициализированном pwrap вызывает защитное отключение блока
   (INIT_DONE2=0 говорит, что preloader НЕ доделал инициализацию после warm reboot?)
2. U-Boot reset (команда `reset`) — «тёплый» сброс, после которого pwrap не полностью
   инициализирован preloader'ом, в отличие от холодного старта (когда работал CMD=0xA)
3. Нужна ручная инициализация pwrap перед использованием (WRAP_EN=1, WACS2_EN=1,
   INIT_DONE2=1 и т.п. — как pwrap_init в ядре, строки 2050-2152)

### Следующие шаги:
- Попробовать ПРИНУДИТЕЛЬНУЮ инициализацию pwrap перед CMD (написать 1 в INIT_DONE2
  в 0x10026000, проверить WACS2_EN/WRAP_EN — но для MT6789 их нет в mt6789_regs!)
- Сравнить с mtk-pmic-wrap.c pwrap_init: для MT6789 (тип PWRAP_MT6789, стр. 2319):
  arb_en_all=0x777f, int_en_all=0x180000, spi_w=PWRAP_MAN_CMD_SPI_WRITE,
  caps=PWRAP_CAP_ARB. В regs НЕТ WRAP_EN/WACS2_EN/HIPRIO_ARB_EN — значит для MT6789
  они не нужны / всегда включены?
- Холодная перезагрузка устройства (через питание), проверить чтение 0x1CD8 —
  если заработает, значит дело в warm reboot
- Проверить, что делает preloader с pwrap: искать обращения к 0x10026000 в preloader
- Альтернатива: включить VMCH/VMC через SPM/mt6358 driver в самом U-Boot
  (drivers/power/pmic/ в u-boot-mtk-g99), тогда инициализация pwrap будет правильной

## Адреса регуляторов SD (MT6366, для последующего включения)

| LDO | Enable (бит 0) | Напряжение (ANA_CON0) |
|---|---|---|
| VMCH (vmmc) | 0x1CD8 | 0x1E48: [10:8]=2→2.9V, 3→3.0V, 5→3.3V |
| VMC (vqmmc) | 0x1CC4 | 0x1E4C: [11:8]=4→1.8V, 0xA→2.9V, 0xB→3.0V, 0xD→3.3V |

Источник: src_linux/linux/drivers/regulator/mt6358-regulator.c (mt6366_regulators, стр. 616-623).
Для обычной SD: VMCH=3.3V (0x0500 в 0x1E48), VMC=3.3V (0x0D00 в 0x1E4C).

## Полезные ссылки
- Ядро: `src_linux/linux/drivers/soc/mediatek/mtk-pmic-wrap.c`
  (mt6789_regs:643, pwrap_get_fsm_state:1411, pwrap_read16:1459, pwrap_write16:1531,
  pwrap_init:2050, pwrap_probe:2563 «if !INIT_DONE2 → pwrap_init»)
- LK: `lk_unpack_v0/lk_a_lk.bin.c:53583` (FUN_0004a334)
- LK использует адреса: 0xD88 (ADC), 0x152 (флаг батареи) — мелкие адреса работают?
- `docs/AGENTS/pwrap_mt6366_pmic_access.md` — ранняя версия анализа
