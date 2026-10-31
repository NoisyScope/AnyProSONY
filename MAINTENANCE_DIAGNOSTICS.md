# PXW-Z190 Maintenance & Diagnostic Mode Analysis

**Objective:** Determine whether a Sony service technician can boot the PXW-Z190 in a maintenance/diagnostic mode to test Wi-Fi capabilities without replacing hardware.

---

## Executive Summary

The PXW-Z190 service manual does **not** document a dedicated maintenance boot mode or Wi-Fi diagnostic mode. However, Sony provides a **multi-layered service toolchain** that allows technicians to diagnose Wi-Fi issues at several levels — from on-camera checks to PC-based calibration — before resorting to board replacement.

**Key finding:** A technician can isolate software vs. hardware Wi-Fi problems using the documented procedures below, but the deepest diagnostic access (Adjust Station, debug UART) requires Sony-proprietary tools not publicly available.

---

## 1. On-Camera Service Menu

**Entry sequence:** MENU → System → Hours Meter → hold SEL/SET dial + Assign1 → press MENU

The Service Menu provides hardware calibration but **no Wi-Fi diagnostics**:

| Menu Item | Purpose | Wi-Fi Relevant? |
|-----------|---------|-----------------|
| CMOS Adjustment | Imager registration | No |
| White Shading | Shading correction | No |
| Auto Black Balance | Black balance calibration | No |
| Black Shading | Black shading correction | No |
| Auto Shading | Auto shading correction | No |
| Flare | Flare compensation | No |
| Auto FB Adjust | Flange back adjustment | No |
| RPN | Random-point noise compensation | No |
| Service Reset | Reset service menu to factory | No |
| Factory Reset | Reset all settings to factory | **Partial** — may clear corrupt Wi-Fi config |
| Information | Serial number, board temp, **Log Dump** | **Yes** — error logs may contain Wi-Fi failures |
| Version | Firmware version display | **Indirect** — confirms firmware state |

### Log Dump (Information → Log Dump)

Stores error log data to the active recording medium. This is the most relevant on-camera diagnostic: a technician can export logs and search for wireless-subsystem errors. The service manual does not describe the log format, but it captures operational errors that may include network stack failures.

---

## 2. Wi-Fi Functional Check (Service Manual §1-7-4)

The documented Wi-Fi verification procedure (p. 23):

1. Turn on the camera
2. Enable Wi-Fi in the Network menu
3. Verify the password displays on camera screen
4. Use a PC or smartphone to detect the camera as a connectable Wi-Fi device

**Assessment:** This is a basic pass/fail test designed for post-reassembly verification. If the camera's SSID appears and a device can connect, the Wi-Fi hardware is functional. If it does not appear at all:
- **Software cause:** Corrupt configuration, firmware issue, or disabled WLAN setting
- **Hardware cause:** Failed Wi-Fi module (IC0630, Marvell 88W8897) or antenna connection

A technician can narrow this further:
- **Factory Reset** (Service Menu) clears all settings, including any corrupt network config
- **Firmware reinstall** eliminates firmware corruption
- If Wi-Fi still fails after both → hardware fault

---

## 3. Adjust Station / Adjust Manual (PC-Based Service Tools)

**Adjust Station** is Sony's proprietary PC software for service technicians, connecting to the camera via USB. The service manual (p. 24) describes it as:

> *"The software to adjust and check digital cameras and camcorders for service"*

It provides two main interfaces:
- **ADJUST tab** — hardware calibration and parameter adjustment
- **DATA tab** — reading/writing calibration data and device information

### Wi-Fi-Specific Capability: MAC Address Programming

The service manual (p. 22) documents one explicit Wi-Fi function:

> **Adjust Station → ADJUST tab → "Wireless LAN Setting (MAC Address)"**

After writing the MAC address, the technician must perform a **"NETWORK RESET"** on the camera.

**Implications:**
- Adjust Station has direct access to the WLAN subsystem at a hardware/firmware level
- It can read and write EEPROM-stored network parameters (MAC address, possibly calibration data)
- The ADJUST tab likely exposes additional network diagnostic fields not documented in the service manual
- The DATA tab may allow reading Wi-Fi module status or error registers

### Limitation

Adjust Station is **not publicly available**. It is distributed exclusively to Sony authorized service centers and requires specific hardware interface cables.

---

## 4. Hardware Debug Interfaces

The service manual's frame wiring diagram (p. 109) reveals three physical debug/service connections:

### 4.1 Debug UART

Signal path from the schematic:
```
(DBG) → (AST)_UART0_RXD  (IC6000, pin — CXD90045 Astra SoC)
(DBG) → (AST)_UART0_TXD  (IC6000, pin — CXD90045 Astra SoC)
```

This is a serial console connection to the main application processor. On BIONZ X cameras running Linux + µITRON, this UART typically provides:
- Boot console output (U-Boot / kernel messages)
- Linux shell access (if enabled in build)
- System diagnostics and log output

**Security assessment:** The UART is physically present but likely requires:
- A 3.3V TTL serial adapter
- Partial disassembly to access test points
- The console may be disabled or require authentication in production firmware

### 4.2 "FW UP" Connection

Located near the VC-41 board area in the wiring diagram. This is likely a dedicated firmware update interface separate from the standard SD-card update path, used by Sony service centers for:
- Recovery from bricked firmware
- Loading service/debug firmware builds
- Factory provisioning

### 4.3 "DEBUG" Connections

Two additional "DEBUG" labels appear in the wiring diagram near:
- The GP-1019 area (possibly a GPIO/test connector)
- The USB3.0 area

These may provide JTAG or SWD access to the various processors (Astra, Rosetta, Beaune).

---

## 5. Software-Level Wi-Fi Diagnostic Path

Based on the documented tools and camera architecture, a technician can diagnose Wi-Fi issues through this escalation path:

### Level 1: On-Camera (No Special Tools)

1. **Factory Reset** (Service Menu → Factory Reset) — eliminates corrupt config
2. **Firmware reinstall** (download V4.21 from Sony support, install via SD card) — eliminates firmware corruption
3. **Wi-Fi functional check** (§1-7-4) — basic pass/fail
4. **Log Dump** (Service Menu → Information → Log Dump) — capture error logs

### Level 2: Adjust Station (Authorized Service Center)

5. **Read Wi-Fi parameters** via Adjust Station ADJUST tab
6. **Reprogram MAC address** if corrupted (Wireless LAN Setting)
7. **Network Reset** on camera
8. **Read/write EEPROM data** via DATA tab — check for corrupt calibration or configuration blocks in IC0852/IC3004

### Level 3: Hardware Debug (Advanced Service)

9. **Debug UART** — connect to SoC serial console for kernel-level diagnostics
10. **FW UP interface** — load diagnostic firmware if available from Sony engineering

---

## 6. Answer to the Core Question

**Can a Sony service technician test Wi-Fi without replacing hardware?**

**Yes, partially.** The documented procedures allow a technician to:

| Test | Confirms | Tool Required |
|------|----------|--------------|
| Wi-Fi functional check (§1-7-4) | Wi-Fi radio transmits SSID | None |
| Factory Reset | Config is not corrupt | None |
| Firmware reinstall | Firmware is not corrupt | SD card + firmware file |
| Log Dump analysis | What errors the subsystem reports | None (export to media) |
| Adjust Station MAC read | EEPROM stores valid MAC | Adjust Station (service center) |
| Adjust Station EEPROM check | Wi-Fi calibration data intact | Adjust Station (service center) |
| Debug UART | Linux-level WLAN driver status | Serial adapter + disassembly |

**What is NOT documented:**
- A dedicated "Wi-Fi diagnostic mode" or "maintenance boot"
- A way to run the Wi-Fi module's built-in self-test (BIST) from the camera menus
- Radio-frequency-level diagnostics (signal strength, antenna impedance)
- Direct communication with the Marvell 88W8897 Wi-Fi chip for register-level diagnosis

**Practical conclusion:** Levels 1-2 cover most software vs. hardware triage. If Wi-Fi fails after factory reset + firmware reinstall + Adjust Station EEPROM verification, the fault is almost certainly hardware (IC0630 Wi-Fi module, antenna cable, or connector), and board replacement becomes the correct action.

---

## 7. Components in the Wi-Fi Signal Path

For reference, the hardware components a technician would inspect:

| Component | IC/Part | Function |
|-----------|---------|----------|
| Wi-Fi/BT Module | IC0630 (Marvell 88W8897) | 802.11ac + Bluetooth combo |
| WLAN Antenna | Internal antenna assembly | RF signal path |
| Main SoC | IC6000 (CXD90045 Astra) | WLAN driver host (Linux) |
| EEPROM | IC0852 / IC3004 | MAC address + calibration storage |
| USB Hub | (between SoC and Wi-Fi module) | Data path for SDIO/USB-connected WLAN |

---

*Analysis based on: PXW-Z190V Service Manual, Sony support documentation, and community research on BIONZ X architecture.*
