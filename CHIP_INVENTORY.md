# PXW-Z190 — Chip / IC Inventory

Hardware identified from the **PXW-Z190V Service Manual** (block diagrams pp99–101, circuit descriptions pp6–9, LED indicators p13, board-replacement notes pp14–16) cross-referenced with public chip databases.

Tags: **[SM]** = Service Manual · **[PUB]** = public datasheet / openmemories / Wikipedia · **[INF]** = inference from signal names and block-diagram context.

---

## 1. Main Processing Chain (VC-41 Board)

The VC-41 is the main board. It carries four key ICs that form the entire image-acquisition → processing → encoding → storage pipeline, plus supporting glue.

### 1.1 IC0100 — Digital Front-End (DFE) "TIGA"

| Field | Detail |
|-------|--------|
| **Board** | VC-41 |
| **Role** | Sensor interface / digital front-end. Receives raw MIPI data from three CMOS imagers (R/G/B), distributes processed image data to IC0600 and IC6000. **[SM]** |
| **Block-diagram label** | Unlabeled box between imagers and ROSE/AST; signal names `(CIS)→(DFE)` and `(DFE)→(ROSE)` **[SM]** |
| **Internal codename** | **TIGA** (LED D0200 "TIGA" indicates IC0100 configuration complete) **[SM]** |
| **Interface** | MIPI CSI-2 from imagers (12 differential data lanes × 3 channels), parallel/LVDS output to IC0600 (ROSE) **[SM/INF]** |
| **Likely CXD part** | CXD90009-class (Sirius) or successor DFE — Sony's dedicated sensor-bridge chip family. Starting with CXD90016 the DFE integrates an ARM core. **[PUB]** |
| **Capabilities** | Sensor timing, black-level clamping, defect correction (RPN/APR executed through service menu), per-channel gain/offset. **[SM/INF]** |

### 1.2 IC0600 — Camera DSP / ISP "ROSE"

| Field | Detail |
|-------|--------|
| **Board** | VC-41 (spans 3 diagram sheets) |
| **Role** | **Main-line camera signal processing.** The block diagram labels it *"CPU, Camera DSP, Video I/F, Mode Control, HDMI Process."* It is the camera's primary image signal processor. **[SM]** |
| **Internal codename** | **ROSE** (LED D1200 "ROSE" indicates IC0600 operating state; signal-bus names throughout: `(ROSE)→(MONI)`, `(ROSE)→(BONU)`, `(ROSE)→(ARIF)`, `(ROSE)→(ADSP)`, `(ROSE)→(FAN)_PWM`) **[SM]** |
| **Key functions** | Camera DSP (white balance, gamma, knee, detail, matrix), video output routing (SDI/HDMI/EVF/LCD monitor), HDMI processing, audio path routing, tally control, mode/state control. **[SM]** |
| **Interfaces** | Parallel video from DFE (IC0100), encode/decode lanes to IC1600 (BONOBO), monitor output to IC2300, EMIF address bus to IC6000 (AST), SPI, I²C. **[SM]** |
| **Likely CXD part** | **CXD4239** (codename **Rosetta**) — quad-core ARM Cortex-A5, Sony's professional ISP released Apr 2018, confirmed in sibling PXW-Z280 and PXW-FX9. The Z190 launched at the same NAB 2018 event with the same "new front-end LSI." **[PUB/INF]** |
| **Notable** | Also generates monitor signals sent through IC2300 to SDI/HDMI outputs. Drives IC0900/IC0901 LVDS serializers for EVF and LCD panel. Controls fan PWM. **[SM]** |

### 1.3 IC1600 — Codec / Encoder-Decoder "BONOBO"

| Field | Detail |
|-------|--------|
| **Board** | VC-41 |
| **Role** | **Video CODEC** — encodes main-line signals from IC0600 for recording, decodes for playback. Block diagram labels it *"CODEC."* **[SM]** |
| **Internal codename** | **BONOBO** (abbreviated **BONU** / **BN** in signal names: `(ROSE)→(BONU)_ENC`, `(BONU)→(ROSE)_AU_PB`, `(ROSE)→(BONU)_AU_RECPB`; LEDs D1900/D1901 "BN" indicate IC1600 state) **[SM]** |
| **Key functions** | XAVC / XAVC-S / MPEG-2 HD422 encoding and decoding. Audio record/playback path. Encoded data forwarded to IC6000 (AST) via IC0700 for storage. **[SM/INF]** |
| **Interfaces** | Dedicated encode/decode differential lanes to/from IC0600, audio record/playback bus, data path to IC0700 on AR-5 board. **[SM]** |
| **Likely CXD part** | **CXD4236** (codename **Beaune**) — Sony's MIPS-based video codec, used in ILCE-7RM2-era and later professional bodies. Or its direct successor for the 2018 generation. **[PUB/INF]** |
| **Notable** | The MPEG HD upgrade license (CBKZ-SLMP) gates the MPEG HD422/HD420 codec paths — this chip contains the codec hardware that the license flag enables. **[INF]** |
| **Power rails** | `+3.3V_BN`, `+1.8V_BN`, `+1.2V_BN`, `+1.1V_BN`, `+0.4V_BN` — five dedicated rails. **[SM]** |

### 1.4 IC6000 — Application SoC "AST"

| Field | Detail |
|-------|--------|
| **Board** | VC-41 (primary), also referenced on JK-106 (Ethernet), GP-1019 (GPS), LD-136 (lens), DD-53 (power) |
| **Role** | **Main application processor / SoC.** Block diagram labels it *"VIDEO/AUDIO I/F, I/O EXPANDER."* Service manual §1-3-31: *"IC6000 detects the image signal to control the lens… performs proxy recording and network control."* Board-replacement notes call it *"the main IC (CPU)."* **[SM]** |
| **Internal codename** | **AST** (signal-bus names throughout: `(AST)→(ROSE)`, `(AST)→(DFE)`, `(AST)→(BONU)`, `(AST)→(GPS)`, `(AST)→(DWN)`, `(AST)→(PANDA)`, `(AST)→(CMR)`) **[SM]** |
| **Key functions** | Runs Linux OS, manages SD card storage (UHS-II via PCIe switch), USB 3.0 host, wired LAN (via PCIe), Wi-Fi SDIO, GPS UART, NFC, proxy recording, network streaming (RTMP/RTMPS/MPEG-TS), FTP, lens control (SIO to LD-136), Multi Interface Shoe UART, LANC remote, firmware update ("Version Up"), menu system, license store management. **[SM]** |
| **Interfaces** | PCIe (via IC0300 switch to SD slots, Ethernet IC2000, USB IC2602), EMIF bus to IC0600, MIPI to IC2300, SDIO to Wi-Fi (CP7330), UART × 4 (debug, shoe front, shoe rear, GPS), SIO × 4 (lens, power, VUP3, other), I²C × 2 (EEPROM, peripherals), ADC inputs (keys, battery, zoom). **[SM]** |
| **Likely CXD part** | **CXD90045** (codename **Astra**) — quad-core ARM Cortex-A5, Sony's BIONZ X SoC released Apr 2017, or a close variant for the professional line. The "AST" signal prefix maps to "Astra." The chip runs Linux + µITRON RTOS. **[PUB/INF]** |
| **Notable** | This is the chip that holds the firmware key in its fuse/key-store, manages the license NVRAM, and runs the encrypted firmware payload. The fwtool.py project has the CXD90045 key for consumer bodies but it failed on the Z190 — the pro variant likely uses a different key. **[INF]** |
| **Stored data** | Product ID, USB Serial No., MAC address (reloaded on board replacement via Adjust Station), destination data, EVF calibration — all written to this board. **[SM]** |

---

## 2. Video Output (VC-41 Board)

### 2.1 IC2300 — SDI/HDMI Monitor Driver "MONI"

| Field | Detail |
|-------|--------|
| **Board** | VC-41 |
| **Role** | Video output interface — receives monitor signals from IC0600 (ROSE) and drives SDI OUT (12G/3G/HD/SD selectable) and HDMI OUT. **[SM]** |
| **Internal codename** | **MONI** (signal names: `(ROSE)→(MONI)_MON4`, `(MONI)→(SDI)`, `(MONI)→(HDMI)`) **[SM]** |
| **LED** | D2400 "DONE" indicates IC2300 configuration complete. **[SM]** |

### 2.2 IC800 — HDMI Interface / DDR3

| Field | Detail |
|-------|--------|
| **Board** | VC-41 |
| **Role** | Block diagram labels it *"HDMI I/F, DDR3."* Likely an HDMI transmitter with DDR3 frame buffer. **[SM]** |

### 2.3 IC101 — SDI Cable Driver (SD-66 Board)

| Field | Detail |
|-------|--------|
| **Board** | SD-66 |
| **Role** | SDI cable driver. Receives SDI signals from IC2300 and control from IC0600, outputs to BNC connector. **[SM]** |

---

## 3. Audio Subsystem (AR-5 Board)

### 3.1 IC0700 — Audio/Video Interface "ARIF"

| Field | Detail |
|-------|--------|
| **Board** | AR-5 |
| **Role** | Interface conversion between IC0600 (ROSE) and IC6000 (AST), plus audio function. Routes encoded data from IC1600 (BONOBO) to IC6000 for SD storage. **[SM]** |
| **Internal codename** | **ARIF** (signal names: `(ARIF)→(AST)`, `(ARIF)→(ROSE)`, `(ARIF)→(SPI)`, `(ARIF)→(AUDSP)`) **[SM]** |
| **LED** | D0851 "DONE" indicates IC0700 configuration complete. **[SM]** |

### 3.2 IC1000 — Audio DSP "AUDSP"/"ADSP"

| Field | Detail |
|-------|--------|
| **Board** | AR-5 |
| **Role** | Audio DSP — processes audio from internal mic, XLR inputs, and Multi Interface Shoe. **[SM]** |
| **LED** | D1000 "DSP" indicates IC1000 operating state. **[SM]** |

---

## 4. Storage & Connectivity

### 4.1 IC0300 — PCIe Switch (AR-5 / DD-53)

| Field | Detail |
|-------|--------|
| **Board** | AR-5 (PCIe distribution), DD-53 (switching regulator — different IC0300) |
| **Role** | **PCIe switch.** Distributes PCIe from IC6000 (AST) to four devices: IC2000 (SD slot A controller), IC2101 (SD slot B controller), IC2602 (USB), and wired LAN. **[SM]** |

### 4.2 IC2000, IC2101 — SD Card Controllers (UHS-II)

| Field | Detail |
|-------|--------|
| **Board** | AR-5 (IC2101), JK-106 area (IC2000) |
| **Role** | SD card UHS-II controllers. PCIe-to-SDIO bridge for memory card slots A and B. **[SM]** |
| **Interface** | PCIe from IC0300, UHS-II differential to card slots. **[SM]** |

### 4.3 IC2602 — USB Host Controller

| Field | Detail |
|-------|--------|
| **Board** | VC-41 / AR-5 |
| **Role** | USB host controller — drives USB 3.0 (Type A) and USB 2.0 (Type A, for LTE dongle). **[SM]** |
| **Clock** | X2600 24 MHz. **[SM]** |

### 4.4 IC2000 (JK-106) — Wired LAN Adapter

| Field | Detail |
|-------|--------|
| **Board** | JK-106 |
| **Role** | USB-to-Ethernet adapter IC. Connects via PCIe through IC0300 to IC6000. Drives RJ-45 (1000BASE-T / 100BASE-T / 10BASE-T). **[SM]** |
| **Clock** | X6000 25 MHz. **[SM]** |

### 4.5 CP7330 — Wi-Fi Module (CT-287A Board)

| Field | Detail |
|-------|--------|
| **Board** | CT-287A |
| **Role** | Wireless LAN module. 2.4 GHz + 5 GHz (Z190V) or 2.4 GHz only (Z190T). SDIO interface to IC6000 (AST). Antenna on WF-1008 board. **[SM]** |
| **Interface** | `WLAN_SDIO_D0-3` to IC6000. **[SM]** |
| **Notable** | MAC address stored on this module; must be reloaded after board replacement. **[SM]** |

---

## 5. Lens Control (LD-136 Board)

### 5.1 IC9600 — Image Stabilization Controller

| Field | Detail |
|-------|--------|
| **Board** | LD-136 |
| **Role** | Drives OIS vibration-proof lens using gyro signals from SE-1187 (yaw/pitch sensors). **[SM]** |

### 5.2 IC9700 — Iris / ND Controller

| Field | Detail |
|-------|--------|
| **Board** | LD-136 |
| **Role** | Controls iris and variable liquid-crystal ND filter. Firmware downloaded from IC6000 at boot. **[SM]** |

### 5.3 IC9800 — Focus / Shutter Controller

| Field | Detail |
|-------|--------|
| **Board** | LD-136 |
| **Role** | Controls focus motor and mechanical shutter. Receives zoom/focus MR sensor signals. **[SM]** |

### 5.4 IC9550, IC9551 — Zoom Motor Drivers

| Field | Detail |
|-------|--------|
| **Board** | LD-136 |
| **Role** | Drive the zoom lens motor. **[SM]** |

### 5.5 IC4804 — Secondary Lens Controller (CT-287A)

| Field | Detail |
|-------|--------|
| **Board** | CT-287A |
| **Role** | Lens control using signals from IC6000 (AST). **[SM]** |

---

## 6. Display

### 6.1 IC4002 — LCD/EVF Controller "VINE" (IF-1346 Board)

| Field | Detail |
|-------|--------|
| **Board** | IF-1346 |
| **Role** | VF controller — communicates with IC6000 (AST) to control both LCD panel and EVF. Detects panel open/close/reverse state, inside-panel key inputs. **[SM]** |
| **Internal codename** | **VINE** (signal names: `(VINE)→(VF_LCD)_SO`) **[SM]** |

### 6.2 IC1101 — EVF Display Controller (VF-98)

| Field | Detail |
|-------|--------|
| **Board** | VF-98 |
| **Role** | Drives the 0.39-inch color OLED EVF. Receives serial LVDS from IC0900 on VC-41. **[SM]** |

### 6.3 IC6001 — LCD Panel Driver (PD-126)

| Field | Detail |
|-------|--------|
| **Board** | PD-126 |
| **Role** | LVDS serial-to-parallel converter driving the 3.5-inch color LCD panel. **[SM]** |

### 6.4 IC0900, IC0901 — LVDS Serializers (VC-41)

| Field | Detail |
|-------|--------|
| **Board** | VC-41 |
| **Role** | Parallel-to-serial LVDS converters for EVF (IC0900) and LCD panel (IC0901). **[SM]** |

---

## 7. Power Management (DD-53 Board)

### 7.1 IC5700 — Power Controller

| Field | Detail |
|-------|--------|
| **Board** | DD-53 |
| **Role** | Receives power switch signal from PSW-109, controls system power ON/OFF. Also controls system reset. **[SM]** |
| **Codename hint** | **DWN** (signal names: `(AST)→(DWN)_SIO3`, `(DWN)→XSYS_RST`) **[SM]** |
| **Clock** | X5701 12 MHz, X5700 32.768 kHz (RTC). **[SM]** |

### 7.2 IC5100 / IC0300 (DD-53) — Switching Regulators

| Field | Detail |
|-------|--------|
| **Board** | DD-53 |
| **Role** | Convert 5V to multiple rails for all ICs. **[SM]** |

### 7.3 IC0501, IC0201 — 5V Regulators

| Field | Detail |
|-------|--------|
| **Board** | DD-53 |
| **Role** | DC-DC conversion from battery/DC-IN (11–17V unregulated) to 5V. **[SM]** |

---

## 8. Miscellaneous

### 8.1 IC2200 — CPLD "VUP3"

| Field | Detail |
|-------|--------|
| **Board** | VC-41 |
| **Role** | CPLD (Complex Programmable Logic Device). Connected to IC6000 (AST) via SIO. Likely handles firmware update sequencing and programmable logic. **[SM]** |
| **Internal codename** | **VUP3** (LED D2200 "VUP3" indicates IC2200 state; signal: `(AST)→(VUP3)_SIO4`) **[SM]** |
| **Notable** | "VUP" likely = "Version Up" — this CPLD may gate the firmware update process. **[INF]** |

### 8.2 IC0507 — RTC (Real-Time Clock)

| Field | Detail |
|-------|--------|
| **Board** | VC-41 |
| **Role** | Real-time clock. 32 kHz signal to IC2300 (MONI) for timecode. **[SM]** |

### 8.3 IC0852 / IC3004 — EEPROMs

| Field | Detail |
|-------|--------|
| **Board** | AR-5 (IC0852), VC-41 area (IC3004) |
| **Role** | Non-volatile storage. IC0852 near UTILITY SD/MS slot. IC3004 near memory card slots. Connected via I²C to IC6000 (AST). **[SM]** |
| **Notable** | These are candidates for license/calibration data storage. The AST SoC reads them at boot via `(AST)_I2C_SDA0/SCL0`. **[INF]** |

### 8.4 IC7401 — NFC Interface

| Field | Detail |
|-------|--------|
| **Board** | IO-283 |
| **Role** | NFC controller. SIO/SCK to IC6000 (AST). **[SM]** |

### 8.5 IC7502 — GPS Interface (GP-1019)

| Field | Detail |
|-------|--------|
| **Board** | GP-1019 |
| **Role** | GPS module interface. UART to IC6000 (AST). 16.388 MHz crystal (X7501). **[SM]** |

### 8.6 IC5200 — LANC Remote Controller (DD-53)

| Field | Detail |
|-------|--------|
| **Board** | DD-53 |
| **Role** | Receives LANC signal from TC-117 REMOTE connector for remote-control operation. **[SM]** |

---

## 9. Image Sensors (BI-342 / BI-350 Boards)

| Board | Channel | Type | Specs |
|-------|---------|------|-------|
| **BI-350** | G (green) | 1/3-type Exmor R CMOS, back-illuminated | 3840 × 2160 effective pixels (≈8.3 MP) **[PUB]** |
| **BI-342** (×2) | R (red), B (blue) | 1/3-type Exmor R CMOS, back-illuminated | Same resolution **[PUB]** |

Each sensor board carries an IC1001 and connects to IC0100 (DFE) via 12 MIPI differential data lanes + clock. **[SM]**

---

## 10. Internal Codename Map

Summary of all internal codenames found in service-manual signal-bus naming:

| Codename | IC | Role |
|----------|----|------|
| **AST** (Astra) | IC6000 | Application SoC — runs Linux, storage, network, lens, license |
| **ROSE** (Rosetta) | IC0600 | Camera DSP / ISP — image processing, video output, mode control |
| **BONU** / **BN** (Bonobo) | IC1600 | Video codec — XAVC/MPEG encode/decode |
| **TIGA** | IC0100 | Digital front-end — sensor interface |
| **MONI** | IC2300 | Monitor output — SDI/HDMI driver |
| **ARIF** | IC0700 | Audio/video interface bridge |
| **VINE** | IC4002 | LCD/EVF display controller |
| **ADSP** / **AUDSP** | IC1000 | Audio DSP |
| **DWN** | IC5700 | Power controller |
| **VUP3** | IC2200 | CPLD — version-up / programmable logic |
| **PANDA** | ? | Unknown — SIO1 partner of AST (serial line only) |
| **CMR** | ? | Unknown — SIO2 partner of AST (likely camera/imager control) |

---

## 11. Security-Relevant Observations

1. **IC6000 (AST / Astra-class SoC)** is the security-critical chip. It holds the firmware decryption key (fused), runs the license store, manages the encrypted firmware update, and runs the main Linux OS. Board replacement requires re-writing Product ID, USB Serial No., and MAC address via Sony's Adjust Station — confirming device-unique identity data resides here. **[SM]**

2. **IC0852 / IC3004 (EEPROMs)** on the I²C bus are the most likely physical locations for license records and calibration data. They are readable/writable by IC6000 and are non-volatile. **[SM/INF]**

3. **IC2200 (VUP3 CPLD)** participates in the firmware update chain and may enforce update-sequencing logic in hardware. **[SM/INF]**

4. **Debug UART** is present: signal `(DBG)→(AST)_UART0_RXD/TXD` — a dedicated debug serial port on IC6000 (AST). Whether it is enabled/authenticated on production units is assessment property **P9** from the companion plan. **[SM]**

---

## 12. Sources

- **Sony PXW-Z190V Service Manual** (1st Edition) — this repo, all sections. **[SM]**
- [OpenMemories Hardware Reference](https://openmemories.readthedocs.io/hardware.html) — community-compiled Sony chip database (CXD part numbers, codenames, architectures). **[PUB]**
- [Wikipedia — BIONZ](https://en.wikipedia.org/wiki/Bionz) — Sony image processor family overview. **[PUB]**
- [Wex Photo Video — NAB 2018 Z190/Z280 announcement](https://www.wexphotovideo.com/blog/pro-video/sony-announces-two-4k-handheld-cameras--z190-and-z280--nab-2018) — "new front-end LSI" reference. **[PUB]**
- [Newsshooter — PXW-Z280/Z190](https://www.newsshooter.com/2018/03/26/sony-pxw-z280-z190/) — NAB 2018 product details. **[PUB]**
