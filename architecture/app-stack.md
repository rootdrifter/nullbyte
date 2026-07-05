# Application Stack

Living build — `MANUAL INPUT REQUIRED` markers flag values still to be captured from the running device; `FUTURE WORK` markers flag planned enhancements.

This document describes the application stack **by category**. Specific per-profile package lists,
account identifiers, and email addresses are intentionally **not** published, consistent with the
operational-security model in [security-model.md](security-model.md) and the compartmentalisation
described in [operational-profiles.md](operational-profiles.md).

## App sources (in trust order)

Applications are obtained from the most-trusted available source for each app. Source choice is a
security decision: GrapheneOS's own apps and F-Droid/Accrescent builds are preferred over the
Play Store, and the Play Store — where used — runs **sandboxed**, not privileged.

| Source | Trust / role | Notes |
|--------|--------------|-------|
| GrapheneOS App Store | Highest — first-party | GrapheneOS apps and core components; signed by GrapheneOS |
| Accrescent | High — modern, signed app store | Reproducible, signed builds; preferred for supported apps |
| F-Droid | High — FOSS, auditable | Open-source apps built from source |
| Aurora Store | Medium — Play proxy | Anonymous front-end to Play catalogue without a Google account |
| Sandboxed Play Store | Lowest — used only where required | Runs as an unprivileged app; installed only in profiles that need it |

## Stack by category

### Privacy and security

| App | Purpose | Typical source |
|-----|---------|----------------|
| Vanadium | Hardened Chromium-based browser (GrapheneOS default) | GrapheneOS App Store |
| Aegis | Offline TOTP/2FA authenticator (encrypted vault) | F-Droid / Accrescent |
| RethinkDNS | Per-app firewall, DNS filtering, WireGuard chaining | F-Droid / Play |
| ProtonVPN | Commercial VPN egress | Play (sandboxed) / F-Droid |
| WireGuard | Encrypted tunnel egress, per-profile routing | F-Droid / Play |
| Tor Browser | Anonymised browsing over the Tor network | F-Droid (Guardian Project) |

### Productivity and remote access

| App | Purpose | Typical source |
|-----|---------|----------------|
| Termux | Terminal emulator; SSH to IRONVEIL incl. dracut-sshd unlock (Façade profile) | F-Droid |
| Termius | Managed SSH client | Play (sandboxed) |
| aFreeRDP | RDP client for remote desktop sessions | F-Droid |

### Communication

| App | Purpose | Typical source |
|-----|---------|----------------|
| Signal | End-to-end encrypted messaging and calls | Play (sandboxed) / direct APK |
| Briar | Peer-to-peer encrypted messaging (works over Tor / local mesh, no central server) | F-Droid |

### Maps and navigation

| App | Purpose | Typical source |
|-----|---------|----------------|
| OsmAnd | Offline OpenStreetMap navigation | F-Droid |
| Organic Maps | Lightweight offline OSM maps | F-Droid / Accrescent |

### Finance

Banking and payment apps are confined to the **Vault** profile (strictest restrictions).

> **Compatibility note:** some finance apps perform hardware-attestation or root/integrity checks
> (Play Integrity API) that can fail on GrapheneOS. Workarounds — installing sandboxed Play
> Services within the Vault profile, and enabling the relevant compatibility toggles — are
> required for these apps to function. Each finance app is evaluated individually; apps that
> demand privileged Play Services rather than sandboxed are not installed.

<!-- MANUAL INPUT REQUIRED (keep private — do not publish specific apps/accounts): note internally which finance app(s) needed the sandboxed-Play / attestation workaround -->

## Mapping apps to profiles

App categories map onto the operational profiles defined in
[operational-profiles.md](operational-profiles.md):

| Profile | Representative categories |
|---------|---------------------------|
| Nexus (owner) | System/security only; no sandboxed Play |
| Void (pentest) | Termux + offensive tooling; broad network, no sensitive data |
| Façade (professional) | Termux (IRONVEIL SSH), productivity, communication |
| Shade (OSINT) | Hardened browser, Tor Browser, research tooling |
| Vault (financial) | Banking/payment apps only; sandboxed Play where required |
| Abyss (learning) | Browser, document/PDF readers, course apps |

<!-- MANUAL INPUT REQUIRED (keep private — do not publish): finalise the exact per-profile category-to-app assignment internally only -->

## Current state

| Item | Status |
|------|--------|
| First-party / FOSS sources prioritised | In place |
| Sandboxed Play confined to profiles that require it | In place |
| Finance apps isolated to Vault with attestation workarounds | Operational |
| Termux → IRONVEIL SSH (Façade) | Operational |
