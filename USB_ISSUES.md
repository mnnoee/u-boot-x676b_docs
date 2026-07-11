# USB Проблемы

## ✅ USB работает (обновлено 2026-07-11)

### Текущий статус
USB stack работает! PC видит устройство через CDC ACM:

```
Bus 003 Device 126: ID 0000:0000 U-Boot USB download gadget
```

**Подключение:**
- Порт: `/dev/ttyACM1`
- Baudrate: 921600
- Driver: `cdc_acm`
- USB path: `pci-0000:2f:00.3-usbv2-0:3.3.4:1.0`

**Подтверждено:**
- ✅ `serial_list_ports` находит устройство
- ✅ `serial_open_port` открывает соединение
- ✅ `serial_write_data` отправляет команды
- ✅ `serial_read_for_duration` читает ответы
- ✅ U-Boot отвечает на команды (`help`, `bdinfo`, `printenv`)

**Пример подключения:**
```bash
# Через picocom
picocom -b 921600 /dev/ttyACM1

# Через opencode serial инструменты
serial_open_port port=/dev/ttyACM1 baud_rate=921600
serial_write_data port_id=<id> data="help\n"
serial_read_for_duration port_id=<id> duration=2
```

### Тестирование кнопок (2026-07-11)

Кнопки работают! Пример вывода `button vup`:
```
button vup
KP: probe start
KP: addr=0x10010000
KP: ioremap...
KP: base=0000000010010000
KP: reading props...
KP: code=115 reg=0xc val=0xdfef
off
```

**Разбор:**
- `code=115` — KEY_VOLUMEUP (Volume Up)
- `reg=0xc` — регистр KP_MEM3
- `val=0xdfef` — значение регистра (кнопка нажата)

**Исправлено (v88):**
- Исправлена логика `button_mtk_kp_get_state()` — была неправильная проверка `(val & matrix_val) != matrix_val`
- Теперь проверяется `val == matrix_val` (active-low: биты LOW когда кнопка нажата)

**Интеграция в bootmenu (v89):**
- Модифицированы `tstc()` и `getchar()` в `common/console.c`
- Добавлен FIFO буфер для символов, генерируемых кнопками
- `button_input_poll()` опрашивает keypad и добавляет символы:
  - Volume Up → Enter (`\r`)
  - Volume Down → Стрелка вниз (`ESC [ B`)
- Добавлено зацикливание навигации в `cmd/bootmenu.c`:
  - BKEY_DOWN: последний пункт → первый
  - BKEY_UP: первый → последний

**Доступные кнопки:**
- `button vup` — Volume Up (code=115, reg=0xc, val=0xdfef)
- `button vdown` — Volume Down (code=114, reg=0xc, val=0xbfdf)

---

## Историческая проблема: pullup toggle (РЕШЕНА)

### Симптомы
- `fastboot 0 usb 0` запускается
- PHY probe успешен (`MUSB: PHY probed OK`)
- pullup включается и сразу выключается:
  ```
  musb_pullup() gadget D+ pullup on
  musb_pullup() gadget D+ pullup off  ← проблема!
  ```

### Анализ (из uart-log-v55.txt)

#### Первый вызов fastboot:
```
=> fastboot 0 usb 0
MUSB: PHY probed OK
MTK MUSB OTG (Peripheral)
musb_start() <== devctl 98
musb_pullup() gadget D+ pullup on              ← pullup включён
musb_interrupt() ** IRQ peripheral usb0008 tx0000 rx0000    ← USB reset от PC!
musb_stage0_irq() <== Power=e0, DevCtl=98, int_usb=0x8     ← int_usb=0x8 = MUSB_INTR_RESET
musb_pullup() gadget D+ pullup off             ← pullup выключен в musb_g_disconnect
=> fastboot 0 usb 0                            ← возврат в shell, БЕЗ второго pullup on!
```

#### int_usb=0x8 означает:
```c
#define MUSB_INTR_RESET     0x08  // USB bus reset
```

PC отправляет USB reset → MUSB получает interrupt → `musb_stage0_irq()` вызывает `musb_g_reset()` → который вызывает `musb_g_disconnect()` → pullup выключается.

### Fix в v55 (есть в коде)

В конце `musb_g_reset()` добавлен блок:

```c
#ifdef __UBOOT__
    /* Re-enable pullup after reset if it was on before */
    if (musb->softconnect)
        musb_pullup(musb, 1);
#endif
```

### Почему fix НЕ работает?

Из логов видно, что **второго `pullup on` НЕТ** после reset. Возможные причины:

#### 1. `musb->softconnect` не установлен
Проверить где `musb->softconnect` устанавливается в 1:

```c
// В musb_gadget.c:musb_pullup()
musb->softconnect = is_on;
```

Если `musb_pullup(musb, 1)` был вызван **до** reset, то `musb->softconnect` должен быть 1.

Из логов:
```
musb_pullup() gadget D+ pullup on   ← musb->softconnect = 1
IRQ reset
musb_g_disconnect()                 ← pullup off, но softconnect НЕ меняется!
musb_g_reset()
  if (musb->softconnect)            ← должен быть TRUE!
    musb_pullup(musb, 1)            ← НО НЕТ В ЛОГАХ!
```

#### 2. Возможная причина: race condition или проблема с lock
`musb_g_reset()` объявлен как:
```c
void musb_g_reset(struct musb *musb)
__releases(musb->lock)
__acquires(musb->lock)
```

Может быть проблема с `musb->lock` — lock release/acquire происходит неправильно и код после него не выполняется?

#### 3. Другая гипотеза: __UBOOT__ не определён при компиляции
Проверить `.config` или `defconfig` — возможно `__UBOOT__` не определён!

### Следующие шаги

#### 1. Добавить debug print перед fix:
```c
#ifdef __UBOOT__
    printf("DEBUG: musb->softconnect = %d\n", musb->softconnect);
    /* Re-enable pullup after reset if it was on before */
    if (musb->softconnect) {
        printf("DEBUG: Re-enabling pullup after reset\n");
        musb_pullup(musb, 1);
    } else {
        printf("DEBUG: softconnect is 0, NOT re-enabling pullup\n");
    }
#endif
```

#### 2. Проверить где вызывается musb_g_reset:
```bash
grep -rn "musb_g_reset" u-boot-mtk-g99/drivers/usb/musb-new/
```

#### 3. Проверить musb_g_disconnect:
Возможно `musb_g_disconnect()` сбрасывает `musb->softconnect` в 0?

```c
// musb_gadget.c:musb_g_disconnect()
static void musb_g_disconnect(struct musb *musb)
{
    // ...
    // Проверить есть ли здесь:
    // musb->softconnect = 0;  ← ЭТО ПРОБЛЕМА!
}
```

#### 4. Альтернативное решение: не вызывать musb_g_disconnect при reset
```c
void musb_g_reset(struct musb *musb)
{
    // ...
    
    /* report disconnect, if we didn't already (flushing EP state) */
    if (musb->g.speed != USB_SPEED_UNKNOWN)
        musb_g_disconnect(musb);  ← МОЖЕТ НЕ НУЖНО ВЫЗЫВАТЬ?
    
    // ...
}
```

## USB PHY Registers

Register bank 0x11f40300+ (НЕ 0x000+!):
- `0x318` — PHY control 1
- `0x31c` — PHY control 2
- `0x320` — PHY control 3
- `0x368` — PHY analog control
- `0x36c` — PHY calibration (eye-vrt, eye-term, discth, rx_sqth)

Stock LK functions:
- `FUN_0004a1f0` — poweron
- `FUN_0004a240` — savecurrent
- `FUN_0004a2bc` — recover + calibration
