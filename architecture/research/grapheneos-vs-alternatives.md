# GrapheneOS vs Alternatives — Reference

Comparison reference explaining why NULLBYTE is built on GrapheneOS / Pixel hardware rather than an
alternative ROM or stock Android. Complements
[grapheneos-security-model.md](grapheneos-security-model.md).

> Mechanism claims below reflect the security architecture of each project as generally documented.
> Update cadence and project status change over time — items flagged "verify" should be checked
> against current project documentation before being published.

---

## 1. ROM comparison

| | **GrapheneOS** | **CalyxOS** | **DivestOS** | **LineageOS** |
|---|---|---|---|---|
| Primary goal | Maximum security/privacy hardening | Usable privacy with Google-free defaults | Hardened LineageOS for a wide device range | De-Googled stock-like ROM, broad device support |
| Devices | **Pixel only** (requires the hardware below) | Pixel (+ a few others historically) | Very wide (LineageOS-supported devices) | Very wide |
| Verified boot with **custom key + re-locked bootloader** | **Yes** — full AVB chain re-established | Yes on Pixel | **Often not** — many supported devices can't re-lock with a custom key | **Usually not** — most builds run with an unlocked bootloader |
| Google services | Optional **sandboxed Play** (no system privilege) | **microG** (signature-spoofing reimplementation, runs privileged) | microG-capable / Google-free | None by default; Google apps via separate flashable |
| OS hardening (hardened malloc, attack-surface reduction, exploit mitigations) | **Extensive** | Some | LineageOS + added hardening patches | Minimal beyond upstream Android |
| Update cadence | **Fast** — typically among the quickest to ship Android/Pixel security patches | Slower than GrapheneOS | Was community-maintained; **verify current status** (the project announced it was winding down — confirm) | Varies by device/maintainer; often lags |
| Threat-model fit | Strongest for a security practitioner | Good for privacy-first general users | Best-effort hardening on older/non-Pixel hardware | Convenience/longevity, **not** a security ROM |

Key architectural distinctions:

- **microG vs sandboxed Play.** CalyxOS's microG *reimplements* Google APIs and runs with elevated
  privileges (and historically relied on signature spoofing). GrapheneOS instead runs the **real**
  Play in an unprivileged sandbox. GrapheneOS's approach keeps Google code in a normal app sandbox
  (no extra privilege, better compatibility); microG reduces data sent to Google but enlarges the
  privileged trusted base. Different trade-offs; for a hardening-first build the sandboxed-Play model
  is the more conservative one.
- **Verified boot is the dividing line.** ROMs that cannot re-lock the bootloader against a custom
  key (most LineageOS targets, many DivestOS targets) cannot offer a hardware-rooted verified-boot
  chain — leaving them exposed to offline OS tampering (evil-maid). This is the single most
  important reason a security build chooses GrapheneOS on Pixel (§2).

---

## 2. Why Pixel hardware specifically

GrapheneOS supports **only** Pixels, and that is a security decision, not a popularity one:

1. **Re-lockable bootloader with a user-supplied AVB key.** Pixels let you flash a custom OS, install
   your own verified-boot key, and **re-lock** the bootloader so the device enforces verified boot
   against *your* key. Almost no other Android OEM permits this — most either forbid unlocking
   entirely or permanently flag/limit the device once unlocked, so verified boot cannot be restored
   for a third-party OS. Without re-locking, you cannot get a hardware-rooted boot chain on a custom
   ROM at all.
2. **Dedicated secure element with attestation.** The Pixel security chip (referred to as Titan M2
   in the build notes — verify the exact generation) provides hardware key storage, **Weaver**
   (hardware-throttled credential verification), and **hardware key attestation** that even reports
   the verified-boot state. This is stronger than relying solely on the SoC's TrustZone/TEE.
3. **TrustZone/TEE vs a discrete secure element.** Generic ARM **TrustZone** is a TEE running on the
   *same* application processor as the OS — a large, shared trust domain. A **discrete secure
   element** is a separate chip with its own CPU and storage, a much smaller and physically isolated
   trusted base, harder to reach from a compromised application processor. Pixels provide both a TEE
   *and* a discrete secure element; the discrete element is what makes offline credential brute-force
   and key extraction impractical (see
   [grapheneos-security-model.md](grapheneos-security-model.md) §2).
4. **Long guaranteed support windows and prompt firmware/patch delivery** on recent Pixels shorten
   the n-day exposure window — essential for a security device.

The combination — re-lockable custom-key verified boot **plus** a discrete attesting secure element
**plus** prompt patches — is not available on other Android hardware, which is why "use a Pixel" is
effectively a hard requirement for this security model.

---

## 3. Trade-offs of GrapheneOS for a security practitioner

### What you give up vs stock Android / stock Pixel

- **Some app compatibility friction.** A minority of apps (certain banking apps, some that abuse
  hardware attestation or refuse to run without privileged Play) may not work or need workarounds.
  Sandboxed Play covers most cases but not all.
- **Google convenience features** that depend on privileged Play integration may be degraded or
  absent.
- **Self-install responsibility.** You own the flashing, key management, and update discipline
  (though GrapheneOS's installer and OTA are straightforward).
- **No carrier-branded features / some carrier-specific functionality** may differ.

### What you gain

- Hardware-rooted **verified boot under your own key** with a re-locked bootloader.
- **Sandboxed (unprivileged) Play** instead of privileged Google system services.
- **Extensive OS hardening** (hardened_malloc, attack-surface reduction, exploit mitigations,
  per-app/per-profile network & sensor permissions, Storage/Contact Scopes).
- **Fast security patches.**
- **Strong multi-user/profile support** with per-profile encryption and End-session key eviction —
  the basis for NULLBYTE's nine-profile compartmentalisation.
- **Duress/panic and auto-reboot-to-BFU** features relevant to physical-seizure threat models.

For a practitioner, the compatibility cost is the main downside, and it is mitigated precisely by
the compartmentalised design: keep awkward/privileged apps in a dedicated profile and keep them away
from sensitive ones.

---

## 4. NitroPhone — does the commercial variant add security value?

The **NitroPhone** (Nitrokey) is a Pixel **pre-flashed with GrapheneOS**, sold with support and, on
some variants, **physical hardware modifications**.

Honest assessment of what it adds:

- **The software is GrapheneOS** — the same OS you can flash yourself, for free. Cryptographically and
  architecturally there is **no security difference in the OS layer** between a self-flashed Pixel and
  a NitroPhone.
- **Value-add is convenience + optional hardware mods + support:**
  - Pre-flashed and verified, saving the install step.
  - Higher-tier variants have historically offered **physical modifications** such as **removed
    microphones** (and, on some, camera removal) for users whose threat model wants a hard guarantee
    against audio capture rather than a software toggle. *Verify which mods the current variant
    offers.*
  - Commercial support and procurement (useful for organisations).
- **Where it does NOT add security:** if you can flash GrapheneOS yourself and you do not need the
  physical microphone/camera removal, the NitroPhone provides **no additional cryptographic or
  software security** over a self-built device. The trust model is also worth noting: a self-flashed
  device lets *you* verify the install; a pre-flashed device asks you to trust the vendor's supply
  chain (you can still re-verify/re-flash yourself).

**Conclusion for NULLBYTE:** a self-flashed Pixel running GrapheneOS achieves the same software
security posture. The NitroPhone is worth considering **only** for the physical hardware
modifications (e.g. mic removal) if those are in scope for the threat model — otherwise it is a
convenience/support purchase, not a security upgrade.

---

## 5. Open items / to verify

- [ ] Current maintenance status of DivestOS (the project announced winding down — confirm).
- [ ] Exact secure-element generation on the Pixel 10 Pro Fold (Titan M2 vs successor).
- [ ] Current NitroPhone hardware-modification options (mic/camera removal) if relevant.
- [ ] Whether any required app (e.g. a specific banking app) forces a profile-placement decision.

> No project's update cadence, device list, or feature set is asserted as current without
> verification; treat the comparison as a design-rationale reference and confirm specifics against
> live project documentation.
