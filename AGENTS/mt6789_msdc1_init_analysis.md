# MT6789 MSDC1 (SD-карта) — анализ инициализации из LK

**Дата:** 2026-08-06  
**Источник:** `lk_unpack_v0/lk_lk.bin.c` (декомпиляция Ghidra)

## Базовые адреса

- **MSDC1 controller:** `0x11240000`
- **GPIO base (MSDC1):** `0x11ef0000`
- **MSDC1 pin config:** `0x11D40090`, `0x11D400B0`, `0x11D400C0`
- **GPIO pull control base:** `0x10C00C00`
- **PWRAP (PMIC access):** `0x10026000`

## Последовательность инициализации

### 1. Точка входа: `FUN_ffff000050f28344` (строка 28349)

```c
_DAT_ffff000050fedfd0 = 0x708;  // MSDC1 ID
FUN_ffff000050f0e114(&DAT_ffff000050fedfe0);  // ??? (возвращает auVar4)
FUN_ffff000050f214c8(auVar4, ..., &DAT_ffff000050fedf78);  // инициализация контроллера
if (ok) {
    FUN_ffff000050f222f0(&DAT_ffff000050fedf78, 0xffff000050fee0d8);  // инициализация SD-карты
}
```

### 2. Инициализация контроллера: `FUN_ffff000050f214c8` (строка 26585)

**Определение базовых адресов (строки 26607-26612):**
```c
if (*param_9 == 0) {  // MSDC0 (eMMC)
    puVar1 = 0x11230000;
    uVar2 = 0x11f50000;
} else {  // MSDC1 (SD)
    puVar1 = 0x11240000;
    uVar2 = 0x11ef0000;
}
```

**Настройка клоков (строка 26613):**
```c
FUN_ffff000050f43878(&local_48, (uint)*param_9);
```
Читает из таблицы `DAT_ffff000050fa0a5c + param_9 * 0x14`:

| Поле | MSDC0 (eMMC) | MSDC1 (SD) |
|------|--------------|------------|
| `+0x00` (uVar1) | `0x0400000001` | `0x0300000001` |
| `+0x08` (uVar2) | `0x0404040404` | `0x0000030303` |
| `+0x10` (local_38) | `0x00000008` | `0x00000004` |
| `+0x14` (local_40) | `0x00000000` | `0x00000000` |
| `+0x18` | `0x00000080` | `0x00000380` |

**Настройка пинов (строки 26620, 26692-26693):**
```c
FUN_ffff000050f43bd4((char *)param_9);  // базовая настройка пинов
...
FUN_ffff000050f43ae0((char *)param_9, '\0');  // режим 0
FUN_ffff000050f43ae0((char *)param_9, '\x02');  // режим 2
```

`FUN_ffff000050f43ae0` для MSDC1 (строки 51263, 51277-51278):
```c
// Режим 1,2:
_DAT_ffff000011d40090 |= 0x01;  // бит 0
_DAT_ffff000011d400c0 |= 0x3f;  // биты [5:0]
_DAT_ffff000011d400b0 &= ~0x3f; // сброс [5:0]
```

**GPIO pull control (строка 26801-26802):**
```c
uVar5 = (*param_9 == 0) ? 0xa3 : 0xa7;  // MSDC0=0xa3, MSDC1=0xa7
FUN_ffff000050f17b0c(uVar5, 1);  // включить pull
FUN_ffff000050f17b08();
```

`FUN_ffff000050f17b0c` (строки 20044-20051):
```c
lVar4 = (param_1 >> 4) * 4;  // 0xa7 >> 4 = 0xa, offset = 0xa*4 = 0x28
base = 0x10C00C00;
shift = ((param_1 & 0xf) << 1);  // 0xa7 & 0xf = 7, shift = 14
if (param_2 != 0) {  // enable pull
    *(base + lVar4) &= ~(2 << shift);  // сброс бита pull
} else {
    *(base + lVar4) |= (2 << shift);   // установка бита pull
}
```
→ **Регистр: `0x10C00C00 + 0x28 = 0x10C00C28`, бит 14, значение 0 (pull enabled)**

**Инициализация регистров MSDC1 (строки 26679-26809):**
```c
base = 0x11240000;
*(base + 0x38) = 0;           // SDC_CFG: сброс
*(base + 0x34) = 0x1000;      // ???
*(base + 0x70) = (*(base+0x70) & ~0x7000) | 0x2000;  // PATCH_BIT0
*(base + 0x70) |= 2;          // PATCH_BIT0: бит 1
// ожидание сброса boot mode (0x78 бит 4)
while ((*(base + 0x78) >> 4) & 1) { delay(1000); }
*(base + 0x70) &= ~0x8000;    // PATCH_BIT0: сброс бита 15

// Основная настройка контроллера
*base |= 0x0F;  // биты [3:0]: RST | FIFOCLR | DMASTART | ???
// ожидание готовности (бит 2)
while ((*base >> 2) & 1) { ... }

*(base + 0x14) |= 0x80000000;  // бит 31
while ((int)*(base + 0x14) < 0) { ... }  // ожидание сброса

*(base + 0x30) |= 0x80000;     // SDC_CFG: бит 19
*(base + 0x30) &= ~0x100000;   // SDC_CFG: сброс бита 20

// Настройка таймингов
*(base + 0xB0) = 0x403c0007;
*(base + 0xB4) = 0xffe64309;
*(base + 0xB8) = 0x14881803;

// Настройка клока: 100 kHz
FUN_ffff000050f207f0(param_9, 0, 100000);
```

### 3. Инициализация SD-карты: `FUN_ffff000050f222f0` (строка 27229)

```c
// CMD0: GO_IDLE_STATE
FUN_ffff000050f24258((long)param_1);

if (*param_1 != 0) {  // SD, не eMMC
    // CMD8: SEND_IF_COND
    local_68[1] = 0x1aa;  // voltage + check pattern
    local_68[0] = 0x88;   // CMD8
    FUN_ffff000050f1fb5c((long)param_1, local_68);
    
    // ACMD41: SD_SEND_OP_COND
    local_90[0] = 0x129;  // ACMD41 (APP_CMD + SD_SEND_OP_COND)
    local_90[1] = voltage;
    for (retry = 5; retry > 0; retry--) {
        FUN_ffff000050f1fb5c(param_1, local_68);  // CMD55
        FUN_ffff000050f1fb5c(param_1, local_90);  // ACMD41
        if (ok && (response & 0x80000000)) break;
    }
}
```

**Отправка команды: `FUN_ffff000050f1fb5c`** — программирует регистры CMD/ARG контроллера и ожидает ответа.

## Критичные регистры MSDC1

| Смещение | Назначение | Значение |
|----------|-----------|----------|
| `+0x00` | MSDC_CFG | `0x0F` (RST + FIFOCLR + ...) |
| `+0x04` | MSDC_IOCON | clock/data config |
| `+0x0C` | MSDC_INTEN | interrupt enable |
| `+0x14` | MSDC_??? | bit 31 = reset? |
| `+0x30` | SDC_CFG | bits 19,20 |
| `+0x34` | ??? | `0x1000` |
| `+0x38` | SDC_CFG? | `0` (reset) |
| `+0x70` | PATCH_BIT0 | биты 1,13-14 |
| `+0x78` | PATCH_BIT1/STATUS | бит 4 = boot mode |
| `+0xB0` | PAD_TUNE0 | `0x403c0007` |
| `+0xB4` | PAD_TUNE1 | `0xffe64309` |
| `+0xB8` | ??? | `0x14881803` |

## PWRAP/PMIC регистры

**База PWRAP:** `0x10026000`

| Смещение | Назначение | Использование |
|----------|-----------|---------------|
| `+0x880` | WACS2_CMD | запись команды |
| `+0x884` | WACS2_WDATA | данные для записи |
| `+0x894` | WACS2_RDATA | данные чтения |
| `+0x8A4` | WACS2_VLDCLR | очистка флага |
| `+0x8A8` | WACS2_STATUS | статус (биты [3:1]) |

**Функция `FUN_ffff000050f475dc` (строки 54027-54049):**
```c
// param_1: 0=read, 1=write, >1=error
// param_2: адрес регистра PMIC (0x152, 0xd88, ...)
// param_3: значение для записи
// param_4: указатель для чтения

while ((_DAT_ffff0000100268a8 & 0xe) != 0);  // ожидание готовности
if (param_1 == 1) {  // write
    _DAT_ffff000010026884 = param_3 & 0xffff;
    _DAT_ffff000010026880 = (param_2 & 0xffff) | 0x20000000;
} else if (param_1 == 0) {  // read
    _DAT_ffff000010026880 = (param_2 & 0xffff);
    while ((_DAT_ffff0000100268a8 & 0xe) != 0xc);
    *param_4 = (short)_DAT_ffff000010026894;
    _DAT_ffff0000100268a4 = 1;  // clear
}
```

**Известные регистры PMIC:**
- `0x152`: детекция батареи (читается в начале загрузки)
- `0xd88`: неизвестно (читается рано)

**⚠️ Проблема:** Не найдены вызовы `FUN_ffff000050f475dc` для включения LDO (регуляторов напряжения) VMCH/VMC перед инициализацией MSDC1. Возможные причины:
1. Питание включается в BL2 (preloader), до LK
2. Регуляторы управляются аппаратно (DT в BL2)
3. Питание постоянно включено для слота SD

## Следующие шаги

1. **Проверить BL2**: дизассемблировать `bl2_ext_disasm.txt` / `lk_bl2_ext.bin`, найти PWRAP PMIC вызовы
2. **Найти source clock MSDC1**: таблица клоков `0x0300000001` указывает на мультиплексор, нужно найти, какой PLL подключён к MSDC1_HCLK
3. **Сравнить с kernel**: `src_linux/linux/drivers/mmc/host/mtk-sd.c` и `drivers/clk/mediatek/clk-mt6789.c` — актуальные значения для MT6789
4. **Проверить регистры через U-Boot shell**: `md.l 0x11240000 0x10` до/после LK → U-Boot, `md.l 0x10026880 8` (PWRAP status)

## Ссылки на исходный код

- Инициализация MSDC: `lk_lk.bin.c:26585-26810` (`FUN_ffff000050f214c8`)
- Инициализация SD: `lk_lk.bin.c:27229-27421` (`FUN_ffff000050f222f0`)
- Настройка пинов: `lk_lk.bin.c:51247-51280` (`FUN_ffff000050f43ae0`)
- GPIO pull: `lk_lk.bin.c:20036-20054` (`FUN_ffff000050f17b0c`)
- PWRAP read/write: `lk_lk.bin.c:54027-54049` (`FUN_ffff000050f475dc`)
- Таблица клоков: `lk_lk.bin` offset `0xa0a5c` (binary dump)
