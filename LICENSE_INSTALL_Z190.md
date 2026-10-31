# PXW-Z190 — Exact Option-License Process (model-specific)

What the **PXW-Z190** specifically requires, traced from the camera's own Operating Instructions (authoritative, in this repo) plus Sony's ULMS documentation. Companion to [LICENSING_PROCESS.md](LICENSING_PROCESS.md) (general Sony flow) and [LICENSE_MECHANISM.md](LICENSE_MECHANISM.md) (firmware).

Tags: **[MANUAL]** = verified in `PXW-Z190_V420_manual_E.pdf` (page cited) · **[SONY]** = Sony ULMS/support · **[3P]** = reputable third-party · **[INF]** = inference.

---

## 0. Headline finding

**The PXW-Z190 Operating Instructions documents how to *verify* a license but not how to *install* one.** Across all 171 pages there are **0** occurrences of "Device ID", "installation key", or any license-install step. The camera manual only provides a **status check**; the install procedure lives in a **separate CBKZ Installation Manual** delivered with the ULMS download, which is not publicly indexed. This is the key documentation-quality result for this model.

---

## 1. What the camera itself provides **[MANUAL]**

### 1.1 The service media slot
The Z190's service/file slot is the **UTILITY SD/MS slot** (front, item 26, p9) — *"Used for proxy recording and storing/loading settings (File function)"*. It is the slot used for every file-based service operation the manual documents:
- **Firmware "Version Up"** (the `.dat` in this repo) — p125 / Update manual.
- User / ALL / Scene file **Load/Save Utility SD/MS** — pp112–113, 126–128.
- **Network Public Key → Key Export** to Utility SD/MS — p113.
- **RTMP/RTMPS → Load Utility SD/MS** (certificates / stream config) — pp67–68, 120.

So "the camera's service SD-card slot" in the original description **= the UTILITY SD/MS slot**. ✅ (terminology: "UTILITY", not "service").

### 1.2 License **verification** — the only license UI in the manual
`MENU → Maintenance → Option → Install Status` (p125):

```
Option  — "Upgrade license (option) check."
  Install Status
    MPEG HD   : Installed / Not Installed   ← CBKZ-SLMP
    Network   : Installed / Not Installed   ← CBKZ-SLNW1
```

Adjacent items: **Version → Version Number** (camcorder software version) and the **Version Up** path for firmware. There is **no** "License Option", "Install License", "Device ID", or key-entry item anywhere in the Maintenance menu or the rest of the manual. *(Contrast VENICE, which has `Maintenance → License Options` showing the Device ID and an on-camera `Execute` install — the Z190 manual exposes neither.)* **[MANUAL] + [3P]**

### 1.3 The two gated licenses (confirmed throughout the manual)
| SKU | In-camera name | What it ungates (manual pages) |
|-----|----------------|-------------------------------|
| **CBKZ-SLNW1** | *NETWORK upgrade license* | Wired/Wireless LAN, Modem, streaming (NCM / RTMP/RTMPS / MPEG-TS), FTP file transfer, planning metadata, Content Browser Mobile, network status UI — pp12–20, 27, 41, 48, 51–72, 104, 111, 124, 140 |
| **CBKZ-SLMP** | *MPEG HD upgrade license* | MPEG HD422/HD420 codecs, 4K & HD (Sub) simultaneous rec, related file-system/codec menu items — pp13, 31, 42–45, 50, 85, 121, 135–136, 140, 163–164 |

These match the two SKUs in the repo `README.md`, and the V2.00 firmware notice states the update **added support for both** licenses. **[MANUAL] + [SONY]**

### 1.4 Collateral facts
- **OS:** the unit runs **Linux (GPL/LGPL)**; OSS license texts sit in an internal mass-storage `PMHOME/LICENSE` folder (p168). Corroborates the companion doc's inference that the encrypted `FDAT` is a Linux image. **[MANUAL]**
- **Firmware-first ordering:** `CBKZ-SLNW1` requires firmware ≥ **Ver.2.00** (later network features track Ver.3.10 / 4.00). Update the firmware **before** the license. **[SONY]**

---

## 2. How the license is actually issued & installed (the part outside the camera manual)

The camera manual stops at verification, so the install itself follows Sony's standard **ULMS** flow. For the Z190 the steps are:

1. **Update firmware to ≥ V2.00** via the UTILITY slot (Version Up). **[SONY/MANUAL]**
2. **Purchase the SKU** (CBKZ-SLNW1 / CBKZ-SLMP) → receive a **12-digit Purchase Code** in a *Software Purchase Key Notification*. **[SONY]**
3. **On `ulms.sony.net`:** enter the Purchase Code → confirm license → register → **enter the camera's Unique Device ID** (bound permanently; wrong ID ⇒ contact Sony). **[SONY]**
4. **Download the Installation Key** issued by Sony's backend, plus its **Installation Manual** (the Z190-specific step-by-step lives here). **[SONY]**
5. **Install onto the camera** per that manual, then confirm under `Maintenance → Option → Install Status` → *Installed*. **[SONY/MANUAL]**

### 2.1 Which transport for step 5? (not fully public — two candidates)
Sony ships two standard install transports; the Z190's own manual documents **neither** a license-file SD path nor a Device-ID screen, which is itself the evidence:

- **(A) USB Upgrade Tool — most likely [INF].** Sony's `UpgradeTool.msi` (downloaded from ULMS) connects to the device **over USB** (*"Only one Device can be connected at a time"*) and applies the ULMS-issued key. The Z190 has USB and the manual exposes **no** on-camera SD license-install item or Device-ID display — consistent with the ID being read and the key applied **by the PC tool over USB**, not through the camera menu. **[SONY-FAQ + INF]**
- **(B) SD file via UTILITY slot — the family alternative.** Other Sony bodies (e.g. VENICE) copy the downloaded key file to the **SD card root** and run an on-camera `Execute`. Possible on the Z190, but **not reflected by any menu item in its manual**, which argues against it being the primary path. **[3P]**

**Cannot be confirmed to one from public sources** — the CBKZ-SLNW1 **Installation Manual** bundled with the ULMS download is authoritative and is not indexed publicly. **[gap]**

---

## 3. Reconciling the original description

> *"Sony ships a physical disk; an `.exe` takes the camera serial and generates a file; the file goes on an SD card into the service slot."*

| Claim | Z190 reality | Verdict |
|-------|--------------|---------|
| Physical disk | Electronic download from **ULMS** (web). No disc. | ❌ |
| An **`.exe`** on the PC | **Plausibly correct for the Z190** — Sony's `UpgradeTool.msi` (candidate A) is a PC tool over USB. But it's a **download**, and it applies a **server-issued** key, not an offline generator. | ⚠️ closer to right here than for SD-based models |
| Takes the **serial**, **generates** a key offline | You submit the **Unique Device ID** to **ULMS**, which **issues** the key **server-side**. The tool/serial doesn't mint it locally. | ❌ (mechanism), ⚠️ (an ID is submitted) |
| File onto **SD → service slot** | True only if transport **(B)**; the Z190's documented path appears to be **USB tool (A)**, in which case no SD file is involved for the license. | ⚠️ unconfirmed for this model |
| "service slot" | Exists as the **UTILITY SD/MS slot** (used for firmware + settings), regardless. | ✅ (naming) |

**Net for the Z190:** the "run a PC tool over USB" half of the description is *plausibly accurate for this specific model* (unlike the SD-card VENICE pattern), but the "physical disk" and "offline serial→key generation" parts are wrong — issuance is Sony's ULMS web backend keyed on the Unique Device ID.

---

## 4. Is the Z190 process documented properly?

**No — for this model the install procedure is absent from the primary manual.**
- The camera **Operating Instructions** exhaustively documents **feature gating** and provides a **verification** screen (`Maintenance → Option → Install Status`), but contains **no** install steps, **no** Device-ID location, and **no** installation-key handling. **[MANUAL, verified: 0 hits]**
- The **install procedure is externalized** to the per-license **CBKZ Installation Manual** (delivered with the ULMS download) and the **ULMS Getting Started / FAQ** — none of which are easy to reach from the camera or from general search.
- Practical consequence for an owner: you can confirm *whether* a license is installed from the camera, but you must go to ULMS (and its bundled install manual) to learn *how* to install it — and reconcile the **Unique Device ID vs. serial** and **firmware-first** gotchas that no single document foregrounds.

**Open item to close with Sony:** obtain the CBKZ-SLNW1/CBKZ-SLMP **Installation Manual** PDF (ULMS download or Sony Pro support) to confirm transport **(A) vs (B)** and the exact Device-ID menu location for the Z190.

---

## 5. Sources
- `PXW-Z190_V420_manual_E.pdf` (this repo) — pp 9, 12–20, 27, 31, 41–72, 85, 104, 110–113, 120–121, 124–125, 135–136, 140, 163–164, 168 **[MANUAL]**
- Sony **ULMS FAQ** — https://ulms.sony.net/faq/faq_en.html (Installation Key, Unique Device ID, `UpgradeTool.msi` over USB, non-transferable) **[SONY]**
- Sony **ULMS Getting Started** — https://ulms.sony.net/document/ULMS_GettingStarted.pdf **[SONY]**
- Sony **CBKZ-SLNW1** product page + V2.00/V3.10/V4.00 firmware notices (license support added in V2.00; firmware-first) **[SONY]**
- **AbelCine**, *Installing Optional Licenses on the Sony VENICE* — https://www.abelcine.com/articles/blog-and-knowledge/tutorials-and-guides/installing-optional-licenses-on-the-sony-venice (family SD-card pattern for contrast) **[3P]**
