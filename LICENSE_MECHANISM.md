# PXW-Z190 — Firmware Update & Option-License Mechanism

Reverse-engineering notes for security assessment of a legacy Sony camcorder platform.

- **Model:** Sony PXW-Z190 (-V)
- **Artifact:** `PXW-Z190_V421_E/FirmwareData_PXWZ190V421.dat`
- **Size:** 309,570,936 bytes
- **MD5:** `f7cd1da30867e3d7824011eb68e33f41` (matches shipped `.md5`; Sony's Readme states the md5 is a download-integrity check only, *"not used for updating"*)
- **Host/tooling note:** analysis done on macOS. REA's `extract_firmware`/`inspect_firmware_regions` need Linux + binwalk/unblob, so the container below was parsed directly from bytes rather than via the extractor.

Every claim is tagged: **[OBS]** observed in the artifact, **[INF]** inferred from observation, **[PUB]** publicly documented Sony behavior for this product family, **[UNK]** not determinable from this artifact alone.

---

## 1. Firmware container format — "UFU" **[OBS]**

The `.dat` is a Sony **UFU** container: a PNG-style magic followed by length-prefixed typed chunks. There is **no per-chunk CRC** (unlike PNG); chunks are simply `[len:4 BE][type:4][body:len]`.

```
magic: 89 55 46 55 0D 0A 1A 0A      "‰UFU\r\n\x1a\n"
```

| off | type | len | body | meaning **[INF]** |
|-----|------|-----|------|-------------------|
| `0x00000008` | `DATV` | 4 | `01 00 00 00` | container/data format version = 1 |
| `0x00000014` | `PROV` | 4 | `01 00 00 00` | provisioning/provider version = 1 |
| `0x00000020` | `UDID` | 44 | model-ID table (below) | list of device/board IDs this image may install onto |
| `0x00000054` | `FDAT` | 309,570,832 | encrypted payload | the actual firmware image |
| `0x1273AD6C` | `DEND` | 4 | `67 1F 26 CF` | end marker = CRC-32 of the whole file |

### 1.1 `UDID` — model binding **[OBS/INF]**

```
00000005 | 0448 054C 01000000 | 047D 054C 01000000 | 0C41 054C 01000000 | 0C42 054C 01000000 | 03E2 054C 02000000
  count=5   ──rec1──            ──rec2──             ──rec3──             ──rec4──             ──rec5──
```

- 4-byte count (`5`) followed by five 8-byte records.
- Each record = `[model-id:2][platform/region const:2 = 0x054C][variant/rev:4 LE]`.
- Varying model IDs: `0448, 047D, 0C41, 0C42, 03E2`; constant `0x054C`; trailing value `1,1,1,1,2`.
- **[INF]** This is the **update-eligibility whitelist**: the bootloader/updater compares the camera's own model/board ID against this table and refuses an image whose ID is absent. It is a *model gate*, not a license gate — it stops you flashing Z190 firmware onto a different body, and vice-versa.

### 1.2 `DEND` — integrity, *not* authenticity **[OBS]**

`DEND = 0x671F26CF = CRC32(file[0x0 .. 0x1273AD6C])` (verified: `crc32` of everything before the DEND chunk reproduces the value exactly).

**Security finding:** the container's only built-in integrity field is a **plain CRC-32**. There is no asymmetric signature over the container. Authenticity therefore depends entirely on the confidentiality of the `FDAT` encryption key (see §2) — i.e. *encryption is being used as authentication*. If the payload key is ever recovered, nothing else in the container cryptographically binds the image to Sony.

---

## 2. Payload (`FDAT`) cryptanalysis **[OBS/INF]**

- **Entropy:** uniform **H ≈ 8.000 bits/byte** across the entire 309 MB payload → encrypted (or encrypted-after-compression). No plaintext, filesystem magic, or string islands anywhere in `FDAT` (confirmed by a full keyword sweep — the only "hits" were random mixed-case coincidences, one each).
- **Mode = ECB (block cipher, 16-byte block):** one 16-byte-aligned block (`71C6A0E6 1174F1EB 8B785547 0393BB56`) repeats **26×** within the first 4 MB while 262,119 of 262,144 blocks are unique. Identical ciphertext blocks at 16-byte alignment only arise when identical *plaintext* blocks encrypt identically → **ECB** with a **128-bit block** (AES-class). **[INF]**
- **[INF]** Because fixed-fill regions (erased flash / padding) survive as repeated ciphertext, the payload is an **uncompressed raw image encrypted directly** (compression would have destroyed the repeats). So `FDAT` is most likely a full NAND/partition image under ECB.

**Security finding (design-level):** ECB over a raw image leaks structure — region boundaries, fill/erased areas, and duplicated blocks (code pages, repeated tables) are all visible in the ciphertext, and identical plaintext is detectable across firmware versions. This is the classic "ECB penguin" weakness applied to a firmware image. A stream/CBC/XTS mode with a per-image IV, plus a real signature, would be the expected modern design.

- **Key:** **[UNK]** not present in this artifact. For this device class the payload key is held by the SoC bootloader / fused key store on the camera, not shipped in the update. Recovering it is a hardware/boot-chain exercise, out of scope here.

---

## 3. Firmware update ("Version Up") process **[PUB — from the bundled manuals]**

From `PXW-Z190_V420_Update_manual_E.pdf` and `..._manual_E.pdf`:

1. **Prepare UTILITY media** — format an SD/SDHC/SDXC card *in the camera* (the UTILITY SD/MS slot). Wrong formatting → update won't start.
2. **Copy** `FirmwareData_PXWZ190V421.dat` to the top level of the card on a PC/Mac.
3. On the camera: use the supplied **AC adaptor**; disconnect USB/SDI/HDMI/mic/shoe accessories and eject other media. *(The manual flags an extra pre-step "set these settings to Off — when the **NETWORK upgrade license (option)** is installed," i.e. the licensed network features must be quiesced before flashing.)*
4. Insert UTILITY media, power on, **Menu → Version Up → Execute**.
5. Camera displays `Version Upgrade VX.XX > VY.YY`, runs in phases, then `Version Upgrade Done`.
6. Error/branch strings observed in the manual (useful as updater state markers):
   - `Version Upgrade No File` — image not found on card.
   - `Version Upgrade NG: Cannot Read. Power Off` — read/verify failure mid-update.
   - `Version Upgrade Error!!` — update failed; redo from start.

**[INF]** The updater's acceptance checks, in order, are: (a) file present & UFU magic, (b) CRC-32 `DEND` valid, (c) model ID ∈ `UDID` table, (d) payload decrypts under the device key. Only (a)–(c) are verifiable from this artifact; (d) is inferred from the payload being encrypted.

---

## 4. Option-license model **[PUB + repo context]**

The PXW-Z190 ships as one hardware body; several capabilities are sold as **CBKZ option licenses**. From the repo's `README.md`, the two licenses in scope:

| License SKU | Capability | Notes |
|-------------|-----------|-------|
| **CBKZ-SLNW1** | Network / wireless suite for PXW-Z190 | Matches the manual's **"NETWORK upgrade license (option)"** gate — streaming, FTP/file-transfer, wireless-LAN control, etc. |
| **CBKZ-SLMP** | **MPEG HD** upgrade | Unlocks the MPEG HD (XDCAM-style) recording/codec path. |

### 4.1 How a CBKZ license is installed and activated **[PUB]**

For this generation of Sony professional camcorders the option-license flow is:

1. **Purchase** the SKU → you receive an **Installation Key** / voucher.
2. **Bind to device:** on Sony's license-issuing service you supply the camera's **Device ID / serial** (readable from the camera's license/version menu). The service returns a **license file cryptographically bound to that specific Device ID** — a license for one body does not work on another.
3. **Install:** load the license file from SD via the camera's **License** menu (some models also accept direct online activation). The camera validates the file and writes a **persistent license record into internal NVRAM/flash** (separate from the firmware image).
4. **Activate:** on boot the firmware reads the installed-license store and **sets per-feature enable flags**. The already-present code paths (network stack, MPEG HD codec) are then exposed in the menus.

**Key point for the assessment:** the *code* for the optional features is already in the firmware image on every unit. The license does not *add* software — it **flips enable flags** read from the NVRAM license store. The license file is **device-bound** (tied to Device ID), which is the primary anti-transfer control.

---

## 5. "Blocked"/gated software in the firmware **[INF/PUB]**

Based on §4 and the licenses in scope, the firmware carries — present but disabled until licensed:

- **Network/streaming subsystem** (gated by **CBKZ-SLNW1**): wireless LAN control, streaming, FTP/file transfer, remote/Web control. The update manual's conditional "*when the NETWORK upgrade license is installed*" step is direct evidence this subsystem is shipped-but-gated. **[OBS, indirect]**
- **MPEG HD codec path** (gated by **CBKZ-SLMP**): the alternative HD recording format. **[PUB]**

**What cannot be enumerated from this artifact, and why [UNK]:** the exact feature-flag symbols, the license-store layout in NVRAM, the Device-ID binding algorithm, and the license-file signature scheme all live **inside the ECB-encrypted `FDAT`**. Reading them requires the payload key (held on-device), so static enumeration of the gating flags is **not possible from the update file alone**. Going further needs either (a) the on-device decryption key via the boot chain, or (b) a dump of a running unit's flash — both hardware exercises outside this file-level analysis.

---

## 6. Security assessment summary (for the engagement)

| # | Observation | Evidence | Risk angle |
|---|-------------|----------|------------|
| S1 | Update container authenticated only by **CRC-32**, no digital signature | `DEND` = crc32 of file body, verified | If payload key leaks, arbitrary images pass integrity. Recommend verifying whether the *bootloader* adds a signature check the container itself lacks. |
| S2 | Payload encrypted in **ECB** over an uncompressed image | 26× repeated aligned 16-byte block; H=8.0 | Structural leakage; cross-version diffing of identical plaintext blocks; not IND-CPA. Recommend authenticated encryption (e.g. AES-XTS/GCM + signature). |
| S3 | **Encryption used as authentication** | No signature + encrypted payload | Confidentiality of one key = the whole trust anchor. Single point of failure. |
| S4 | Model gate is a **plaintext whitelist** (`UDID`) | parsed table | Trivially readable/editable in the container; only meaningful because the payload is encrypted. Worth checking the updater re-validates model ID *after* decryption, not just from the cleartext chunk. |
| S5 | Option features are **shipped-but-flagged**, unlocked by a **device-bound NVRAM license record** | §4 (public) + manual gate | Assess: license-file signature strength, Device-ID binding (is it signed, or just compared?), and whether the NVRAM store is integrity-protected against tampering/replay/cloning between bodies. |

### Recommended next steps (hardware-level, beyond this file)
- Recover the boot/decryption chain from a unit (UART/JTAG/boot ROM) to confirm S1/S2 and to obtain the plaintext image for flag-level analysis.
- Dump NVRAM from a licensed vs. unlicensed body to characterize the license record format and its integrity protection (the real target for an anti-cloning review).
- Confirm whether model-ID and any signature are re-checked post-decrypt inside the updater.

---

## 7. Scope note

This document explains **how the licensing and update mechanism works** and where its security properties are weak, for defensive assessment of a legacy platform. It deliberately does **not** provide a method to forge a license, bypass validation, or unlock paid features without entitlement — that was neither requested nor produced. Recovering in-image feature flags or the license-binding algorithm requires on-device key material not present in this artifact.
