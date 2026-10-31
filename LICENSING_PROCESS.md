# PXW-Z190 — Option-License Issuance & Installation Process (validated)

Validation of the licensing workflow against public Sony documentation, and an assessment of how well that process is documented. Companion to [LICENSE_MECHANISM.md](LICENSE_MECHANISM.md) and [ASSESSMENT_PLAN.md](ASSESSMENT_PLAN.md).

Tags: **[SONY]** official Sony source · **[3P]** reputable third-party · **[INF]** inference.

---

## 1. The description under test

> *"Sony ships a physical disk which the user runs on his computer. The `.exe` takes the camera serial as input and generates a special file that is placed on an SD card, then inserted into the camera's service SD-card slot."*

**Verdict: directionally right about the SD-card install, but wrong/outdated on the front half.** The real system is a **web portal (ULMS)** that issues the license file **server-side** — there is no shipped disc, and no offline `.exe` that generates a key from the serial. Point-by-point below.

---

## 2. What the process actually is **[SONY]**

Sony licenses are issued through the **Upgrade and License Management Suite (ULMS)** at `ulms.sony.net` — a website, delivered as an **"Electronic Download"** (so listed by resellers). Flow:

1. **Buy the SKU** (e.g. `CBKZ-SLNW1`). You receive a **Purchase Code** (a 12-digit key) in a *Software Purchase Key Notification*. *"A purchase code (key) is required in order to issue an installation key for a license."*
2. **Read the Unique Device ID off the camera.** On the body, open the full menu → **Maintenance → License Options** → the **Device ID** is listed there. (This is a distinct license identifier, **not** the printed serial number — a common point of confusion.)
3. **On the ULMS site:** *Enter Purchase Code* → confirm the license → *register* → when prompted, **enter the Unique Device ID**. The ID **cannot be changed once registered**, and licenses are **tied to that specific equipment and non-transferable** (unless sold as a "Portable License").
4. **Issue & download the Installation Key.** Click *Download Installation Key(s)*. The portal generates the key **on Sony's backend** and gives you a **file to download** (a `.txt` on VENICE; name varies by model). The download page also links the per-license **Installation Manual** (PDF).
5. **Install on the camera via SD:** copy the key file to the **root** of an SD card **formatted in the camera**, insert it into the camera's **UTILITY** SD slot, then **Maintenance → License Option → Execute**. The camera detects the key and, on success, shows **"Install License OK: Turn Power Off."** Reboot; confirm under *License Options*.

**[INF]** The installed entitlement is then written to the camera's NVRAM license store (separate from firmware) and read at boot to flip the feature flags — consistent with the firmware-side findings in the companion doc (features are shipped-but-gated, not added by the license).

### 2.1 The one grain of truth about a PC tool **[SONY]**
Sony *does* offer a Windows helper — the **Upgrade Tool** (`UpgradeTool.msi`, downloaded from ULMS) — for devices that install over **USB** rather than SD. But it: (a) is a **download, not a shipped disc**; (b) **connects to the device over USB** (*"Only one Device can be connected at a time"*); and (c) still relies on the key **issued by the ULMS backend** — it does **not** generate a key offline from a serial. Portable licenses similarly use a **USB flash drive** to carry install/uninstall key files. None of this is the camcorder's SD path, which is the one that applies to the Z190.

---

## 3. Claim-by-claim validation

| User's claim | Reality **[SONY/3P]** | Verdict |
|---|---|---|
| Sony ships a **physical disk** | ULMS is a **website**; the license is an **Electronic Download**. No disc. | ❌ **Inaccurate** (likely conflated with the old optical-disc XDCAM era or with the downloadable Upgrade Tool) |
| User runs an **`.exe`** on a PC | Mostly **web-based**. An optional `UpgradeTool.msi` exists but installs over **USB** and still uses the ULMS-issued key. | ⚠️ **Partly true, conflated** |
| The exe takes the **camera serial** as input | You enter the **Unique Device ID** (from *Maintenance → License Options*) — not the serial — into the **ULMS website**, together with the **Purchase Code**. | ⚠️ **Partly right** (an ID is entered, but it's the Device ID, into the portal, not a local exe) |
| It **generates a special file** | ULMS **issues/downloads** an **Installation Key file**, generated **server-side** by Sony's licensing backend. | ⚠️ **Right that a file results; wrong that a local exe makes it** |
| File is **placed on an SD card** | ✅ Copied to the **root** of a camera-formatted SD card. | ✅ **Accurate** |
| Inserted into the **service SD slot** | ✅ Camcorders use the **UTILITY** SD slot; install via the *Maintenance / License* menu. | ✅ **Accurate** (the slot is branded "UTILITY," not "service") |

**Net:** the user's mental model matches the **device-side** install (SD card → UTILITY slot) but misdescribes the **issuance** side: it's a Sony **web portal** keyed on the **Device ID + Purchase Code**, not a shipped disc with an offline serial-to-key generator.

---

## 4. Firmware prerequisite **[SONY]**

`CBKZ-SLNW1` (the network/streaming upgrade) requires the camera to be on **firmware Ver. 2.00 or later** before the license applies; later network features (e.g. RTMP/RTMPS streaming) track still-later firmware (Ver. 4.00 / 4.10). So the correct order is **firmware update first, then license install** — which is why the update manual in this repo carries the conditional *"when the NETWORK upgrade license (option) is installed"* steps.

---

## 5. Is the process documented properly?

**Partially — accurate but fragmented across three places, with model-specific gaps.**

**Well documented [SONY]:**
- The **ULMS portal workflow** — purchase code → register → Device ID → issue/download key — is covered by the **ULMS Getting Started guide** and the **ULMS FAQ** on `ulms.sony.net`.
- Each license's **Installation Manual** (PDF) is linked from its ULMS download page and gives the device-side steps.
- The camera's own **Operating Instructions / Version-Up manual** (in this repo) document the UTILITY-slot and the network-feature gating.

**Gaps / friction points:**
- **Serial vs. Unique Device ID** is a recurring confusion and is *not* flagged prominently where buyers first look.
- The **exact installation-key file name** is not consistently public and is **model-dependent** (VENICE = a `.txt`; some Alpha/ZV bodies expect exactly `RQ_LIC.DAT` at card root — a wrong/duplicated name like `RQ_LIC (1).DAT` is rejected). The **PXW-Z190-specific** file name/slot detail lives in its install manual rather than in general search results.
- The **once-registered Device ID cannot be changed**, and a key issued against a **wrong Device ID** requires contacting Sony support — a sharp edge that is documented only in the FAQ, not at the point of entry.
- Public search surfaces the **marketing** pages far more readily than the **install manuals**, so a first-time installer can easily land on the "what it does" page and miss the "how to install" PDF.

**Recommendation for your write-up:** treat ULMS (portal) + per-license Installation Manual + camera Operating Instructions as the authoritative triad; call out the Device-ID (not serial) and the firmware-first ordering explicitly, since those are the two mistakes the current documentation layout invites.

---

## 6. Sources

- Sony **CBKZ-SLNW1** product page (network upgrade license; "Electronic Download") — pro.sony `/products/software-licences-and-updates/cbkz-slnw1` **[SONY]**
- Sony **ULMS FAQ** — https://ulms.sony.net/faq/faq_en.html (Installation Key definition, Device ID, UpgradeTool.msi/USB, Portable License/USB flash, non-transferable) **[SONY]**
- Sony **ULMS Getting Started** — https://ulms.sony.net/document/ULMS_GettingStarted.pdf (Purchase Code → issue Installation Key workflow) **[SONY]**
- Sony **Software Licences & Upgrades / ULMS** landing — pro.sony `/products/software-licences-and-updates` **[SONY]**
- **AbelCine**, *Installing Optional Licenses on the Sony VENICE* — https://www.abelcine.com/articles/blog-and-knowledge/tutorials-and-guides/installing-optional-licenses-on-the-sony-venice (device-side steps: Maintenance → License Options, Device ID, SD root, Execute, "Turn Power Off") **[3P]**
- Sony community thread (ZV-E1) on license-file rejection — exact filename `RQ_LIC.DAT`, card root, slot 1 **[3P]**
- PXW-Z190 support / CBKZ-SLNW1 firmware-dependency notes (Ver. 2.00+ for the license; Ver. 4.00/4.10 for RTMP/RTMPS) **[SONY]**

*Caveat:* the ULMS Getting Started PDF is dated 2020 and VENICE menu paths are model-specific; confirm the exact PXW-Z190 menu names and installation-key file name against the Z190's own installation manual on Sony's support site.
