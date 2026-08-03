# CachyOS Kernel — Kingston KC3000 / FURY Renegade Overheating Fix

Personal fork of the [CachyOS kernel](https://github.com/CachyOS/linux-cachyos), built for my own daily use. Includes a patch that fixes the Kingston KC3000 / FURY Renegade running hot at all times — regardless of load — by disabling FUA (Force Unit Access) for that specific device.

**Just here for the fix?** Jump to [Just want the patch](#just-want-the-patch). You don't need my whole build.

> ⚠️ This repo is opinionated — PKGBUILD options and config are tuned for my own system. Read the PKGBUILD if you want the details; this README only covers the KC3000/FUA patch.

---

## The problem

Some KC3000/Renegade owners see the drive run hot constantly, idle or under load. Mine sat around **~75°C no matter what it was doing**. Known issue, reported here: https://bugs.launchpad.net/ubuntu/+source/nvme-cli/+bug/2064042 — full credit to that report, I only ported the fix to a current kernel.

Working theory (unconfirmed, just observed): a FUA-flagged write locks the drive's firmware into a high power state and it never comes back down. Disabling FUA avoids that.

## The patch

Adds an `NVME_QUIRK_NO_FUA` quirk and applies it to the KC3000/Renegade PCI ID (`0x2646:0x5013`). With it set, the kernel stops tagging writes as FUA and falls back to regular cache-flush commands instead. Three files touched, small and self-contained diff.

**Result:** ~38°C idle, ~60°C max under load — instead of a constant ~75°C.

## ⚠️ Before you apply it: what you're trading away

FUA lets a drive confirm a *specific write* is durably stored without a full cache flush. It matters for things that need to survive a sudden power loss: filesystem journals, database `fsync()`/`fdatasync()`, etc.

Disabling it doesn't remove durability — the kernel still flushes the cache, just less granularly. Normal use and clean shutdowns are unaffected. **The risk is unclean power loss** (power cut, crash, dead laptop battery) while data sits in the drive's cache:

- Consumer SSDs like the KC3000 don't have power-loss-protection capacitors anyway, so this trades away some of an already-limited safety margin — not a huge amount, but not nothing.
- **Laptop with a healthy battery, or desktop on a UPS:** low risk, reasonable tradeoff for most people.
- **Desktop with no UPS, unstable power, or write-heavy workloads** (databases, NAS, metadata-heavy ZFS/btrfs): be more careful — consider a UPS, or skip this patch on that machine.
- I'm not a storage engineer and don't know Kingston's firmware internals — this is a community workaround, not a vendor fix.

**In short:** small durability tradeoff around unclean power loss, in exchange for a big temperature drop. Your call.

---

## Just want the patch?

Recommended for most people — build it on the **official** CachyOS repo instead of mine:

1. Clone the official `linux-cachyos` repo and pick your variant (e.g. `-lts`, `-bore`).
2. Put the patch file in that variant's folder, next to its `PKGBUILD`.
3. Add the patch filename to the PKGBUILD's `source=()` list.
4. Run `updpkgsums`.
5. `makepkg` as usual.

**⚠️ You may need to re-port it yourself.** I don't update this fork on a schedule, and CachyOS's tree moves fast enough that `core.c`, `nvme.h`, and `pci.c` can shift and break the patch context. If it fails to apply: open the patch and target files side by side, find the same three spots (FUA feature block in `core.c`, quirks enum in `nvme.h`, KC3000 PCI entry in `pci.c`), and reapply by hand — it's small enough to redo in a few minutes.

**⚠️ A patch that applies cleanly is not the same as a patch that's still correct — check it before you boot it, this touches your storage driver and your data.** A clean apply only means the surrounding code lines still matched; it says nothing about whether the logic is still doing what it's supposed to. Before trusting it on a real system:
- Re-read the applied diff against the current source and confirm it's still sitting in the right place — the FUA/write-cache block in `core.c`, and that the `NVME_QUIRK_NO_FUA` bit (`1 << 23`) hasn't since been claimed by a newer upstream quirk (a silent bit collision would corrupt more than just this one quirk).
- After building, confirm the fix actually took effect at runtime, not just that the build succeeded — a lower temperature is a good sign, but on its own it doesn't prove the quirk applied the way it's supposed to.
- If you're not confident reading the diff yourself, don't apply it blind — ask someone who can, or wait for a more current port. This is a block-layer driver sitting directly between your OS and your data; a mistake here is a different order of risk than a userspace tool.

## Using my exact build

You're welcome to, but it's opinionated — PKGBUILD/config choices are mine, not documented here. Read the PKGBUILD for details. Most people are better off with the patch-only route above.

## Applies to

- Kingston KC3000 (PCI ID `2646:5013`)
- Kingston FURY Renegade (same controller, rebadged)

Different drive or different symptom? This patch probably isn't your fix — check `pci.c` for an existing quirk entry first.

## Credit

Fix originally from: https://bugs.launchpad.net/ubuntu/+source/nvme-cli/+bug/2064042
I ported it to a current kernel as an `NVME_QUIRK_NO_FUA` quirk. No guarantees beyond "works on my machine."

## Disclaimer

No warranty. Hobbyist patch based on observed behavior, not a root-cause fix or vendor statement. Understand the FUA tradeoff above, keep backups regardless of your SSD or kernel.
