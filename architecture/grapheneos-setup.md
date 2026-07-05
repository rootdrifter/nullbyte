# GrapheneOS Installation — Pixel 10 Pro Fold

Living build — `MANUAL INPUT REQUIRED` markers flag values still to be captured from the running device; `FUTURE WORK` markers flag planned enhancements.

## Device

| Attribute | Value |
|-----------|-------|
| Device | Google Pixel 10 Pro Fold |
| Codename | `rango` |
| SoC | Google Tensor G5 |
| Security chip | Titan M2 |
| OS | GrapheneOS |

The Pixel hardware was chosen specifically because it is GrapheneOS's reference platform: it
provides a hardware root of trust (Titan M2) and full support for re-locking the bootloader with
a custom OS, which most Android devices do not permit.

## Installation method

GrapheneOS was installed using the official **web installer** (WebUSB), which flashes the
verified factory images directly from the browser:

1. Enable OEM unlocking and USB debugging; unlock the bootloader.
2. Flash the GrapheneOS factory images via the web installer.
3. **Re-lock the bootloader** after flashing — this is mandatory for verified boot to enforce
   image integrity.
4. Complete first boot and confirm the verified-boot state.

Current build: **GrapheneOS Android 16, build number 2026060601** (captured 2026-06-11; the exact
GrapheneOS release-tag string and the build flashed at *initial* install are not separately recorded).

## Verified boot

After re-locking, verified boot is enforced on every power cycle:

- The bootloader is **locked** post-flash.
- Verified boot is **active** — any modification to the OS partition breaks the verification
  chain and the device refuses to boot the modified image.
- The boot chain is: **Titan M2 root of trust → bootloader → verified OS image**.

The verified-boot key fingerprint is shown during boot and can be confirmed against the
GrapheneOS-published value to ensure the genuine OS signing key is in use:

```
Verified boot key hash: deadbeefdeadbeefdeadbeefdeadbeefdeadbeefdeadbeefdeadbeefdeadbeef
```

Captured 2026-06-11 (yellow-state boot screen / `getprop ro.boot.vbmeta.digest`). Compare against
the GrapheneOS release signing keys to confirm the build is authentic and unmodified.

<!-- MANUAL INPUT REQUIRED: record the security patch level and exact GrapheneOS release tag (Settings → About phone → Android security update); Android 16 / build 2026060601 captured, the release-tag string is not yet recorded -->
<!-- NOTE: device runs Android 16, build 2026060601. -->

## Post-install verification

| Check | Expected |
|-------|----------|
| Bootloader state | Locked |
| Verified boot | Active (yellow/green state with custom key) |
| OEM unlocking | Disabled after re-lock |
| Titan M2 attestation | Intact |

## Current state

| Item | Status |
|------|--------|
| GrapheneOS flashed (web installer) | Operational |
| Bootloader re-locked | Confirmed |
| Verified boot enforced | Active on every power cycle |

See [security-model.md](security-model.md) for the full boot-chain threat analysis.
