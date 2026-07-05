# GrapheneOS Security Model — Reference

Technical reference explaining the mechanisms behind the NULLBYTE build documented in
[../security-model.md](../security-model.md), [../operational-profiles.md](../operational-profiles.md), and
[../network-security.md](../network-security.md). Reference/research document.

> **Naming caveat to verify.** The build notes refer to the on-device secure element as the
> **Titan M2** (the chip introduced with the Pixel 6 generation). Confirm the exact secure-element
> generation shipped in the specific Pixel 10 Pro Fold and the Tensor G5 platform before publishing
> hardware-specific claims; the *mechanisms* below hold regardless of the chip's marketing name,
> but the name should be checked.

---

## 1. Verified boot chain

GrapheneOS relies on **Android Verified Boot (AVB)** with a hardware root of trust, made possible
because Pixel bootloaders can be **re-locked against a user-supplied AVB key** (see §5 in
[grapheneos-vs-alternatives.md](grapheneos-vs-alternatives.md) for why this is Pixel-specific).

```
Hardware root of trust (secure element + SoC keystore)
   │  stores the verified-boot public key fingerprint; enforces rollback protection
   ▼
Bootloader (re-locked after flashing GrapheneOS)
   │  verifies the signature on vbmeta / the boot images against the trusted key
   ▼
Boot / init / kernel
   │  dm-verity protects the read-only system & vendor partitions
   │  every block hashed against a signed Merkle tree → on-the-fly integrity check
   ▼
system_server / framework
   ▼
Per-profile encrypted userspace (File-Based Encryption)
```

What each stage verifies:

- **Secure element / keystore:** holds the trusted AVB key fingerprint, performs key attestation,
  and enforces **rollback protection** (a signed image older than the recorded minimum is rejected,
  defeating downgrade-to-vulnerable attacks).
- **Bootloader:** checks the cryptographic signature of the boot chain. A locked bootloader with a
  modified image **refuses to boot** (or boots to a tamper warning), so offline OS modification
  (evil-maid on the OS partition) is detected.
- **dm-verity:** the system/vendor partitions are read-only and hash-verified block-by-block at
  runtime; any tampering of a read-only partition produces a verity error rather than silent
  execution of modified code.
- **Boot-state display:** at boot the device shows the verified-boot state and the key fingerprint
  (yellow = custom key / GrapheneOS), letting the owner confirm the expected root of trust.

### 1.1 Key correction on "the SSD/clone" framing

The build prompt phrases this as "cloning the SSD without the Titan M2 is insufficient." On a phone
there is no SSD — the relevant storage is the internal UFS flash. The accurate statement is below
(§2): copying the **flash contents** to other hardware does not yield decryptable data, because the
encryption keys are bound to *this device's* secure element and cannot be extracted or replayed
elsewhere.

---

## 2. Why cloning the storage is useless without the device's secure element

Android uses **File-Based Encryption (FBE)**. Each file is encrypted with a per-file key; those
keys are wrapped by class keys that are, in turn, protected by a **Key Encryption Key (KEK)** that
is **not derivable from the storage alone**:

- The KEK is derived from **(user credential) + (a hardware-bound key inside the secure element /
  StrongBox keystore)**.
- The secure element implements **Weaver**: it stores a secret tied to the user's credential and
  enforces **hardware throttling** on credential attempts. Guessing the lockscreen credential
  cannot be parallelised on copied flash, because the throttling/secret lives in the secure
  element, not on the flash.
- Keymaster/StrongBox keys can be **bound to the verified-boot state**, so keys are usable only on a
  device booted into the attested OS state.

Consequence: an attacker who images the UFS flash and moves it (or its bits) to other hardware has
**ciphertext plus a salt-like wrapper**, but not the hardware-bound KEK and not the Weaver secret.
Brute force must happen *on the original device, through its rate-limited secure element* — which is
exactly what makes a strong random credential effectively unbreakable here. This is the cryptographic
basis for the "locked profile is inaccessible" property in
[../operational-profiles.md](../operational-profiles.md).

> Precision note: it is the **hardware-bound keystore + Weaver**, not literally "the verified-boot
> key hash," that protects the FBE keys. Verified boot *attestation* can additionally bind keys to
> the boot state. State it this way to stay technically accurate.

---

## 3. Per-profile encryption architecture

NULLBYTE uses nine Android user profiles as security boundaries (the operational profile model in
[../operational-profiles.md](../operational-profiles.md)). The encryption model:

- Each Android **user profile** has its own **Credential Encrypted (CE)** storage, keyed from that
  profile's lockscreen credential combined with the hardware-bound key (§2).
- A **locked** profile has its CE keys **evicted from memory** — its data is ciphertext-at-rest even
  to the owner. GrapheneOS's **per-profile "End session"** control forces this eviction on demand
  rather than waiting for a reboot.
- The **owner** profile (Nexus) is special: it can create/delete profiles and holds device admin,
  and its key is the one most tightly bound to the secure element.

### 3.1 The actual cross-profile attack surface

Profiles are a strong boundary but **not** a hypervisor-grade one. What they share — and therefore
the real attack surface between them:

- **One Linux kernel.** All profiles run on the same kernel. A kernel LPE exploit reachable from a
  compromised app could, in principle, cross profiles. GrapheneOS's kernel hardening (attack-surface
  reduction, hardened allocator, exploit mitigations) raises this bar but does not make it zero.
- **One `system_server` / framework instance.** Shared system services are a cross-profile surface;
  a framework compromise is profile-agnostic.
- **The secure element and TEE** are shared; a TEE compromise would be device-wide.
- **Side channels** (timing, sensors, microarchitectural) are not partitioned by the profile model.
- **Locked-profile limit:** data in *locked* profiles is encrypted and not in memory, so a
  compromise of the *currently unlocked* profile cannot read locked profiles' data — this is the
  property that makes compartmentalisation meaningful. The window of exposure is "profiles unlocked
  at the same time."

Operational takeaway already reflected in the build: keep high-sensitivity profiles (Vault,
Nexus) locked and unlocked rarely; keep the broad-surface profile (Void, pentesting) free of
sensitive data so that even a same-session compromise yields nothing valuable.

---

## 4. Sandboxed Google Play security model

GrapheneOS runs Google Play as **sandboxed Play** rather than as a privileged system component.

| | Stock Android Play Services | GrapheneOS Sandboxed Play |
|---|---|---|
| Privilege level | System/signature-level, deeply integrated | Ordinary app, in the user's app sandbox |
| Permissions | Granted many privileged/system permissions implicitly | Only the runtime permissions the user grants, like any app |
| Cross-app/profile reach | Broad (privileged) | None beyond its own sandbox/profile |
| Persistence | System component | Lives and dies with its profile; uninstallable |
| Compatibility | Native | A compatibility layer redirects privileged calls to user-grantable equivalents |

What Google **can** still see when sandboxed Play is used:

- Data the apps that *use* Play deliberately send (e.g. an app's own telemetry, account data if you
  sign in to a Google account in that profile), and the network metadata (IP, unless tunnelled —
  see [../network-security.md](../network-security.md)).
- Anything tied to a Google account you choose to log into within that profile.

What Google **cannot** do with sandboxed Play:

- Exercise system/signature privileges, read other apps' or other profiles' data, silently persist,
  or pull device-wide identifiers it would get as a privileged system service.

Profiles that need no Google services simply **do not install Play** (e.g. Nexus; Vault unless a
specific banking app demands it), eliminating that surface entirely in those compartments.

---

## 5. Specific improvements over stock Android (with rationale)

| GrapheneOS improvement | Threat-model rationale |
|------------------------|------------------------|
| **Hardened memory allocator (`hardened_malloc`)** | Mitigates heap exploitation primitives (use-after-free, overflow) that underpin many RCE/LPE chains |
| **Attack-surface reduction** (disable/limit risky features; toggle USB, sensors, Bluetooth/Wi-Fi auto-off) | Fewer reachable interfaces = fewer exploit entry points, esp. against physical/proximity attacks |
| **Per-connection / per-network MAC randomisation; Wi-Fi/LTE-only toggles** | Reduces tracking and proximity attack surface |
| **Sandboxed Play (§4)** | Removes Google's privileged system access while retaining app compatibility |
| **Per-profile + per-app network/sensor permissions; Storage Scopes; Contact Scopes** | Least privilege; apps get scoped, not blanket, access |
| **Faster security patch cadence than typical OEMs** | Shrinks the exposure window for n-day kernel/userspace bugs |
| **Strong rollback protection + locked bootloader with custom key** | Blocks downgrade-to-vulnerable and offline OS tampering |
| **Duress PIN / panic features; auto-reboot to "before first unlock" state after a timeout** | Returns the device to a state where FBE keys are not in memory, defeating live-extraction after seizure |

GrapheneOS does **not** change the fundamentals it inherits (the kernel, the Pixel hardware root of
trust); it reduces attack surface and adds mitigations on top. Stating it as "hardening +
de-Google-ing on a strong hardware base," not "a different security architecture," is the accurate
framing.

---

## 6. Open items / to verify

- [ ] Exact secure-element generation in the Pixel 10 Pro Fold (Titan M2 vs successor) — §0 caveat.
- [ ] GrapheneOS version and security patch level in this build (also a TODO in the build notes).
- [ ] Whether auto-reboot-to-BFU timeout and per-profile End-session/auto-lock are enabled
      (cross-ref [../operational-profiles.md](../operational-profiles.md) TODOs).
- [ ] Whether duress/panic features are configured.

> No CVEs, benchmark figures, or vendor claims are asserted without verification. Mechanism
> descriptions (FBE, Weaver, AVB, dm-verity, StrongBox) reflect documented Android/GrapheneOS
> behaviour; confirm specifics against current GrapheneOS documentation before reuse in published
> material.
