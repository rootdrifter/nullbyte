# NULLBYTE MANUAL INPUTS — COMPLETE BEFORE APPLICATIONS

> Single-session resolution guide for the device values still to be confirmed on the nullbyte
> GrapheneOS build (Pixel 10 Pro Fold). Run each step **on the device (NullByte)**, paste the value
> into the matching block, then transcribe it into the referenced README section. Target: one
> ~5-minute pass with the phone in hand.
>
> **Privacy rule:** per-profile app lists stay **private** — never publish them. Only the values
> requested below (boot key hash, chip name, profile *names*) go into the public repo.

---

### 1. Verified boot key hash
**What this enables:** concrete root-of-trust evidence — turns the "verified boot enforced" claim into
an attestable hash a reviewer can compare against the GrapheneOS-published value. **Expected format:**
a hex digest string (from the yellow-state boot screen or `ro.boot.vbmeta.digest`).

On NullByte: **Settings → About phone → (tap Build number / verified-boot screen)** for the key
hash, OR over ADB:
```
adb shell getprop ro.boot.vbmeta.digest
```
Paste hash below:
```
[PASTE HERE]
```
Update: `README.md` — **Security Architecture → Verified Boot** section (the
"Titan M2 root of trust → bootloader → verified OS image" paragraph, around line 52–55). Add the
captured key hash as the concrete root-of-trust evidence.

---

### 2. Secure element name confirmation
**What this enables:** an accuracy guard — confirms whether "Titan M2" is correct for the Tensor G5
generation. A wrong chip name is a credibility hit in front of a hardware-literate reviewer, so
verifying it protects the whole hardware-security claim. **Expected format:** the exact chip string
the device reports (e.g. `Titan M2`, or its named successor).

On NullByte: **Settings → About phone** — check the exact security-chip name for the Tensor G5 / Pixel
10 Pro Fold generation. (Open question: is it still **"Titan M2"** or a successor on this hardware?)
Paste exact string below:
```
[PASTE HERE]
```
Update: `README.md` — **Hardware** table, **Security chip** row (currently `Titan M2`, line 35) and
the verified-boot references that name it (lines 40–43, 55, 63). If the device reports a different
part, correct **every** occurrence and the docs/index.html + CV references too.

> Accuracy note: CLAUDE.md §0.3 flags this as unverified for the Tensor G5 era — do not assume
> "Titan M2"; transcribe exactly what the device reports.

---

### 3. Profile list confirmation
**What this enables:** confirms the headline "nine compartmentalised profiles" architecture matches the
device exactly (count + names) — the central claim of the build. **Expected format:** a list of nine
profile names matching Nexus, Plague, Ghost, Abyss, Void, Façade, Shade, Vault, Joker.

On NullByte: **Settings → System → Multiple users**. List all profile names exactly as shown:
```
[PASTE HERE]
```
Expected (nine — already documented in `architecture/operational-profiles.md` §Profiles):
**Owner (Nexus), Plague, Ghost, Abyss, Void, Façade, Shade, Vault, Joker.**
Note any discrepancy (count ≠ 9, a renamed/missing profile, or an extra one).

Update if anything differs: `README.md` (the "nine compartmentalised profiles" claim, lines 3–4 and
§Operational Profile Architecture line 106+) and `architecture/operational-profiles.md` §Profiles
(the nine-row table, lines 20–28). These are the clean **public** codenames — confirm the count and
names match; **do not** add the private per-profile app inventories.

---

## Resolution checklist — RESOLVED 2026-06-11 (operator device session)
- [x] Verified-boot key hash captured → README §Verified Boot + grapheneos-setup.md updated
      (`deadbeef…eadbeef`)
- [x] Secure-element name confirmed exactly → **Titan M2** confirmed; full stack documented
      (Tensor G5 security core + Titan M2 + Trusty TEE) in README + docs
- [x] Nine profiles confirmed present & named → all nine on device (Settings → System → Users);
      README + operational-profiles.md + docs cross-checked
- [x] **Codename correction:** the device shows **Ghost** as the third profile, correcting the
      earlier codename; the old name was swept to **Ghost** across all nullbyte files (and the
      rootdrifter-hub copies) — none of the old codename remains anywhere
- [x] Device details: Android 16, build 2026060601 ingested
- [x] Re-run the privacy scan (no app lists, no demonology codenames, no IMEI), commit, push

### Still pending (no value provided this session)
- Exact GrapheneOS release-tag string and security patch level.
- Tensor G5 kernel version.
- Per-profile lockscreen policy and WireGuard-vs-cleartext routing (RethinkDNS per-profile config).
- IOMMU/baseband research item (README) — research against current GrapheneOS docs, not a device value.
- Per-profile app inventories stay **private** by policy — never publish.

### Deliberately NOT published (privacy)
- **IMEI** — never recorded in any file (operator constraint + standing rule).
- **Tailscale IP** (`100.*` overlay address) — operational value, not requested for insertion;
  omitted as a private network detail.
