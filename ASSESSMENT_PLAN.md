# PXW-Z190 — License & Secure-Update Assessment Plan

Companion to [LICENSE_MECHANISM.md](LICENSE_MECHANISM.md). This is a **defensive design-review framework** for evaluating the robustness of the firmware-update and option-license trust chain on a legacy platform. It is scoped to *what properties to verify and what "good" looks like* — the assessment questions and hardening targets a report should contain. It intentionally does **not** provide a key-extraction or license-forgery procedure.

---

## 1. Assets & trust boundaries

| Asset | Where it lives | Why an attacker wants it |
|-------|----------------|--------------------------|
| Firmware payload key (AES/ECB, §2 of companion) | On-device key store / boot ROM **[UNK]** | Decrypts/forges firmware images |
| Option-license records (CBKZ-SLNW1, CBKZ-SLMP) | NVRAM/flash, separate from firmware | Unlock paid features; clone across bodies |
| Device ID / serial | Readable from menu; bound into license | The identity the license is tied to |
| Root of trust (does secure boot exist?) | Boot ROM / first-stage loader **[UNK]** | Determines whether any of the above actually matters |

**Trust boundaries to map:** (a) PC → SD card → updater; (b) updater → encrypted payload; (c) license service → license file → NVRAM store; (d) boot ROM → each boot stage. The companion doc establishes (a) and (b) at the container level; (c) and (d) are the open questions below.

---

## 2. Security properties to verify (the assessment checklist)

For each, the question is binary-ish: *is this property actually enforced, or only assumed?*

### 2.1 Secure update chain
- **P1 — Signature, not just CRC.** Companion §1.2 showed `DEND` is a CRC-32. Does *any* stage (updater or boot ROM) verify an **asymmetric signature** over the image, or is the CRC the only integrity gate? If CRC-only, integrity reduces to "did the payload decrypt," i.e. to key secrecy.
- **P2 — Rollback protection.** Will the device accept an older signed/valid image (downgrade to a version with known defects)? Is there a monotonic version/anti-rollback counter?
- **P3 — Key scope.** Is the payload key **unique per device**, per **model/batch**, or **global**? A single global key is a single point of total failure. (The container ships no key, so this is answered only on-device.)
- **P4 — Post-decrypt re-validation.** Companion §1.1 noted `UDID` (model whitelist) is cleartext in the container. Does the updater re-check model ID and any signature **after** decryption, or trust the cleartext chunk?

### 2.2 Option-license store
- **P5 — License authenticity.** Is each license record **signed** by the issuing service (asymmetric), or merely a structured blob compared for well-formedness? "Compared, not verified" is the classic break.
- **P6 — Device binding strength.** Is the Device-ID binding part of the **signed** payload, or a field checked independently of the signature? If the binding isn't under signature, it can be edited without breaking validation.
- **P7 — Store integrity & anti-clone.** Is the NVRAM license region integrity-protected (MAC/signature) and **bound to immutable per-device identity** so a record copied from body A to body B is rejected? Replay/clone resistance is the core anti-piracy property here.
- **P8 — Flag vs. entitlement coupling.** Companion §4 established features are shipped-but-flagged. Are the runtime enable-flags derived **only** from a verified license record each boot, or cached/derived from mutable state that can be set independently of a valid license?

### 2.3 Debug & recovery surface
- **P9 — Debug authentication.** Are hardware debug/console interfaces disabled or **authenticated** on production units, or open? (Open debug on production is the usual entry point and the usual finding.)
- **P10 — Fault/glitch resistance of the verify step.** Is the signature/decryption decision a single branch (glitchable) or defense-in-depth? *Assessment observes resistance qualitatively; it does not require a working bypass to report the risk.*

---

## 3. Evidence sources (non-destructive first)

1. **This update artifact** — already mined (container format, ECB, CRC trailer, model table). Exhausted at file level.
2. **Menu/UI enumeration** — License menu, Version/Device-ID screens: capture the Device-ID format and any license-status display. Pure observation, no teardown.
3. **Cross-version container diffing** — obtain other published PXW-Z190 firmware versions and diff the *ciphertext*. Because the payload is ECB over an uncompressed image (companion §2), unchanged regions remain identical ciphertext — this reveals which regions change between versions **without any key**, and is a legitimate, file-only technique.
4. **Vendor documentation / service-manual review** — the authoritative source for the license-issuance protocol and Device-ID binding; prefer this over invasive work where it answers the question.
5. **On-device evidence** — only where 1–4 leave a property (P1, P3, P5–P10) unresolvable from artifacts alone. This is standard hardware-assessment territory (debug-port/boot-chain/flash analysis on units you own, under the engagement's authorization). Keep it to *establishing whether the property holds*, documented as findings — not to producing a persistent unlock.

Sequence the work non-destructive → on-device, and stop at the first evidence that answers each P-item.

---

## 4. What "good" looks like (hardening targets to recommend)

- **Update:** authenticated encryption (AES-GCM/XTS) **plus** an asymmetric signature over the whole container; per-image nonce; anti-rollback counter; model + signature re-checked post-decrypt.
- **Keys:** per-device or at least per-batch keys from a hardware-protected store; no global key; debug/JTAG disabled or authenticated in production.
- **License:** records signed by the issuer; Device-ID binding **inside** the signed payload; NVRAM store MAC'd and bound to immutable per-device identity so records don't survive cloning; enable-flags recomputed from a verified record every boot.
- **General:** move away from encryption-as-authentication (companion §S3) — confidentiality and authenticity should be independent so a key leak doesn't forfeit integrity.

---

## 5. Deliverable structure (for the engagement report)

1. Scope, units tested, authorization reference.
2. Trust-chain diagram (§1).
3. Findings table keyed to P1–P10: property, evidence, verdict (enforced / assumed / broken), severity.
4. Artifact-level findings already established (companion §6, S1–S5).
5. Hardening roadmap (§4), prioritized by severity × effort.
6. Explicit residual-risk / out-of-scope statement.

---

## 6. Scope & ethics note

This plan evaluates **whether** the platform's protections hold and **how to strengthen them**. It is deliberately framed at the level of security properties and methodology. It does not include, and this engagement did not produce, a procedure to extract the firmware key, forge or clone a license, or otherwise unlock entitlements without authorization. Any on-device work assumes units the assessor owns or is authorized to test under the engagement.
