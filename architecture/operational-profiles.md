# Operational Profile Architecture

Living build — `MANUAL INPUT REQUIRED` markers flag values still to be captured from the running device; `FUTURE WORK` markers flag planned enhancements.

## Concept

The device is partitioned into **nine compartmentalised Android user profiles**. Each profile is
treated as a distinct security boundary, not a convenience feature. An application in one profile
cannot read the filesystem, contacts, clipboard, or network state of another. If a profile is
compromised — by a malicious app, a social-engineering vector, or a sandboxed-service zero-day —
the damage is contained and cannot propagate across the boundary.

Each profile carries a short codename: a memorable, operationally distinct identifier that
conveys **no information about purpose** to an observer who sees only the profile name.

## Profiles

| Profile | Operational purpose | App stack (category level) |
|---------|---------------------|----------------------------|
| Nexus | Owner — device administration, hardware-key management | System settings, hardware-key management; no sandboxed Play |
| Plague | Junk — disposable interactions, untrusted app testing | Untrusted apps under test; sandboxed Play where required; expendable |
| Ghost | Throwaway — short-lived registrations, ephemeral accounts | Minimal messaging/registration apps; sandboxed Play as needed; expendable |
| Abyss | Learning — courses, documentation, research reading | Hardened browser, document/PDF readers, course apps |
| Void | Pentesting — offensive tooling, CTF work | Termux + offensive tooling, network/CTF clients; broadest network, no sensitive data |
| Façade | Professional — work communications, productivity | Termux (IRONVEIL SSH unlock), productivity and comms apps |
| Shade | OSINT — open-source intelligence gathering | Hardened browser, research/collection tooling; no PII linkage |
| Vault | Financial — banking, payments | Banking/payment apps only; strictest restrictions |
| Joker | Fallback — backup operational identity | Minimal mirror of essential baseline apps; held in reserve |

> App stacks are described at category level by design. Specific package lists, account
> identifiers, and email addresses are intentionally **not** published.

## Encryption isolation model

- Each Android user profile has its **own encryption key**, derived from that profile's
  lockscreen credential.
- The owner profile (Nexus) key is further protected by the **Titan M2**.
- A **locked** profile's data is cryptographically inaccessible to every other profile —
  including the owner — without that profile's credential.

Practical consequence: if the device is handed over under legal compulsion or duress with only
one profile unlocked, data in all other profiles remains inaccessible.

## Profile-switching procedure

1. Open the quick-settings user switcher (or Settings → System → Multiple users).
2. Select the target profile and authenticate with that profile's lockscreen credential.
3. The previous profile is **locked** on switch — its keys are evicted, so its data is no longer
   in memory.

<!-- MANUAL INPUT REQUIRED: confirm whether End-session-on-switch / per-profile auto-lock timeout is enforced — check per-profile Security settings on the device -->

## Design principles

- No personally identifying information crosses profile boundaries.
- Elevated-sensitivity profiles (Vault, Nexus) carry the strictest app restrictions and avoid
  sandboxed Play unless a specific app requires it.
- Void (pentesting) holds the broadest network access but **no** financial or professional data,
  so compromise yields no sensitive personal material.
- Plague and Ghost are expendable — they can be deleted and recreated without loss.

<!-- MANUAL INPUT REQUIRED: record the per-profile lockscreen policy (biometric vs PIN vs passphrase) -->
<!-- MANUAL INPUT REQUIRED: record which profiles route via WireGuard vs cleartext (see network-security.md; from the RethinkDNS per-profile config) -->
