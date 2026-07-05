# nullbyte — Threat Model

A structured threat model for the nine-profile compartmentalised mobile architecture. The
[operational-profiles](architecture/operational-profiles.md) document describes *what* each profile is
for; this document asks the harder question for each one: **what is it defending against, what is lost
if it falls, and what stops that.** The aim is to show threat-modelling discipline — adversary-driven
reasoning, explicit trust boundaries, and honest residual-risk acknowledgement — not just a feature
list.

> Method: per-profile attack-surface analysis framed against a defined adversary set, plus device-wide
> threats that cross all profiles. Privacy-impact reasoning follows the data-isolation goal
> (no PII linkage across boundaries). Specific package lists, accounts, and identifiers are
> intentionally out of scope (private by policy).

## 1. Adversary model

| Adversary | Capability | Primary goal | In scope? |
|-----------|-----------|--------------|-----------|
| Opportunistic thief | Physical possession of a locked device | Resale; casual data access | ✅ |
| Malicious app / sandboxed-service 0-day | Code execution inside one profile | Data theft, lateral movement | ✅ |
| Network adversary (hostile Wi-Fi, rogue AP) | On-path traffic interception, DNS manipulation | Surveillance, credential capture | ✅ |
| Targeted social-engineering | Trick the user into installing/authorising | Foothold in a sensitive profile | ✅ |
| Coercion / lawful compulsion | Demand the device be unlocked | Access to all data | ✅ (partial mitigation) |
| Supply-chain / baseband | Compromise below the OS (modem, firmware) | Persistent, OS-invisible access | ◐ (hardware-dependent) |
| Nation-state full-chain exploit | Chained 0-days, unlimited resources | Total compromise | ⬜ (out of realistic scope — acknowledged) |

**Trust anchor.** All software trust derives from a hardware root of trust: **verified boot anchored in
the Titan M2 secure element** (full chain: Tensor G5 security core → Titan M2 → Trusty TEE). Each boot
stage's signature is verified against the immutable anchor; a tampered boot chain fails verification
before user data is decrypted. This is the foundation every profile-level control sits on — if the
boot chain is trusted, the per-profile encryption guarantees hold.

## 2. The core trust boundary — profile isolation

Each of the nine Android user profiles is a **distinct security boundary** with its **own encryption
key**, derived from that profile's lockscreen credential (the owner key additionally protected by Titan
M2). A *locked* profile's data is cryptographically inaccessible to every other profile, including the
owner. On profile switch, the previous profile is locked and its keys evicted from memory.

**The single most important property:** a compromise of one profile — malicious app, sandboxed-service
0-day, or social-engineering — is **contained**. It cannot read another profile's filesystem,
contacts, clipboard, or network state, and cannot propagate across the boundary. The threat model
below is therefore *per-profile blast-radius analysis*: assume one profile falls, and reason about
exactly what is and is not lost.

## 3. Per-profile threat analysis

For each profile: the threat it counters · compromise impact (blast radius) · data at risk · controls
(digital + physical).

### Nexus — Owner / device administration
- **Counters:** unauthorised device administration, hardware-key misuse, settings tampering.
- **If compromised:** highest-severity — owner can manage other profiles' lifecycle and device policy.
  But owner *cannot* read a locked profile's data (separate keys), so even owner compromise does not
  yield other profiles' decrypted contents.
- **Data at risk:** device configuration, hardware-key management material.
- **Controls:** strictest app restrictions; **no sandboxed Play**; owner key bound to Titan M2;
  reserved for administration, not daily use → minimal exposure surface.

### Vault — Financial
- **Counters:** banking-credential theft, payment fraud, financial-app supply-chain risk.
- **If compromised:** direct financial loss; the highest-value *data* target.
- **Data at risk:** banking/payment app data and sessions only — and *nothing else* (no professional,
  OSINT, or pentest data lives here).
- **Controls:** strictest restrictions; banking/payment apps **only**; sandboxed Play only where a
  specific banking app demands it; isolated from every other profile's data. Concentrating financial
  apps here and *excluding everything else* minimises both the attack surface and the blast radius.

### Façade — Professional
- **Counters:** compromise of work communications; theft of the IRONVEIL remote-unlock key.
- **If compromised:** loss of work comms confidentiality; **the [ironveil](../ironveil) SSH unlock key
  is here** — its exposure would aid (but not alone achieve) remote-unlock abuse.
- **Data at risk:** professional communications/productivity data; the Termux SSH private key for
  ironveil pre-boot unlock.
- **Controls:** the unlock key is stored **only** in Façade and nowhere else; Termux network access
  scoped to the unlock path; FIDO2/touch still required at the ironveil end, so a stolen key is not
  sufficient by itself (defence in depth across the two builds).

### Void — Pentesting / CTF
- **Counters:** the inherent risk of running offensive tooling — tool-supply-chain compromise, an
  exploit "turning around," noisy/risky network activity.
- **If compromised:** the *intended* sacrificial surface. Broadest network access, so the most likely
  to be attacked — but holds **no sensitive personal, financial, or professional data**, so a
  compromise yields nothing of value.
- **Data at risk:** ephemeral CTF/lab artefacts only.
- **Controls:** isolated from all sensitive profiles; broad network access deliberately paired with a
  zero-value data set; the offensive tooling is quarantined here so it can never touch Vault/Façade.

### Shade — OSINT
- **Counters:** **attribution / linkage** — the privacy threat of OSINT activity being tied back to a
  real identity.
- **If compromised:** exposure of research activity and sources; risk of de-anonymisation if linkage
  controls failed.
- **Data at risk:** research/collection data — **no PII linkage** by design.
- **Controls:** hardened browser; strict no-PII-crosses-the-boundary rule; collection tooling isolated
  so OSINT activity cannot be correlated with personal/professional identities.

### Abyss — Learning
- **Counters:** malicious documents/PDFs and hostile course/reference content (the classic
  document-borne exploit vector).
- **If compromised:** exposure of study material; foothold from a malicious document.
- **Data at risk:** course material, reference documents, reading history.
- **Controls:** hardened browser + document readers isolated here; untrusted documents open in a
  profile that holds nothing sensitive.

### Plague — Junk / untrusted-app testing
- **Counters:** untrusted/unknown apps — the deliberate detonation chamber.
- **If compromised:** **expected and acceptable.** This profile exists to *be* compromised safely.
- **Data at risk:** nothing of value — disposable interactions only.
- **Controls:** **expendable** — deleted and recreated at will; sandboxed Play where an app requires
  it; total isolation means a malicious test app cannot reach any real data.

### Ghost — Throwaway / ephemeral registration
- **Counters:** registration-time tracking, ephemeral-account linkage, throwaway-signup risk.
- **If compromised:** minimal — short-lived accounts with no durable value.
- **Data at risk:** ephemeral registration/messaging data only.
- **Controls:** expendable; minimal app set; recreated frequently so no long-lived linkage accrues.

### Joker — Fallback
- **Counters:** loss of operational continuity if a primary profile fails or must be destroyed.
- **If compromised:** loss of the reserve identity; low impact as it is a minimal mirror.
- **Data at risk:** minimal baseline app data only.
- **Controls:** held in reserve, minimal footprint; exists for resilience (availability), not daily
  exposure.

## 4. Device-wide threats (cross all profiles)

| Threat | Mitigation | Residual risk |
|--------|-----------|----------------|
| **Device theft (locked)** | Per-profile encryption; Titan M2-bound owner key; verified boot | Low — data at rest is cryptographically protected per profile |
| **Coercion / lawful compulsion** | Hand over with one profile unlocked → all *other* profiles stay cryptographically inaccessible | Partial — the unlocked profile is exposed; plausible-minimal-profile unlock limits this |
| **Hostile network / DNS manipulation** | WireGuard full-tunnel routing + RethinkDNS per-profile; no cleartext DNS on sensitive profiles | Per-profile routing config must be correct (see [network-security](architecture/network-security.md)) |
| **Malicious app install** | App contained to its profile; sensitive profiles restrict sandboxed Play | Profile-local damage only — no cross-boundary propagation |
| **Baseband / modem compromise** | Cellular baseband isolated from the application processor via IOMMU | ◐ Hardware-dependent; below-OS threats are acknowledged, not fully eliminated |
| **Supply-chain (OS/firmware)** | GrapheneOS verified boot; signed updates; Titan M2 rollback protection | Low for OS; firmware/baseband is the harder residual |
| **Full-chain nation-state 0-day** | Out of realistic scope | **Accepted** — no consumer device defeats this; honesty over false assurance |

## 5. What this architecture deliberately does **not** claim

- It does **not** defend against a determined nation-state full-chain exploit — that is out of scope
  and stated plainly rather than papered over.
- It does **not** make a *single* profile invulnerable — the design assumes individual profiles will
  fall and engineers for **containment**, not prevention-everywhere.
- Below-OS threats (baseband, firmware) are **mitigated, not eliminated** — the honest residual.

## 6. Why this matters (skills demonstrated)

This is threat *modelling*, not tool configuration: a defined adversary set, explicit trust boundaries,
per-asset blast-radius reasoning, defence-in-depth across two builds (the Façade↔ironveil unlock key),
and — most importantly — **honest residual-risk acknowledgement**. The compartmentalisation is an
applied instance of the principles a security team uses daily: least privilege (each profile holds only
what it needs), separation of duties (financial ≠ pentest ≠ OSINT), blast-radius minimisation, and a
hardware root of trust anchoring the whole stack.

---

*Part of the [rootdrifter](https://github.com/rootdrifter) security portfolio. Cross-references:
[operational-profiles](architecture/operational-profiles.md) ·
[security-model](architecture/security-model.md) ·
[research/mobile-threat-model](architecture/research/mobile-threat-model.md) ·
[ironveil](../ironveil) (the paired remote-unlock build).*
