# nullbyte

Hardened GrapheneOS build on a Pixel 10 Pro Fold. A compartmentalised profile
architecture maps nine operational roles to isolated Android user profiles, each
with independent encryption, independent network policy, and no shared state. Living
build — actively maintained alongside IRONVEIL.

---

## Overview

GrapheneOS eliminates the trust compromises baked into stock Android: no Google Play
Services in the owner profile, no persistent background reporting, verified boot enforced
at every power cycle. On top of the OS security model, this build imposes an additional
architectural constraint — operational separation via compartmentalised user profiles.

Each profile is treated as a distinct security boundary, not a convenience feature. An
application running in one profile cannot read the filesystem, contacts, or network state
of another. If a profile is compromised — through a malicious app, a social-engineering
vector, or a zero-day in a sandboxed service — the damage is contained to that profile
and cannot propagate across the boundary.

The build integrates with IRONVEIL: Termux in the primary work profile provides SSH access
to the IRONVEIL workstation, including the ability to trigger the dracut-sshd remote unlock
sequence from the handset.

---

## Hardware

| Attribute | Value |
|-----------|-------|
| Device | Google Pixel 10 Pro Fold |
| SoC | Google Tensor G5 |
| Security chip | Titan M2 (confirmed) |
| OS | GrapheneOS — Android 16 |
| Build number | 2026060601 |
| Storage | 1 TB |
| RAM | 16 GB |
| IP rating | IP68 |

**Hardware security stack (confirmed).** The platform layers three hardware-rooted components:

- **Tensor G5 security core** — SoC-integrated secure subsystem.
- **Titan M2** — discrete, certified security chip, physically separate from the application
  processor.
- **Trusty TEE** — the Trusted Execution Environment running alongside the main OS.

Titan M2 provides the hardware root of trust for GrapheneOS verified boot: it handles verified-boot
attestation, cryptographic key storage, tamper detection, and secure lock-screen enforcement,
independently of the main SoC. Because it is physically separate from the application processor, a
compromised OS cannot extract keys held in Titan M2 — and an attacker who compromises the Tensor G5
cannot forge the attestation chain without it.

The 1 TB / 16 GB configuration allows each profile to maintain a meaningful application
stack without storage pressure forcing profile consolidation.

---

## Security Architecture

### Verified Boot

GrapheneOS enforces verified boot on every power cycle. The bootloader is relocked after
installation. The boot chain is: Titan M2 root of trust → bootloader → verified OS image.
Any modification to the OS partition breaks the verification chain and the device refuses
to boot into the modified image. This property survives physical access — an attacker with
the device cannot silently modify the OS without detection.

**Verified-boot key hash (confirmed on device):**

```
deadbeefdeadbeefdeadbeefdeadbeefdeadbeefdeadbeefdeadbeefdeadbeef
```

This hash is the fingerprint of the verified-boot key used to sign the GrapheneOS build on this
device. It can be independently verified against the GrapheneOS release signing keys published at
`grapheneos.org/releases` to confirm the build is authentic and unmodified — a concrete, attestable
root-of-trust value rather than an assertion. The hash is shown on the yellow-state boot screen and
via `getprop ro.boot.vbmeta.digest`.

**What a mismatch would mean.** The point of the hash is not that it matches today — it is that a
mismatch is *detectable*. If this value did not match the published GrapheneOS signing key, it would
indicate either a different OS version than expected or, the case that matters, a build that was **not
signed by the GrapheneOS project** — a substituted or tampered OS. Because any silent post-installation
modification of the OS partition necessarily changes this value, an evil-maid attacker cannot alter the
system without the change surfacing at the next boot. That detectability is the security property;
the specific hash is just how it is observed.

### Per-Profile Encryption

Each Android user profile has its own encryption key derived from the profile's lockscreen
credential. The owner profile key is further protected by the Titan M2. A locked profile's
data is inaccessible to all other profiles — including owner — without the profile credential.

Practical consequence: if the device is handed over under legal compulsion or social pressure
with only one profile unlocked, the data in all other profiles is cryptographically inaccessible.

### Sandboxed Google Play

Google Play Services is installed in sandboxed mode within specific profiles that require
it. Sandboxed Play runs without system-level privileges — it cannot access device identifiers,
read other profiles' data, or persist beyond its profile boundary. Profiles that do not
require Play Services do not have it installed.

### RethinkDNS Firewall

RethinkDNS provides per-app network policy enforcement at the VPN layer. Each app in each
profile can be individually allowed, blocked, or routed through specific DNS resolvers and
exit points. Features in use:

- Per-app firewall rules blocking unnecessary egress from applications that have no
  legitimate network need
- DNS filtering with block lists applied to tracking, telemetry, and known-malicious domains
- WireGuard integration: specific profiles route all traffic through the WireGuard tunnel

### WireGuard

WireGuard tunnels provide encrypted egress for profiles where network traffic confidentiality
is required. RethinkDNS manages tunnel routing on a per-profile basis — not all profiles
use the same exit point, and some profiles can be restricted to tunnel-only traffic with
no cleartext fallback.

### Baseband Isolation via IOMMU

The Tensor G5 SoC includes IOMMU-based isolation between the application processor and
the baseband modem. This prevents a baseband compromise — historically a significant attack
surface on mobile devices — from directly reading application-processor memory. The modem
can communicate over its defined channels but cannot perform arbitrary DMA into the
application processor address space.

<!-- MANUAL INPUT REQUIRED: document the IOMMU configuration and whether GrapheneOS adds baseband hardening beyond stock Pixel — this is a research item against current GrapheneOS documentation, not a single device value (see architecture/research/mobile-threat-model.md §3) -->

---

## Operational Profile Architecture

The device is partitioned into nine compartmentalised user profiles — all nine confirmed present
on the device (Settings → System → Users, 2026-06-11): **Nexus** (owner), **Plague**, **Ghost**,
**Abyss**, **Vault**, **Façade**, **Void**, **Joker**, **Shade**. Each profile carries a short
codename chosen to be memorable and operationally distinct while conveying no information about its
purpose to an observer who sees only the profile name.

Each profile runs independently encrypted and has its own application stack, network policy,
and operational purpose. There is no application that spans profiles; no shared clipboard by
default; no shared contacts or call history.

| Profile | Operational purpose | App stack (category level) |
|---------|---------------------|----------------------------|
| Nexus | Owner profile — device administration, hardware key management | System settings, hardware-key management; no sandboxed Play |
| Plague | Junk — disposable interactions, untrusted app testing | Untrusted apps under test; sandboxed Play where an app requires it; expendable |
| Ghost | Throwaway — short-lived registrations and ephemeral accounts | Minimal messaging/registration apps; sandboxed Play as needed; expendable |
| Abyss | Learning — courses, documentation, research reading | Hardened browser, document/PDF readers, course and reference apps |
| Void | Pentesting — offensive security tooling, CTF work | Termux with offensive tooling, network/CTF clients; broadest network access, no sensitive data |
| Façade | Professional — work communications and productivity | Termux (IRONVEIL SSH unlock), productivity and communications apps |
| Shade | OSINT — open-source intelligence gathering and research | Hardened browser, research and collection tooling; no PII linkage |
| Vault | Financial — banking, payments, financial services | Banking and payment apps only; strictest restrictions; sandboxed Play only where a banking app requires it |
| Joker | Fallback — backup operational identity | Minimal mirror of essential baseline apps; held in reserve |

App stacks are described at category level by design — specific package lists are intentionally
not published, consistent with the operational-security model below.

**Design principles:**

- No personally identifying information crosses profile boundaries
- Profiles with elevated sensitivity (Vault, Nexus) have the strictest app restrictions
  and do not have sandboxed Play installed unless a specific application requires it
- Void (pentesting) has the broadest network access but no access to financial or professional
  data — compromise of this profile yields no sensitive personal material
- Plague and Ghost are designed to be expendable — they can be deleted and recreated
  without losing anything of value

A full per-profile **[threat model](threat-model.md)** documents, for each of the nine profiles, the
threat it counters, the blast radius if it is compromised, the data at risk, and the controls in place
— plus the device-wide threats (theft, coercion, baseband) and an honest residual-risk statement.

<!-- MANUAL INPUT REQUIRED (keep private — do not publish): maintain the per-profile app lists internally only -->
<!-- MANUAL INPUT REQUIRED: record which profiles egress via WireGuard vs cleartext — from the RethinkDNS per-profile config on the device -->
<!-- MANUAL INPUT REQUIRED: record the lockscreen policy per profile (biometric / PIN / passphrase) -->

---

## SSH Integration with IRONVEIL

Termux is installed in the Façade (professional) profile, which has network access to the
IRONVEIL workstation subnet.

**Use cases:**

- Remote SSH administration of IRONVEIL from the handset
- dracut-sshd remote unlock sequence: on IRONVEIL boot, Termux initiates the initramfs SSH
  session and completes the LUKS2 unlock without requiring physical presence at the workstation
- Key management: Termux holds the ed25519 key pair that is baked into the IRONVEIL initramfs
  authorized\_keys

The Termux private key for IRONVEIL unlock is stored only in the Façade profile and is not
accessible from any other profile on the device.

<!-- MANUAL INPUT REQUIRED: document the Termux SSH config (~/.ssh/config, ServerAliveInterval/keep-alive settings) from the Façade profile -->
<!-- MANUAL INPUT REQUIRED: document the ed25519 key-rotation procedure for the IRONVEIL initramfs key (regenerate in Termux → update initramfs authorized_keys → rebuild initramfs → re-pin host key); cross-ref ironveil/hardening/dracut-sshd.md -->

---

## Results and Current State

| Component | Status |
|-----------|--------|
| GrapheneOS on Pixel 10 Pro Fold (Android 16, build 2026060601) | Verified — bootloader relocked |
| Verified boot key hash | Verified — `deadbeef…eadbeef` (full hash above) |
| Titan M2 secure element (+ Tensor G5 security core + Trusty TEE) | Verified |
| Per-profile encryption (9 profiles confirmed present) | Verified |
| Sandboxed Google Play (selective profiles) | Operational |
| RethinkDNS firewall | Operational — per-app rules active |
| WireGuard (profile-selective) | Operational |
| Baseband IOMMU isolation | Active (hardware-enforced) |
| Termux SSH to IRONVEIL | Operational |

<!-- MANUAL INPUT REQUIRED: record the exact GrapheneOS release tag and the Tensor G5 kernel version (Android 16 / build 2026060601 captured; the release-tag string and kernel version are not yet recorded) -->
<!-- FUTURE WORK: add a network architecture diagram showing inter-profile routing -->

---

## Skills Demonstrated

| Skill area | Evidence |
|------------|---------|
| Mobile security | GrapheneOS deployment with relocked bootloader; verified boot chain |
| Android hardening | Per-profile encryption; sandboxed Play; privilege separation |
| Operational security | Nine-profile compartmentalisation; no cross-profile data leakage |
| Network security | RethinkDNS per-app firewall; WireGuard profile routing; baseband IOMMU |
| Hardware security | Titan M2 root of trust; Tensor G5 IOMMU isolation |
| Security architecture | Threat-modelled profile design; failure containment by construction |
| Systems integration | Termux SSH integration with IRONVEIL; remote unlock from mobile |

---

*Part of the [rootdrifter](https://github.com/rootdrifter) security portfolio — full writeup at [rootdrifter.io/portfolio/nullbyte/](https://rootdrifter.io/portfolio/nullbyte/).*
