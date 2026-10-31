# PXW-Z190 Firmware Decryption Analysis

**Objective:** Determine if the PXW-Z190 firmware can be decrypted using the publicly available fwtool.py tool and its known keys.

---

## Executive Summary

The PXW-Z190 firmware is **partially decryptable**. The FDAT header (first 1024 encrypted bytes) decrypts successfully with the publicly known `key_aes` using AES-128-ECB, revealing firmware metadata (version, model ID, file system layout). However, the firmware body (the actual code and filesystem data, ~264-295 MB) uses a **second encryption layer with an unknown key**, likely AES-CBC, that is not present in fwtool.py's public key database.

**Bottom line:** The FDAT metadata is readable; the firmware contents are not.

---

## 1. UFU Container Structure

All three firmware versions share the same UFU container format:

| Field | Value |
|-------|-------|
| Magic | `89 55 46 55 0D 0A 1A 0A` (PNG-style) |
| Chunks | DATV, PROV, UDID, FDAT, DEND |
| CRC | CRC-32 in DEND chunk (integrity, not a signature) |

### USB Device Identifiers (UDID Chunk)

All versions share identical USB IDs:

| VID | PID | Mode | Description |
|-----|-----|------|-------------|
| 0x054C | 0x0448 | Normal | Sony mass storage |
| 0x054C | 0x047D | Normal | Sony MTP |
| 0x054C | 0x0C41 | Normal | Professional camcorder |
| 0x054C | 0x0C42 | Normal | Professional camcorder |
| 0x054C | 0x03E2 | Updater | Firmware update mode |

PID `0x03E2` is the updater-mode USB identity, used when the camera enters firmware update state.

---

## 2. Encryption Scheme

### 2.1 Proven: Block 1 is AES-128-ECB with `key_aes`

The first 1024-byte encrypted block uses AES-128-ECB with the publicly known `key_aes` (the same key used by CXD4132 consumer cameras):

```
key_aes = E3 B0 C4 42 98 FC 1C 14 9A FB F4 C8 99 6F B9 24
```

**Evidence (three independent confirmations):**

1. **Known plaintext — UDTRFIRM magic:** Decryption produces `UDTRFIRM` at the expected offset, the standard Sony FDAT header magic.

2. **Known plaintext — version/mode fields:** The 16-byte ciphertext block `448c526ad9bea7609e94e1ddc03690ae` appears at the same position in ALL three firmware versions. Encrypting the known FDAT plaintext `30313030 55000000 4e000000 00000000` (`0100` version + `U` mode + `N` luw flag) with `key_aes` produces this exact ciphertext.

3. **Known plaintext — zero padding:** 26 copies of the ciphertext `71c6a0e61174f1eb8b7855470393bb56` appear in block 1, corresponding to the 26 unused (zero-filled) file system header slots. `AES-ECB(key_aes, 0x00*16)` produces exactly this ciphertext.

### 2.2 Block 1 Structure

The decrypted first block contains a 4-byte `BlockCrypter` header followed by the FDAT header:

```
Offset 0-1: Checksum (0x6638 for V4.21) — non-standard algorithm, does not match fwtool's sum-of-16bit
Offset 2-3: Size/Flag = 0x03FC → size=1020, end_flag=0
Offset 4+:  FDAT header (UDTRFIRM...)
```

**Why fwtool.py fails:** The checksum at offset 0-1 does not match fwtool's `_calcSum` algorithm (sum of 16-bit little-endian words). The Z190 uses a different checksum scheme, causing `BlockCryptException: Wrong checksum` for every key tried.

### 2.3 Not Decryptable: Body uses unknown key

Blocks 2+ (everything after the first 1024 bytes) exhibit:

- **256/256 byte value diversity** — maximum entropy, consistent with encryption or strong compression
- **Zero repeated 16-byte blocks** in >1 million sampled blocks — rules out AES-ECB entirely
- **No encrypted-zero patterns** (`71c6a0e6...`) found anywhere in the body
- **Per-version unique trailer** — last 0x110 bytes differ across versions, consistent with a per-firmware IV for AES-CBC

This matches the `AesCbcCrypter` architecture used by CXD90045 consumer cameras:
- Block 1 header area: AES-ECB with `key_aes`
- Block 1 body + all subsequent blocks: AES-CBC with a **different, per-SoC key** and an IV stored in the file trailer

The Z190's CBC key is **not** any of the following known keys:
- `key_cxd90045` (32 bytes, AES-256)
- `key_cxd90014` (16 bytes, AES-128)
- Any of the SHA-based keys (CXD4105, CXD4115, CXD4120 families)

---

## 3. Decrypted FDAT Metadata

From the ECB-decrypted first block across all three versions:

| Field | V1.01 | V2.00 | V4.21 |
|-------|-------|-------|-------|
| Magic | UDTRFIRM | UDTRFIRM | UDTRFIRM |
| Version Tag | 0100 | 0100 | 0100 |
| Mode Type | U (User) | U (User) | U (User) |
| LUW Flag | N (Normal) | N (Normal) | N (Normal) |
| FW Version | 1.01 | 2.00 | 4.21 |
| Model ID | 0x81040012 | 0x81040012 | 0x81040012 |
| Region | 0x00000000 | 0x00000000 | 0x00000000 |
| FW Offset | 201,216 | 201,216 | 201,216 |
| FW Size | 275,573,248 (262.8 MB) | 279,382,528 (266.5 MB) | 308,159,488 (293.9 MB) |
| Num FS | 2 | 2 | 2 |
| FS[0] | U, offset=512, size=200,704 | U, offset=512, size=200,704 | U, offset=512, size=200,704 |
| FS[1] | P, offset=512, size=0 | P, offset=512, size=0 | P, offset=512, size=0 |
| Payload Size | 276,857,104 (264.0 MB) | 280,680,720 (267.7 MB) | 309,570,832 (295.2 MB) |
| Trailer IV | 31e76abf...1a6de024 | 9f94a594...87980533 | 76ebdf3d...9a152657 |

### Key Observations

- **Model 0x81040012:** This is the Z190's internal model identifier in Sony's numbering scheme. The `0x81` prefix likely indicates the professional camcorder product line (consumer ILCEs use `0x7103xxxx`, `0x8103xxxx`, `0x9103xxxx`).
- **Firmware growth:** 262.8 MB → 266.5 MB → 293.9 MB across three major versions, suggesting significant feature additions over time.
- **Consistent FS layout:** The updater filesystem (FS[0]) is exactly 200,704 bytes (196 KB) across all versions, with a fixed offset of 512 bytes. The production filesystem (FS[1]) has size=0, suggesting the production image is embedded in the firmware.tar, not a separate partition.
- **Unique IVs:** Each firmware version has a distinct 16-byte trailer IV, confirming CBC mode for the body with per-build initialization vectors.

---

## 4. Devices.yml Database Gap

fwtool.py's `devices.yml` contains 502 device entries covering 143 total devices across 8 SoC architectures. **No professional camcorder models** are listed:

| Architecture | Devices | Product Lines |
|-------------|---------|---------------|
| CXD90045 | 16 | Consumer only (ILCE, DSC, FDR, ZV) |
| CXD90014 | 66 | Consumer only |
| CXD4132 | 42 | Consumer only |
| Others | 19 | Consumer only |

The Z190 (model 0x81040012) is absent. The fwtool.py project is focused exclusively on consumer camera bodies (Alpha, Cyber-shot, Handycam), not the professional XDCAM line.

---

## 5. What IS Recoverable Without the CBC Key

Despite the body remaining encrypted, the FDAT header provides:

1. **Firmware version identification** — exact version number from any .dat file
2. **Model verification** — confirm firmware is for model 0x81040012 (Z190)
3. **Size analysis** — track firmware size growth across versions
4. **Update structure** — FS layout, firmware offsets, updater size
5. **USB identity mapping** — PIDs for normal/updater modes
6. **ECB weakness demonstration** — the repeated ciphertext blocks (all-zero FS headers, version/mode fields) confirm the key and reveal structural information

---

## 6. Security Assessment

| Finding | Severity | Detail |
|---------|----------|--------|
| S1: ECB mode on header | Medium | Header metadata leaked via ciphertext patterns. The repeated `448c526a...` block across all versions reveals the FDAT structure. |
| S2: Shared key with consumer line | Medium | The same `key_aes` encrypts the header for both professional (Z190) and consumer (ILCE-7M3, etc.) cameras. Compromise of this key (which is already public) exposes metadata for all models. |
| S3: Body encryption with unique key | Strength | The firmware body uses a separate, non-public key. This limits exposure — even with the leaked header key, the actual code remains protected. |
| S4: Per-build IV | Strength | Unique CBC initialization vectors per firmware version prevent cross-version ciphertext comparison attacks on the body. |
| S5: No digital signature | High | The DEND chunk contains only a CRC-32 checksum, not a cryptographic signature. If the CBC key were recovered, firmware could be modified and repacked with a valid CRC. |
| S6: Modified BlockCrypter checksum | Low | The Z190 uses a non-standard checksum in the block header, which prevents fwtool.py from decrypting even with the correct key. This is security-by-obscurity, not a real protection — once the algorithm is identified, it can be reimplemented. |

---

## 7. Tools and Reproduction

### Decrypt FDAT Header (Block 1 Only)

```python
from Cryptodome.Cipher import AES
key_aes = bytes.fromhex('E3B0C44298FC1C149AFBF4C8996FB924')
cipher = AES.new(key_aes, AES.MODE_ECB)

with open('FirmwareData.dat', 'rb') as f:
    # Skip UFU container parsing, extract FDAT chunk
    # (use fwtool.sony.dat.readDat for proper extraction)
    pass

# Decrypt first 1024-byte block
decrypted_block1 = cipher.decrypt(encrypted_block1)
# FDAT header starts at offset 4 (after 4-byte block header)
fdat_header = decrypted_block1[4:]
```

### Verify with Known Plaintext

```python
# Encrypt known plaintext to verify key
assert cipher.encrypt(b'\x00' * 16).hex() == '71c6a0e61174f1eb8b7855470393bb56'
assert cipher.encrypt(bytes.fromhex('30313030550000004e00000000000000')).hex() == '448c526ad9bea7609e94e1ddc03690ae'
```

---

*Analysis performed on firmware versions V1.01, V2.00, and V4.21 using fwtool.py (ma1co/nex-hack) and PyCryptodome.*
