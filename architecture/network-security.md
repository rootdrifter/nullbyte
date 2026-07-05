# Network Security Layer

Living build — `MANUAL INPUT REQUIRED` markers flag values still to be captured from the running device; `FUTURE WORK` markers flag planned enhancements.

## RethinkDNS — per-app firewall and DNS control

RethinkDNS enforces network policy at the VPN layer, per app, per profile:

- **Per-app firewall:** apps with no legitimate network need are blocked from egress entirely.
- **DNS filtering:** block lists applied to tracking, telemetry, and known-malicious domains.
- **WireGuard integration:** selected profiles route all traffic through a WireGuard tunnel,
  managed on a per-profile basis.

Because Android exposes a single VPN slot per profile, RethinkDNS acts as the on-device policy
engine and can chain to a WireGuard endpoint where confidentiality is required.

<!-- MANUAL INPUT REQUIRED: record the specific RethinkDNS block lists and any custom rules — copy from the RethinkDNS app → Configure → DNS / Firewall on the device -->

## WireGuard

WireGuard provides encrypted egress for profiles where traffic confidentiality is required.
Routing is per-profile — not all profiles share an exit point, and some profiles are restricted
to **tunnel-only** traffic with no cleartext fallback.

<!-- MANUAL INPUT REQUIRED: enumerate which profiles are tunnel-only vs split-tunnel — from the RethinkDNS / WireGuard per-profile routing config -->

## Sandboxed Google Play

Where a profile requires Google services, Google Play is installed in **sandboxed mode**:

- Runs as an ordinary app with **no system-level privileges**.
- Cannot read device identifiers or other profiles' data.
- Cannot persist beyond its profile boundary.

Profiles that do not require Play services do not have it installed (e.g. Nexus, and Vault
unless a banking app requires it).

## Baseband isolation via IOMMU

The Tensor G5 SoC enforces **IOMMU-based isolation** between the application processor and the
baseband modem. A baseband compromise — historically a significant mobile attack surface —
cannot perform arbitrary DMA into application-processor memory. The modem communicates only over
its defined channels.

<!-- MANUAL INPUT REQUIRED: confirm whether GrapheneOS adds baseband hardening beyond stock Pixel — research against current GrapheneOS documentation (see architecture/research/mobile-threat-model.md §3) -->

## Tailscale — remote access to IRONVEIL

Tailscale provides a mesh overlay for remote access to the IRONVEIL workstation from the handset.
Combined with Termux (in the Façade profile), this supports remote administration and the
dracut-sshd remote-unlock workflow without exposing services on the public internet.

| Capability | Detail |
|------------|--------|
| Overlay | Tailscale mesh (WireGuard-based) between handset and IRONVEIL |
| Use | Remote SSH admin; reach the IRONVEIL initramfs unlock listener |
| Exposure | No inbound ports published to the public internet |

<!-- MANUAL INPUT REQUIRED: record the Tailscale ACL tags scoping the handset's reachable nodes — from the Tailscale admin console ACL config -->

## DNS / egress summary

```
app → RethinkDNS (per-app policy + DNS filtering) → [WireGuard tunnel, per profile] → internet
remote admin → Tailscale overlay → IRONVEIL (SSH / initramfs unlock)
```

## Current state

| Component | Status |
|-----------|--------|
| RethinkDNS per-app firewall | Operational |
| WireGuard (profile-selective) | Operational |
| Sandboxed Google Play (selective profiles) | Operational |
| Baseband IOMMU isolation | Active (hardware-enforced) |
| Tailscale to IRONVEIL | Operational |
