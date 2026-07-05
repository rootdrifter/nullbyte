# Mobile Threat Model — Reference

Threat-model reference for the NULLBYTE hardened mobile platform. Complements the adversary table
in [../security-model.md](../security-model.md) and the compartmentalisation design in
[../operational-profiles.md](../operational-profiles.md). Honest about what the build does and does not stop.

> See the secure-element naming caveat in
> [grapheneos-security-model.md](grapheneos-security-model.md) (Titan M2 vs successor) — confirm
> the chip generation for the Pixel 10 Pro Fold before publishing hardware-specific claims.

---

## 1. Who the adversaries are, and what they can do

| Adversary | Realistic capability | What they're after |
|-----------|----------------------|--------------------|
| **Opportunistic thief** | Physical possession of a powered-off or locked device | Resale; incidental data access |
| **Border / checkpoint inspection** | Compelled handover, possibly compelled unlock of one profile; forensic imaging tools (e.g. commercial mobile-forensic kits) | Imaging storage; reviewing accessible data |
| **Targeted physical adversary** | Time-bounded physical access ("evil maid"); may attempt hardware tamper | OS implantation; data extraction |
| **Network adversary** | Passive observation, active MITM on hostile Wi-Fi; rogue base station (IMSI catcher) | Traffic capture, location, injection |
| **Carrier-level / SS7 adversary** | Access to telecom signalling | Location tracking, SMS/call interception (esp. SMS OTPs) |
| **Malicious app / supply chain** | Code execution inside one profile | Data theft, persistence, lateral movement |
| **Coercive adversary ("$5 wrench")** | Legal or physical compulsion of the *person* | Forced unlock / disclosure |
| **Nation-state with 0-day** | Remote 0-click/1-click chains, baseband or kernel exploits | Full device compromise |

The build is explicitly designed against the first six and meaningfully raises cost for the
seventh and eighth — but does **not** claim to defeat a resourced 0-day operator (§5).

---

## 2. Physical attack vectors

### 2.1 What the secure element + verified boot **do** protect against

- **Offline OS modification (evil-maid on the OS partition).** A re-locked bootloader with verified
  boot refuses to boot a tampered image; dm-verity catches read-only-partition tampering. See
  [grapheneos-security-model.md](grapheneos-security-model.md) §1.
- **Storage cloning / chip-off against a powered-off device.** FBE keys are bound to the secure
  element with hardware-throttled credential checks (Weaver). Imaged flash is ciphertext; brute
  force can only run on-device through the rate limiter. A strong random credential is therefore
  effectively unbreakable by extraction (ibid. §2).
- **Downgrade-to-vulnerable.** Rollback protection rejects older signed images.
- **Live extraction after seizure** is mitigated by returning the device to a **Before First Unlock
  (BFU)** state — auto-reboot-after-timeout evicts keys from memory so forensic tools face an
  unauthenticated, encrypted device. *Verify this timeout is enabled.*

### 2.2 What it does **not** protect against

- **Coercion / the "$5 wrench."** No at-rest cryptography survives a user being compelled to unlock.
  Mitigation is **compartmentalisation + duress features**, not cryptography: hand over a device on
  which only an innocuous profile is unlocked; other profiles remain BFU-encrypted. *Confirm whether
  a duress PIN (wipe-on-entry) is configured.*
- **After-First-Unlock (AFU) seizure.** If the device is taken while unlocked (or in AFU with the
  target profile's keys resident), far more is recoverable. The defence is to keep sensitive
  profiles locked and minimise the unlocked window.
- **Hardware implants / sophisticated lab tamper.** IP68 sealing and tamper detection raise cost
  but a resourced lab attacker with the device for an extended period is out of scope.
- **Screen/credential observation (shoulder-surf, camera).** Operational, not technical — mitigate
  with credential discipline.

---

## 3. Network attack vectors

### 3.1 Baseband isolation via IOMMU — what it actually prevents

The cellular **baseband** runs on a separate processor with historically poor memory safety and a
large remote attack surface (it parses untrusted radio protocols). On the Tensor platform the
application processor is isolated from the baseband by an **IOMMU**, which constrains what memory
the modem can reach via DMA.

- **What it prevents:** a baseband compromise cannot perform *arbitrary DMA* into application-processor
  memory to pivot into the OS. The modem is confined to its defined shared-memory channels.
- **What it does NOT prevent:** anything the baseband can do *within its own role* — it still
  handles your radio traffic, so a compromised baseband (or a rogue base station) can affect
  connectivity, and the network can still see metadata. IOMMU isolation is a containment boundary,
  not an end-to-end confidentiality control.

### 3.2 SS7 and the carrier signalling surface

**SS7/Diameter attacks operate on the carrier signalling network, not on the device.** No amount of
on-device hardening (verified boot, IOMMU, GrapheneOS) defends against them, because the device is
not in the loop:

- **Location tracking** via signalling queries.
- **SMS interception / redirection** — directly defeats **SMS-based OTP/2FA**.
- **Call interception/redirection** in some configurations.

Operational mitigations (the only ones that work here):
- **Do not use SMS for 2FA** anywhere it matters — use app-based TOTP or hardware security keys.
- **Use end-to-end encrypted messaging** so signalling-level interception yields ciphertext.
- **Tunnel data egress** (WireGuard, per [../network-security.md](../network-security.md)) so data
  traffic confidentiality does not depend on the carrier.
- Treat the phone number itself as a weak identifier/attack surface; avoid binding sensitive
  accounts to it.

### 3.3 Local / Wi-Fi network adversary

- **Rogue Wi-Fi / MITM:** mitigated by WireGuard egress (ChaCha20-Poly1305, authenticated peers —
  see [../../ironveil/hardening/research/adguard-wireguard-reference.md] in the IRONVEIL repo for
  the crypto detail) and per-app firewalling via RethinkDNS.
- **IMSI catcher (rogue base station):** can capture metadata and attempt downgrade; mitigated
  partially by disabling 2G (where supported) and by tunnelling data. Not fully solved on-device.
- **MAC/identifier tracking:** GrapheneOS per-network MAC randomisation reduces cross-network
  correlation.

---

## 4. Application-layer threats and why profile isolation matters

### 4.1 The role of profile isolation

Each profile is a separate security boundary with its own encrypted storage (see
[grapheneos-security-model.md](grapheneos-security-model.md) §3). A malicious or trojanised app is
confined to its profile: it cannot read another profile's files, contacts, clipboard, accounts, or
network state. This directly bounds the blast radius of:

- a trojanised app installed in a junk/throwaway profile (Plague/Ghost),
- a sandboxed-service exploit, or
- a social-engineering install.

### 4.2 Residual cross-profile leakage vectors (be honest)

Profile isolation is strong but the shared substrate creates real residual vectors:

- **Shared kernel / `system_server` exploit** → potential cross-profile escalation (the main one).
- **TEE / secure-element compromise** → device-wide.
- **Side channels** (sensors, timing, microarchitectural) not partitioned by profiles.
- **User-mediated leakage** — manually copying data, reusing the same account/identifier across
  profiles, or photographing one profile's screen from another. The model assumes operator
  discipline ("no PII across boundaries").
- **Simultaneously-unlocked profiles** widen the window; only *locked* profiles are
  cryptographically out of reach.

---

## 5. Operational security for a mobile pentesting platform (the Void profile)

The Void profile holds offensive tooling (Termux, network/CTF clients) and has the **broadest
network reach** of any profile.

### 5.1 What Void-profile isolation actually achieves

- **Contains tooling risk.** Offensive tools, dependencies pulled from package repos, and target
  interactions are confined to one profile. A compromise there cannot reach Vault (financial),
  Façade (professional), or Nexus (owner).
- **No sensitive data to lose.** By design Void holds **no** financial or professional data, so
  even a same-session compromise of Void yields nothing personally valuable — the deliberate
  "broad surface, low value" placement.
- **Clean teardown.** The profile can be reset without touching other identities.

### 5.2 Gaps that remain

- **Shared kernel/TEE** (§4.2) — a kernel exploit triggered by hostile tooling or a malicious target
  response could, in principle, escape the profile.
- **Attribution / OPSEC on the network side.** Profile isolation does nothing for network-level
  attribution; that depends on the WireGuard exit and avoiding identifier reuse. Running offensive
  traffic from a personally-attributable exit is an OPSEC failure the profile model does not fix.
- **Legal/authorisation scope.** A pentesting platform's biggest "threat" is operating outside
  authorisation — out of scope for technical controls but the most important operational rule.
- **Tooling supply chain.** Packages and scripts pulled into Termux are an ingestion risk; pin and
  review sources.
- **Clipboard/notification bleed** if the OS surfaces content across the lock boundary — verify
  notification privacy settings.

---

## 6. Threat-model summary matrix

| Vector | Primary control | Residual risk |
|--------|-----------------|---------------|
| Offline OS tamper | Verified boot + locked bootloader | Lab-grade hardware tamper |
| Storage extraction (BFU) | FBE + secure-element-bound keys + Weaver throttling | Weak credential; AFU seizure |
| Coercion | Compartmentalisation + duress features | The person can still be compelled |
| Baseband exploit | IOMMU isolation | Connectivity/metadata effects within the modem's role |
| SS7 / carrier | **Off-device — not mitigated by the platform** | Use non-SMS 2FA + E2EE + tunnelled data |
| Hostile Wi-Fi / MITM | WireGuard + RethinkDNS | IMSI catcher metadata; on-device DoH bypass |
| Malicious app | Profile isolation | Shared kernel/TEE; user-mediated leakage |
| 0-day operator | Attack-surface reduction + fast patches | Not defeated; cost-raising only |

---

## 7. Open items / to verify

- [ ] Secure-element generation (Titan M2 vs successor) on the Pixel 10 Pro Fold.
- [ ] Auto-reboot-to-BFU timeout enabled; duress PIN configured.
- [ ] Whether 2G is disabled to reduce IMSI-catcher downgrade surface.
- [ ] Per-profile notification/clipboard privacy across the lock boundary.
- [ ] Which profiles are tunnel-only vs split-tunnel (cross-ref network-security.md TODO).

> No CVEs or vendor capability claims are asserted without verification. SS7/baseband behaviour is
> described at a mechanism level; confirm specifics before reuse in published material.
