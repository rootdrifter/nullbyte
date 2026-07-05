# Security Model

Living build — `MANUAL INPUT REQUIRED` markers flag values still to be captured from the running device; `FUTURE WORK` markers flag planned enhancements.

## Threat model

A hardened mobile platform must resist a broader set of adversaries than a desktop, because the
device travels and is frequently in untrusted physical environments.

| Adversary | Capability | Primary mitigation |
|-----------|-----------|--------------------|
| Physical seizure (theft, border, duress) | Possession of the device, possibly powered on | Per-profile encryption; locked profiles cryptographically inaccessible |
| Coerced unlock of one profile | One profile credential under compulsion | Compartmentalisation — other profiles stay encrypted |
| Malicious / trojanised app | Code execution within a profile | Profile boundary contains the blast radius; expendable junk profiles |
| Network adversary | Traffic observation / injection | WireGuard egress; RethinkDNS per-app policy |
| Baseband / modem exploit | Compromise of the radio stack | Tensor G5 IOMMU isolation from the application processor |
| OS tampering / evil-maid | Offline modification of the OS partition | Verified boot enforced by Titan M2 root of trust |
| Tracking / profiling | Cross-app and cross-profile correlation | No PII across profiles; sandboxed (not privileged) Play |

## Titan M2 verified-boot chain

```
Titan M2 (hardware root of trust)
        │  attestation + key storage, independent of the SoC
        ▼
Bootloader (re-locked after flashing)
        │  verifies the signature of the OS image
        ▼
Verified GrapheneOS image
        │  any modification breaks the chain → device refuses to boot
        ▼
Per-profile encrypted userspace
```

The Titan M2 handles key storage, attestation, and tamper detection independently of the
Tensor G5. An attacker who compromises the SoC cannot forge the attestation chain without the
Titan M2. See [grapheneos-setup.md](grapheneos-setup.md) for the install and re-lock steps.

## Per-profile encryption

Each profile's data is encrypted under a key derived from that profile's lockscreen credential;
the owner key is additionally bound to the Titan M2. Locking a profile evicts its keys. This is
the cryptographic basis for the compartmentalisation described in
[operational-profiles.md](operational-profiles.md).

## Physical-attack mitigations

| Attack | Mitigation |
|--------|------------|
| Offline OS modification (evil-maid) | Verified boot + locked bootloader |
| Cold extraction of a powered-off device | Per-profile encryption at rest |
| Coerced single-profile unlock | Other profiles remain encrypted (plausible compartmentation) |
| Hardware tamper | Titan M2 tamper detection; IP68 sealed chassis |

## Operational-security principles

- **One identity per profile.** Operational identities never share a profile.
- **Least privilege per profile.** Sandboxed Play and network access are granted only where a
  specific app requires them.
- **Expendability by design.** Junk/throwaway profiles can be destroyed without loss.
- **No cross-profile linkage.** No shared clipboard, contacts, or accounts by default; no PII
  crosses boundaries.
- **Separation of sensitive data from broad access.** The pentesting profile (broadest network
  reach) holds no financial or professional data.

<!-- MANUAL INPUT REQUIRED: record the GrapheneOS version / security patch level — Settings → About phone → Build number / Android security update -->
<!-- FUTURE WORK: add an inter-profile routing diagram showing which profiles egress via WireGuard vs cleartext -->
