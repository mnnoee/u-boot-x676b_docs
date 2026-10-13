Полный анализ: MMC/MSDC, питание и домены из DTB stock firmware
---
1. MMC0 (eMMC) — mmc@11230000 (строки 4266-4293)
mmc0: mmc@11230000 {
    compatible = "mediatek,mt6789-mmc";
    reg = <0x00 0x11230000 0x00 0x10000 0x00 0x11f50000 0x00 0x1000>;
    //        base: 0x11230000, size: 0x10000  (MSDC0 base)
    //        base: 0x11f50000, size: 0x1000   (eMMC command DMA / SMC?)
    interrupts = <0x00 0x83 0x04 0x00>;
    clocks = <0x2a 0x13 0x2a 0x14 0x2a 0x32 0x2b 0x14 0x2b 0x16 0x2b 0x1d>;
    clock-names = "bus_clk", "source", "crypto_clk",
                  "hclk", "source_cg", "crypto_cg";
    status = "okay";
    host-index = <0x00>;           // HOST1 = MSDC0 = eMMC
    pinctrl-names = "default", "state_uhs";
    pinctrl-0 = <0x81>;            // -> mmc0_pins_default
    pinctrl-1 = <0x82>;            // -> mmc0_pins_uhs
    bus-width = <0x08>;            // 8-bit bus
    max-frequency = <0xbebc200>;   // 200 MHz
    ocr-voltage = <0x30000>;       // 1.8V + 3.0V dual voltage
    cap-mmc-highspeed;
    mmc-ddr-1_8v;
    mmc-hs200-1_8v;
    mmc-hs400-1_8v;
    no-sdio;
    no-sd;
    hs400-ds-delay = <0x12814>;    // HS400 data strobe delay
    vmmc-supply = <0x80>;          // -> mt6358_vemc_reg (VEMC LDO)
    non-removable;
    supports-cqe;
    dvfsrc-vcore-supply = <0x42>;  // -> dvfsrc_vcore
    req-vcore = <0x9eb10>;         // 650000 uV (0.65V) minimum vcore for eMMC
    phandle = <0x173>;
};
Ключевой вывод: eMMC питается от VEMC (phandle 0x80), dvfsrc-vcore (phandle 0x42).
---
2. MMC1 (SD card) — mmc@11240000 (строки 4295-4324)
mmc1: mmc@11240000 {
    compatible = "mediatek,mt6789-mmc";
    reg = <0x00 0x11240000 0x00 0x1000 0x00 0x11ef0000 0x00 0x1000>;
    //        base: 0x11240000, size: 0x1000  (MSDC1 base = SD)
    //        base: 0x11ef0000, size: 0x1000
    interrupts = <0x00 0x87 0x04 0x00>;
    clocks = <0x2a 0x15 0x2b 0x15 0x2b 0x1c>;
    clock-names = "source", "hclk", "source_cg";
    status = "okay";
    host-index = <0x01>;           // HOST2 = MSDC1 = SD card
    pinctrl-names = "default", "state_uhs";
    pinctrl-0 = <0x83>;            // -> mmc1_pins_default
    pinctrl-1 = <0x84>;            // -> mmc1_pins_uhs
    bus-width = <0x04>;            // 4-bit bus
    max-frequency = <0xbebc200>;   // 200 MHz
    ocr-voltage = <0x30000>;       // 1.8V + 3.0V
    cap-sd-highspeed;
    sd-uhs-sdr12;
    sd-uhs-sdr25;
    sd-uhs-sdr50;
    sd-uhs-sdr104;
    sd-uhs-ddr50;
    cd-debounce-delay-ms = <0x00>;
    cd-gpios = <0x49 0x07 0x00>;   // GPIO 7 for card detect
    vmmc-supply = <0x85>;          // -> mt6358_vmch_reg (VMCH LDO)
    vqmmc-supply = <0x86>;         // -> mt6358_vmc_reg (VMC LDO)
    no-mmc;
    no-sdio;
    dvfsrc-vcore-supply = <0x42>;  // -> dvfsrc_vcore
    req-vcore = <0x9eb10>;         // 650000 uV
    phandle = <0x174>;
};
Ключевой вывод: SD card питается от VMCH (phandle 0x85) + VMC (phandle 0x86) + dvfsrc-vcore (phandle 0x42).
---
3. SD Card Pin Groups (строки 1825-1909)
// === mmc0 (eMMC) pins ===
mmc0_pins_default: mmc0default {
    pins_cmd_dat {
        pinmux = <0x3a01 0x3b01 0x3c01 0x3d01 0x3e01 0x3f01 0x4001 0x4101 0x3801>;
        input-enable;
        drive-strength = <0x04>;    // 8mA
        bias-pull-up = <0x65>;
    };
    pins_clk {
        pinmux = <0x3701>;
        drive-strength = <0x04>;
        bias-pull-down = <0x66>;
    };
    pins_rst {
        pinmux = <0x3901>;
        drive-strength = <0x04>;
        bias-pull-up = <0x64>;
    };
};
mmc0_pins_uhs: mmc0@0 {
    // Same as default, plus:
    pins_ds {                        // Data strobe for HS400
        pinmux = <0x3601>;
        drive-strength = <0x04>;
        bias-pull-down = <0x66>;
    };
};
// === mmc1 (SD) pins ===
mmc1_pins_default: mmc1default {
    pins_cmd_dat {
        pinmux = <0x4901 0x4a01 0x4b01 0x4c01 0x4801>;
        input-enable;
        drive-strength = <0x03>;    // 4mA
        bias-pull-up = <0x65>;
    };
    pins_clk {
        pinmux = <0x4701>;
        drive-strength = <0x03>;
        bias-pull-down = <0x66>;
    };
};
mmc1_pins_uhs: mmc1@0 {
    // Same as mmc1_pins_default (no DS pin for SD)
};
---
4. SCPSYS Power Controller (строки 1034-1042)
scpsys: power-controller@10006000 {
    compatible = "mediatek,mt6789-scpsys", "syscon";
    reg = <0x00 0x10006000 0x00 0x1000>;
    #power-domain-cells = <0x01>;
    clocks = <...>;       // 30 clocks for all power domains
    clock-names = "mfg", "isp", "ipe", "vdec", "venc", "mdp",
                  "disp", "audio", "cam",
                  "isp-0", "isp-1", "ipe-0", "ipe-1", "ipe-2", "ipe-3",
                  "vdec-0", "vdec-1", "venc-0",
                  "disp_lp-0", "disp-0", "disp-1", "disp-2", "disp-3",
                  "audio-0", "audio-1",
                  "cam-0", "cam-1", "cam_rawa-0", "cam_rawb-0";
    infracfg = <0x2b>;
    phandle = <0x2d>;
};
---
5. Power Domain ID Map (phandle 0x2d = scpsys)
Все power-domains引用通过 <0x2d 0xNN>, где NN = domain ID:
ID (hex)	Domain	Используется в
0x06	isp (IMG)	camisp_legacy, disable-unused-pd-isp
0x07	ipe	disable-unused-pd-ipe
0x08	vdec	vdec (MDP3), disable-unused-pd-vdec
0x09	venc	jpgenc, disable-unused-pd-venc
0x0a	disp	dispsys_config, mmsys, mdpsys, disable-unused-pd-disp
0x0b	audio	disable-unused-pd-audio
0x0c	cam	seninf_top, camisp_legacy, disable-unused-pd-cam
0x0d	cam_rawa	cam1_legacy, disable-unused-pd-cam_rawa
0x0e	cam_rawb	camsys_rawb, disable-unused-pd-cam_rawb
Важно: MMC/MSDC НЕ имеет power-domain! В нодах mmc0 и mmc1 нет power-domains свойства. MSDC не зависит от scpsys power domain.
---
6. Disable-unused PDs (строки 904-947)
disable-unused-pd-isp    { compatible = "mediatek,scpsys-disable-unused"; power-domains = <0x2d 0x06>; };
disable-unused-pd-ipe    { compatible = "mediatek,scpsys-disable-unused"; power-domains = <0x2d 0x07>; };
disable-unused-pd-vdec   { compatible = "mediatek,scpsys-disable-unused"; power-domains = <0x2d 0x08>; };
disable-unused-pd-venc   { compatible = "mediatek,scpsys-disable-unused"; power-domains = <0x2d 0x09>; };
disable-unused-pd-disp   { compatible = "mediatek,scpsys-disable-unused"; power-domains = <0x2d 0x0a>; };
disable-unused-pd-audio  { compatible = "mediatek,scpsys-disable-unused"; power-domains = <0x2d 0x0b>; };
disable-unused-pd-cam    { compatible = "mediatek,scpsys-disable-unused"; power-domains = <0x2d 0x0c>; };
disable-unused-pd-cam_rawa { compatible = "mediatek,scpsys-disable-unused"; power-domains = <0x2d 0x0d>; };
disable-unused-pd-cam_rawb { compatible = "mediatek,scpsys-disable-unused"; power-domains = <0x2d 0x0e>; };
---
7. MSDCPLL Clock (строки 1116-1119)
msdcpll {
    fh-id = <0x0b>;
    pll-id = <0x06>;
};
Это определение PLL для MSDC — PLL ID 6, Frequency Handler ID 11.
---
8. PMIC Regulators — MT6358 (строки 2419-2828)
8a. vcore (buck) — phandle 0x56 (строки 2433-2441)
mt6358_vcore_reg: buck_vcore {
    regulator-name = "vcore";
    regulator-min-microvolt = <0x7a120>;     // 500000 uV (0.5V)
    regulator-max-microvolt = <0x13bdb6>;    // 1312000 uV (1.312V)
    regulator-enable-ramp-delay = <0xc8>;    // 200 us
    regulator-always-on;
    regulator-allowed-modes = <0x00 0x01>;
    phandle = <0x56>;
};
8b. vio18 (LDO, fixed) — phandle 0x11e (строки 2534-2542)
mt6358_vio18_reg: ldo_vio18 {
    compatible = "regulator-fixed";
    regulator-name = "vio18";
    regulator-min-microvolt = <0x1b7740>;    // 1800000 uV (1.8V)
    regulator-max-microvolt = <0x1b7740>;    // фиксированная 1.8V
    regulator-enable-ramp-delay = <0xa8c>;   // 2700 us
    regulator-always-on;
    phandle = <0x11e>;
};
8c. vio28 (LDO, fixed) — phandle 0x12d (строки 2703-2711)
mt6358_vio28_reg: ldo_vio28 {
    compatible = "regulator-fixed";
    regulator-name = "vio28";
    regulator-min-microvolt = <0x2ab980>;    // 2800000 uV (2.8V)
    regulator-max-microvolt = <0x2ab980>;    // фиксированная 2.8V
    regulator-enable-ramp-delay = <0x10e>;   // 270 us
    regulator-always-on;
    phandle = <0x12d>;
};
8d. vmch (LDO) — phandle 0x85 (строки 2660-2666)
mt6358_vmch_reg: ldo_vmch {
    regulator-name = "vmch";
    regulator-min-microvolt = <0x2c4020>;    // 2900000 uV (2.9V)
    regulator-max-microvolt = <0x325aa0>;    // 3300000 uV (3.3V)
    regulator-enable-ramp-delay = <0x3c>;    // 60 us
    phandle = <0x85>;                        // <-- SD vmmc-supply!
};
8e. vmc (LDO) — phandle 0x86 (строки 2756-2762)
mt6358_vmc_reg: ldo_vmc {
    regulator-name = "vmc";
    regulator-min-microvolt = <0x1b7740>;    // 1800000 uV (1.8V)
    regulator-max-microvolt = <0x325aa0>;    // 3300000 uV (3.3V)
    regulator-enable-ramp-delay = <0x3c>;    // 60 us
    phandle = <0x86>;                        // <-- SD vqmmc-supply!
};
8f. vemc (LDO) — phandle 0x80 (строки 2695-2701)
mt6358_vemc_reg: ldo_vemc {
    regulator-name = "vemc";
    regulator-min-microvolt = <0x2c4020>;    // 2900000 uV (2.9V)
    regulator-max-microvolt = <0x325aa0>;    // 3300000 uV (3.3V)
    regulator-enable-ramp-delay = <0x3c>;    // 60 us
    phandle = <0x80>;                        // <-- eMMC vmmc-supply!
};
8g. vmch_eint_high/low (строки 2813-2827)
mt6358_vmch_eint_high: VMCH_EINT_HIGH {
    regulator-name = "mt6358_vmch_eint_high";
    regulator-min-microvolt = <0x2c4020>;    // 2.9V
    regulator-max-microvolt = <0x325aa0>;    // 3.3V
    regulator-enable-ramp-delay = <0x3c>;
    phandle = <0x136>;
};
mt6358_vmch_eint_low: VMCH_EINT_LOW {
    regulator-name = "mt6358_vmch_eint_low";
    regulator-min-microvolt = <0x2c4020>;    // 2.9V
    regulator-max-microvolt = <0x325aa0>;    // 3.3V
    regulator-enable-ramp-delay = <0x3c>;
    phandle = <0x137>;
};
8h. vcore_sshub (строки 2790-2795)
mt6358_vcore_sshub_reg: buck_vcore_sshub {
    regulator-name = "vcore_sshub";
    regulator-min-microvolt = <0x7a120>;     // 0.5V
    regulator-max-microvolt = <0x13bdb6>;    // 1.312V
    phandle = <0x59>;
};
---
9. DVFSRC vcore (строки 3035-3048)
dvfsrc: dvfsrc@10012000 {
    compatible = "mediatek,mt6789-dvfsrc";
    reg = <0x00 0x10012000 0x00 0x1000 0x00 0x10006000 0x00 0x1000>;
    //      dvfsrc reg               spm reg (shared with SCPSYS!)
    #interconnect-cells = <0x01>;
    phandle = <0x22>;
    dvfsrc_vcore: dvfsrc-vcore {
        regulator-name = "dvfsrc-vcore";
        regulator-min-microvolt = <0x86470>;   // 550000 uV (0.55V)
        regulator-max-microvolt = <0xb1008>;   // 725000 uV (0.725V)
        regulator-always-on;
        phandle = <0x42>;
    };
};
Это phandle 0x42 — основной vcore-supply для MMC0, MMC1, cam, display, и многих других модулей.
---
10. DVFSRC Helper (строки 3093-3101)
dvfsrc-helper {
    compatible = "mediatek,dvfsrc-helper";
    vcore-supply = <0x56>;           // mt6358_vcore_reg (бак)
    rc-vcore-supply = <0x42>;        // dvfsrc_vcore (обратная связь)
    rc-vscp-supply = <0x57>;         // dvfsrc_vscp
    interconnects = <0x22 0x13 0x22 0x00 0x22 0x13 0x22 0x00 0x22 0x1c 0x22 0x14>;
    interconnect-names = "icc-bw", "icc-perf-bw", "icc-hrt-bw";
    required-opps = <0x23 0x24 0x25 0x26 0x27 0x28>;
};
---
11. SCP DVFS — sshub vcore (строки 3123-3136)
scp_dvfs {
    compatible = "mediatek,scp_dvfs";
    ...
    sshub-vcore-supply = <0x59>;     // -> mt6358_vcore_sshub_reg
    sshub-vsram-supply = <0x5a>;     // -> mt6358_vsram_others_sshub_reg
    ...
};
---
12. Display — MMSYS power domain (строки 4895-4918)
dispsys_config: dispsys_config@14000000 {
    compatible = "mediatek,mt6789-disp";
    reg = <0x00 0x14000000 0x00 0x1000>;
    power-domains = <0x2d 0x0a>;     // -> DISP power domain
    dvfsrc-vcore-supply = <0x42>;    // -> dvfsrc_vcore
    ...
};
---
13. DSC wrapper @ 0x14012000 (строка 5065-5071)
disp_dsc_wrap0: disp_dsc_wrap0@14012000 {
    compatible = "mediatek,disp_dsc_wrap", "mediatek,mt6789-disp-dsc";
    reg = <0x00 0x14012000 0x00 0x1000>;
    interrupts = <0x00 0x12c 0x04 0x00>;
    clocks = <0x35 0x0b>;
    phandle = <0x186>;
};
---
14. msdc2_base / msdc3_base (строки 5631-5633)
msdc2_base = <0x17020000 0x14 0xffff0000>;
msdc3_base = <0x18000000 0x16 0xffff0000>;
Это DMA base addresses для MSDC2 и MSDC3 (используются модемом).
---
15. MSDC1 Insert EINT (строки 3228-3230)
msdc1_ins: msdc1_ins {
    phandle = <0x146>;
};
Пустой нод — SD card insert event interrupt.
---
16. Overlays — vmch-supply для fingerprint (overlays.txt строка 1118)
fingerprint@0 {
    compatible = "tran_fp";
    ...
    vmch-supply = <&mt6358_vio28_reg>;   // Fingerpint питается от VIO28 (2.8V)!
    ...
};
Важно: В overlay fingerprint использует vmch-supply, но подключён к vio28 (2.8V), а НЕ к VMCH! Это переназначение в overlay.
---
17. Итоговая карта связей (Supply Chain)
Устройство	Supply property	Phandle	PMIC LDO/Buck	Напряжение
eMMC (mmc0)	vmmc-supply	0x80	vemc	2.9-3.3V
 	dvfsrc-vcore-supply	0x42	dvfsrc-vcore	0.55-0.725V
SD (mmc1)	vmmc-supply	0x85	vmch	2.9-3.3V
 	vqmmc-supply	0x86	vmc	1.8-3.3V
 	dvfsrc-vcore-supply	0x42	dvfsrc-vcore	0.55-0.725V
Display	power-domains	0x2d:0x0a	scpsys DISP	-
 	dvfsrc-vcore-supply	0x42	dvfsrc-vcore	0.55-0.725V
Fingerprint	vmch-supply	overlay → 0x12d	vio28	2.8V (fixed)
Cap Touch	vtouch-supply	0x12d	vio28	2.8V (fixed)
Core SoC	always-on	0x56	vcore	0.5-1.312V
SCP	sshub-vcore-supply	0x59	vcore_sshub	0.5-1.312V
---
Ключевые выводы для U-Boot:
1. MSDC0 (eMMC): base = 0x11230000, не имеет scpsys power-domain. Питание от VEMC (LDO, 2.9-3.3V) и dvfsrc-vcore (0.55-0.725V).
2. MSDC1 (SD): base = 0x11240000, не имеет scpsys power-domain. Питание от VMCH (LDO, 2.9-3.3V) и VMC (LDO, 1.8-3.3V) и dvfsrc-vcore.
3. SDMMC PLL: pll-id = 6, fh-id = 11 (msdcpll).
4. SCPSYS @ 0x10006000 — общий power domain controller. MMC к нему НЕ подключён.
5. VIO18 и VIO28 оба regulator-always-on с фиксированным напряжением (1.8V и 2.8V соответственно).
6. VEMC — динамический LDO (2.9-3.3V, ramp 60us), не always-on — управляется драйвером.
7. VMCH — динамический LDO (2.9-3.3V, ramp 60us), не always-on.
8. VMC — динамический LDO (1.8-3.3V, ramp 60us), не always-on — переключается между 1.8V (UHS) и 3.3V (legacy).
