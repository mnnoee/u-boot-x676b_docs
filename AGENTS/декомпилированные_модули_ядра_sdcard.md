Now I have the complete picture. Here is the full report of all findings.
---
Полные результаты поиска MSDC/MMC инициализации в модулях ядра
Файлы с совпадениями (абсолютные пути):
1. /workdir/git/heliog99/unpack/modules_decompile/mtk-mmc.ko_detailed.c — основной MSDC драйвер
2. /workdir/git/heliog99/unpack/modules_decompile/mtk-scpsys.ko_detailed.c — общий SCPSYS (power domain)
3. /workdir/git/heliog99/unpack/modules_decompile/mtk-scpsys-mt6789.ko_detailed.c — SCPSYS-драйвер специфичный для MT6789
4. /workdir/git/heliog99/unpack/modules_decompile/rpmb-mtk.ko_detailed.c — RPMB (eMMC Security)
5. /workdir/git/heliog99/unpack/modules_decompile/mtk-mmc-autok.ko_detailed.c — AUTO-K (автоматический tuning)
6. /workdir/git/heliog99/unpack/modules_decompile/mt6789_dcm.ko_detailed.c — DCM infracfg
7. /workdir/git/heliog99/unpack/modules_decompile/pinctrl-mt6789.ko_detailed.c — пинмаппинг MT6789
Совпадения по паттернам:
Паттерн	Результат
0x11240000	НЕ НАЙДЕН — нет явного кода адреса
0x14012000	НЕ НАЙДЕН — нет явного кода адреса
clk_enable.*msdc	НЕ НАЙДЕН как строка — нет string-form обращений
power_domain	Найден в mtk-scpsys.ko (28 совпадений), mtk-scpsys-mt6789.ko, clk-common.ko
scpsys	Найден в mtk-scpsys.ko (132 совпадения), mtk-scpsys-mt6789.ko
bus_prot	Найден в mtk-scpsys.ko (10 совпадений)
infracfg	Найден в mtk-scpsys.ko (5 совпадений), mtk_iommu.ko, mt6789_dcm.ko, mediatek-drm.ko
0x1000108C	НЕ НАЙДЕН
0x10001088	НЕ НАЙДЕН
0x10001094	НЕ НАЙДЕН
---
ПОЛНАЯ ПОСЛЕДОВАТЕЛЬНОСТЬ MSDC/MMC ИНИЦИАЛИЗАЦИИ ИЗ ЯДРА
1. ПРОБИРКА (probe): msdc_drv_probe() — mtk-mmc.ko (строки 751-1168)
Функция: msdc_drv_probe(struct platform_device *pdev)
Файл: mtk-mmc.ko_detailed.c
Последовательность:
a) mmc_alloc_host(0x570, dev)                           // строка 782
b) mmc_of_parse()                                       // строка 787
c) devm_platform_ioremap_resource(pdev, 0)               // строка 790 — MSDC base ioremap
d) platform_get_resource(pdev, IORESOURCE_MEM, 1)        // строка 793 — CQHCI base (optional)
e) mmc_regulator_get_supply(host)                        // строка 802
f) clk_get(dev, "source")                                // строка 805 — MSPLL clock
g) clk_get(dev, "sys_cg")                                // строка 808 — system clock gate
h) clk_get_optional(dev, "bus_clk")                      // строка 811
i) clk_get_optional(dev, "crypto_clk")                   // строка 817
j) clk_get_optional(dev, "source_cg")                    // строка 823
k) clk_get_optional(dev, "sys_cg")                       // строка 829
l) clk_get_optional(dev, "crypto_cg")                    // строка 835
--- НАЧАЛО ВКЛЮЧЕНИЯ ЧАСОВ (line 842-861) ---
m) clk_prepare(sys_cg)    → если ok → clk_enable(sys_cg)  // строка 842-853
n) devm_clk_bulk_get_optional(dev, 3, {"pclk_cg","axi_cg","ahb_cg"}) // строка 857
o) __devm_reset_control_get(dev, "msdc", ...)            // строка 868 — reset control
p) platform_get_irq(pdev, 0)                             // строка 876 — IRQ
--- ПИНМУКС (lines 882-896) ---
q) devm_pinctrl_get(dev)                                 // строка 882
r) pinctrl_lookup_state(pinctrl, "default")              // строка 885
s) pinctrl_lookup_state(pinctrl, "state_uhs")            // строка 891
--- ЧТЕНИЕ DT СВОЙСТВ (lines 898-998) ---
t) of_property_read "mediatek,latch-ck"                  // строка 899
u) of_property_read "hs400-ds-delay"                     // строка 900
v) of_property_read "mediatek,hs200-cmd-int-delay"       // строка 902
w) of_property_read "mediatek,hs400-cmd-int-delay"       // строка 904
x) of_property_read "host-index"                         // строка 906
y) devm_regulator_get_optional(dev, "dvfsrc-vcore")      // строка 907
z) of_property_read "req-vcore"                          // строка 914
aa) of_property_read "ocr-voltage"                       // строка 923
--- УСТАНОВКА CALLBACKS ---
ab) msdc_ungate_clock(plVar2)                            // строка 1005 — ВКЛЮЧЕНИЕ ВСЕХ ЧАСОВ
ac) msdc_init_hw(plVar2)                                // строка 1081 — АППАРАТНАЯ ИНИЦИАЛИЗАЦИЯ
ad) devm_request_threaded_irq(msdc_irq)                  // строка 1082
--- CQHCI (CMD Queue) ---
ae) cqhci_init()                                         // строка 1016 (optional)
--- ФИНАЛ ---
af) mmc_add_host(host)                                   // строка 1111
ag) mmc_rpmb_register(host)                              // строка 1114
2. ВКЛЮЧЕНИЕ ЧАСОВ: msdc_ungate_clock() — строки 1391-1445
Файл: mtk-mmc.ko_detailed.c:1391
Полная последовательность включения 6 индивидуальных часов + bulk:
// Строка 1401-1405: Clock 1 (offset 0x2b = sys_cg)
clk_prepare(param_1[0x2b]);  // sys_cg
clk_enable(param_1[0x2b]);
// Строка 1406-1410: Clock 2 (offset 0x29 = bus_clk)
clk_prepare(param_1[0x29]);  // bus_clk
clk_enable(param_1[0x29]);
// Строка 1411-1415: Clock 3 (offset 0x2a = crypto_clk)
clk_prepare(param_1[0x2a]);  // crypto_clk
clk_enable(param_1[0x2a]);
// Строка 1416-1420: Clock 4 (offset 0x28 = source)
clk_prepare(param_1[0x28]);  // source (main clock)
clk_enable(param_1[0x28]);
// Строка 1421-1425: Clock 5 (offset 0x2c = source_cg)
clk_prepare(param_1[0x2c]);  // source_cg
clk_enable(param_1[0x2c]);
// Строка 1426-1430: Clock 6 (offset 0x2d = crypto_cg)
clk_prepare(param_1[0x2d]);  // crypto_cg
clk_enable(param_1[0x2d]);
// Строка 1431-1441: Bulk clocks: pclk_cg, axi_cg, ahb_cg (3 штуки)
clk_bulk_prepare(3, param_1 + 0x2f);  // pclk_cg, axi_cg, ahb_cg
clk_bulk_enable(3, param_1 + 0x2f);
// Строка 1436-1438: Ожидание готовности MSDC
while ((*(uint*)(param_1[8] + 4) >> 7 & 1) == 0) {
    Yield();  // Ждём bit 7 = DLL ready
}
3. АППАРАТНАЯ ИНИЦИАЛИЗАЦИЯ: msdc_init_hw() — строки 1568-1682
Файл: mtk-mmc.ko_detailed.c:1568
// Строки 1578-1581: Reset sequence
if (param_1[0x27] != 0) {         // reset_control
    reset_control_assert();        // ASSERT reset
    usleep_range(10, 50);          // Wait 10-50 us
    reset_control_deassert();      // DEASSERT reset
}
// Строка 1583-1586: MSDC Global control - SW_RST (bit 0)
*(uint*)(base + 0) = *(uint*)(base + 0) | 1;   // MSDC_CFG |= SW_RST
msdc_reset_hw(param_1);
// Строка 1588: Clear all interrupts
*(uint*)(base + 0x10) = 0;                      // MSDC_INTEN = 0
// Строка 1589-1591: Acknowledge all pending IRQs
*(uint*)(base + 0x0c) = *(uint*)(base + 0x0c);  // MSDC_INT = read to clear
// Строки 1592-1618: FIFO and DMA configuration (depends on v2)
// Если v2 (line 1604-1618):
//   base[0x08] &= 0xffff0fff; base[0x08] |= 0x8000;  // CMD_RESP_SEL
//   base[0x08] |= 1;                                  // SDIO_INT_CMD
//   base[0x10] |= 2;                                  // Enable INT_CMD_DONE
//   base[0x30] |= 2;                                  // Enable INT_CMD_DONE status
// Строки 1619-1626: Pad tuning
// base[0xb8] &= 0xfff8ffff; base[0xb8] |= 0x20000;   // PAD_TUNE
// base[0xb8] &= 0x1fffffff; base[0xb8] |= 0x40000000;
// Строка 1640-1648: DAT_RD_DLY (if supported)
// base[0xb8] |= 2;
// Строка 1655-1663: SDIO_CFG (if needed)
// base[0x68] |= 1;
// Строка 1665-1676: MSDC_INTEN and MSDC_PATCH
// base[0x30] |= 0x80000;                             // DATA_IRQ_EN
// base[0x30] &= 0xffefffff;                          // Clear bit 20
// base[0x64] |= 0x80000;                             // CMD_RESP_SEL
// base[0x30] |= 0x3000000;                           // Bits 24-25
// Строка 1677: Tune settings init
msdc_init_tune_setting(param_1);
// Строка 1680:
_dev_info(*param_1, "init hardware done!");
4. MSDC RESET: msdc_reset_hw() — строки 2041-2067
Файл: mtk-mmc.ko_detailed.c:2041
// Строка 2051: Trigger SW_RST
**base = **base | 4;              // MSDC_CFG |= SW_RST (bit 2)
// Строка 2052-2054: Wait for reset complete
while ((**base >> 2 & 1) != 0) {  // Wait until SW_RST clears
    Yield();
}
// Строка 2055-2058: Clear FIFO reset
base[0x14] = base[0x14] | 0x80000000;  // MSDC_FIFOCS |= CLR_FIFO (bit 31)
// Строка 2059-2061: Wait FIFO clear
while (*(int*)(base + 0x14) < 0) {     // Wait until bit 31 clears
    Yield();
}
// Строка 2064-2065: Acknowledge interrupts
*(uint*)(base + 0x0c) = *(uint*)(base + 0x0c);  // MSDC_INT read-to-clear
5. POWER DOMAIN (SCPSYS) ДЛЯ MSDC
scpsys_power_on() — mtk-scpsys.ko, строка 2018 — Последовательность power-on домена:
a) regulator_enable()                                    // строка 2052 (optional)
b) clk_prepare(base_clk[0]) → clk_enable(base_clk[0])  // строка 2055-2058
c) clk_prepare(base_clk[1]) → clk_enable(base_clk[1])  // строка 2062-2064
d) clk_prepare(base_clk[2]) → clk_enable(base_clk[2])  // строка 2067-2073
e) Включение MTCMOS (строки 2105-2111):
   puVar2 = (uint*)(spm_base + domain_offset);
   *puVar2 = uVar7 | 4;        // PWR_ON (bit 2)
   *puVar2 = uVar7 | 0xc;      // PWR_ON | PWR_ISO解除
f) Ожидание power-on с таймаутом 1 сек (строки 2114-2142):
   while (scpsys_domain_is_on() == 0) {
       if (timeout > 1s) break;
       __const_udelay();
   }
g) __const_udelay(0x68dbc);      // строка 2143 — задержка после power-on
h) Снятие изоляции (строки 2144-2149):
   *puVar2 = uVar7 & 0xffffffef | 0xc;   // PWR_RST_B=1, PWR_ISO=0
   *puVar2 = uVar7 & 0xffffffed | 0xc;   // PWR_RST_B=1
   *puVar2 = uVar7 & 0xffffffed | 0xd;   // PWR_CLK_DIS=0
i) Включение subsystem clocks (строки 2152-2287):
   до 10 subsystem clocks sequentially
j) scpsys_sram_enable()                                // строка 2440
k) clear_bus_protection() для каждого домена (строки 2449-2478):
   // Цикл по 7 entry (от offset 0x188 до 0x70, step -0x1c)
   do {
       clear_bus_protection(regmap, &protection_entry);
   } while (...)
scpsys_bus_protect_enable() — строка 2922 — Установка bus protection при выключении:
// Строки 2960-3030:
do {
    // Определяем regmap по type (1=infracfg, 2=smi_comm, 3=vlpcfg, 4=mfgrpc)
    switch(*(uint*)(lVar11 + 0x8c)) {
        case 1: uVar12 = infracfg_regmap; break;
        case 2: uVar12 = smi_comm_regmap; break;
        case 3: uVar12 = vlpcfg_regmap; break;
        case 4: uVar12 = mfgrpc_regmap; break;
    }

    // Установка protection bits
    if (*(int*)(lVar11 + 0x90) == 0) {
        regmap_update_bits(regmap, addr, mask, mask);  // Set bits
    } else {
        regmap_write(regmap, write_addr, mask);         // Write directly
    }

    // Ожидание с таймаутом 1 сек
    while ((reg & mask) != mask) {
        if (timeout > 1s) break;
    }
} while (++index != 10);  // 10 bus protection entries
clear_bus_protection() — строка 2842 — Снятие bus protection:
// Строки 2864-2902:
if (*(int*)(param_2 + 8) == 0) {
    regmap_update_bits(regmap, clr_addr, mask, 0);  // Clear bits
} else {
    regmap_write(regmap, write_addr, mask);           // Write 0
}
// Ожидание с таймаутом 1 сек
if (*(char*)(param_2 + 0x18) == 0) {
    do {
        regmap_read(regmap, status_addr, &val);
        if ((val & mask) == 0) return 0;  // Protection cleared
        if (timeout > 1s) return -ETIMEDOUT;
    } while (...);
}
6. MT6789 SCPSYS PROBE: mt6789_scpsys_probe() — mtk-scpsys-mt6789.ko, строка 83
Файл: mtk-scpsys-mt6789.ko_detailed.c:83
// Строка 96-124:
puVar2 = of_device_get_match_data(dev);             // mt6789_data
init_scp(dev, *puVar2, count, data + 0x1c);        // Initialize SCPSYS subsystem
mtk_register_power_domains(dev, scp_data, count);   // Register all power domains
// Строка 102-115: Add subdomains
for (i = 0; i < subdomain_count; i++) {
    pm_genpd_add_subdomain(parent_pd, child_pd);
}
7. ПИНМУКС (pinctrl)
Из msdc_drv_probe() (строки 882-896):
// Получение pinctrl handles
devm_pinctrl_get(dev);                              // строка 882
pinctrl_lookup_state(pinctrl, "default");           // строка 885
pinctrl_lookup_state(pinctrl, "state_uhs");         // строка 891
// Переключение при смене напряжения (из msdc_ops_switch_volt, строка 3180):
if (signal_voltage == MMC_SIGNAL_VOLTAGE_180) {
    pinctrl_select_state(host->pinctrl, host->pinctrl_uhs);  // строка 3180
} else {
    pinctrl_select_state(host->pinctrl, host->pinctrl_default);
}
8. РЕГУЛЯТОРЫ (power supply)
msdc_drv_probe() (строка 802): mmc_regulator_get_supply() — vmmc, vqmmc
msdc_drv_probe() (строка 907): devm_regulator_get_optional("dvfsrc-vcore")
msdc_ops_set_ios() (строки 2937-3031):
  power_mode=OFF: mmc_regulator_set_ocr(vmmc, 0)     // строка 2962
  power_mode=ON:  mmc_regulator_set_ocr(vmmc, voltage) // строка 2991
  power_mode=UP:  regulator_enable(vqmmc)              // строка 2980
msdc_sd_power_off() (строка 488):
  mmc_regulator_set_ocr(vmmc, 0)                      // строка 501
  regulator_disable()                                  // строка 505
9. dvfsrc-vcore (Voltage Scaling)
msdc_drv_probe() строка 907: devm_regulator_get_optional("dvfsrc-vcore")
msdc_drv_probe() строка 914: of_property_read "req-vcore"
msdc_runtime_suspend() строка 1336-1338:
  regulator_set_voltage(dvfsrc-vcore, 0, INT_MAX)
msdc_runtime_resume() строка 1365-1368:
  regulator_set_voltage(dvfsrc-vcore, req_vcore, INT_MAX)
10. RUNTIME PM последовательность
msdc_runtime_suspend() (строка 1294):
  1. msdc_save_timing_setting()              // строка 1306
  2. Clear MSDC_INTEN bit 20                 // строка 1307-1310
  3. Если SDIO: enable_irq + irq_set_wake    // строки 1311-1314
  4. clk_bulk_disable(3) + unprepare(3)      // строка 1316-1317
  5. clk_disable + unprepare × 6 clocks      // строки 1318-1335
  6. regulator_set_voltage(dvfsrc, 0)         // строка 1336-1338
msdc_runtime_resume() (строка 1353):
  1. cpu_latency_qos_update_request(0)        // строка 1364
  2. regulator_set_voltage(dvfsrc, req)       // строка 1365-1368
  3. msdc_ungate_clock()                      // строка 1370
  4. msdc_restore_timing_setting()            // строка 1371
  5. disable_irq + irq_set_wake(0)           // строки 1372-1375
  6. MSDC_INTEN |= 0x100000                   // строка 1378-1380
11. INFRACFG (DCM) — mt6789_dcm.ko
Файл: mt6789_dcm.ko_detailed.c
dcm_infracfg_ao_base = ioremap(...)           // строка 758
// DCM control registers (infracfg_ao_base + offsets):
0x70: infra_bus_dcm                          // строки 884-902
0x74: peri_bus_dcm / peri_module_dcm         // строки 1047-1118
0xa0: infra_rx_p2p_dcm                       // строки 1001-1016
0xa4/0xa8: infra_conn_bus_dcm                // строки 955-970
0xac: infra_conn_bus_dcm status              // строка 945
0xa30: aximem_bus_dcm                        // строки 835-853
---
ИТОГОВАЯ СВОДКА: Последовательность включения eMMC
1. scpsys_mt6789_probe()
   └─> init_scp() → mtk_register_power_domains()
       └─> scpsys_power_on() для eMMC domain:
           a) regulator_enable("vmmc-supply") — optional
           b) clk_prepare + clk_enable × 3 (base clocks)
           c) MTCMOS ON: SPM registers (PWR_ON | PWR_ISO off)
           d) Wait power-on (1s timeout)
           e) PWR_RST_B=1, PWR_ISO=0, PWR_CLK_DIS=0
           f) clk_prepare + clk_enable × up to 10 (subsystem clocks)
           g) scpsys_sram_enable()
           h) clear_bus_protection() × up to 7 entries per domain
2. msdc_drv_probe()
   a) devm_platform_ioremap_resource() — MSDC MMIO base
   b) clk_get × 6 ("source", "sys_cg", "bus_clk", "crypto_clk", "source_cg", "crypto_cg")
   c) devm_clk_bulk_get_optional("pclk_cg", "axi_cg", "ahb_cg")
   d) reset_control_get("msdc")
   e) devm_pinctrl_get → lookup "default" + "state_uhs"
   f) devm_regulator_get_optional("dvfsrc-vcore")
3. msdc_ungate_clock()
   └─> clk_prepare + clk_enable × 6 individual + 3 bulk
   └─> Wait MSDC bit 7 (DLL ready)
4. msdc_init_hw()
   a) reset_control_assert → usleep(10,50) → reset_control_deassert
   b) MSDC_CFG |= SW_RST (bit 0)
   c) msdc_reset_hw() — full FIFO clear
   d) Configure FIFO, DMA, pad tuning, timing registers
   e) msdc_init_tune_setting()
5. devm_request_threaded_irq(msdc_irq)
6. mmc_add_host() — register with MMC core
