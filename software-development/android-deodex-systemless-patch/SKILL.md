---
name: android-deodex-systemless-patch
description: Patch an odex-era Android framework on an owned device.
version: 1.0.0
license: MIT
metadata:
  hermes:
    tags: [android, odex, oat, smali, baksmali, magisk, deodex, framework, rom-patching]
---

# Android deodex → smali patch → systemless deploy

Patch a vendor's prebuilt Android framework (the odex/oat era: Nougat/Eclair
to Oreo, API ≤ 27) without touching /system: extract the ROM, deodex the
target, edit smali, rebuild, compile the odex on-device, ship as a Magisk
module. Own hardware only; keep a pinned boot-image rollback before flashing
anything.

## Workflow

1. **Get the exact /system contents offline.** Full OTA zip → `system.new.dat`
   + `system.transfer.list` → raw ext4 via sdat2img → `debugfs -R 'rdump /framework dir' image`.
   Read with debugfs; mounting needs root and loop devices you may not have.
2. **Check whether the jar is a stub before planning a deodex.** Vendor ROMs
   often ship `/system/framework/*.jar` as 304-byte MANIFEST-only ZIPs — ALL
   code lives in `oat/<arch>/<name>.odex`. A tiny jar means deodex is
   mandatory; there is no shortcut via the jar's classes.dex.
3. **Pick the deodex tool by trying it, not by lore.** `baksmali help deodex`
   must print a deodex subcommand — recent baksmali builds dropped odex
   support entirely. smali 2.5.2-dev (Arch `/usr/share/java/smali`) deodexes
   Nougat arm64 cleanly. SmaliEx oat2dex 0.90 dies on Nougat boot.oat with
   `UnresolvedOdexInstruction` → `Instruction35ms` ClassCastException (its
   known issue #61) — don't burn time on it for Nougat.
4. **Deodex with the boot images on the classpath:**
   `baksmali deodex -a <api> -d <dir-containing-boot*.oat> -o out oat/<arch>/target.odex`.
   `-d` must point at the ARCH dir (e.g. framework/arm64) — baksmali looks up
   `boot.oat` inside it. Success = 0 files containing `-quick` opcodes.
5. **Find the gate by its log string.** grep the smali tree for the exact
   denial message from logcat; then map the whole method AND its callers
   before patching. Audit sibling branches first: vendor gates often have a
   build-type check, a factory/dev `ro.boot.*` prop, or a boolean field
   bypass that needs NO patch at all. Remember `Build.TYPE`/`TAGS` come from
   /system/build.prop — a userdebug BOOT image does not change them.
6. **Patch the conditional, mind the semantics.** Inverting `if-eqz`↔`if-nez`
   flips both outcomes: foreign keys pass AND vendor-keyed installs become
   the denied side. If vendor-signed installs must keep working, overwrite
   the result (`const/4 v0, 0x1` after move-result) instead of inverting.
7. **Verify the round-trip before packaging:** `smali a -a <api>` the tree
   back to a dex, re-disassemble with `baksmali d`, and grep that the edited
   branch is present exactly once. Count methods first (`grep -rh '^\.method'`):
   < 65k stays a single dex; more needs a multi-dex-aware smali.
8. **The odex must come from the device's own dex2oat at the REAL path.**
   odex headers record dex location + checksums; a host-compiled or
   /data/local/tmp-compiled odex is treated as stale. On-device recipe:
   bind-mount the new jar over /system/framework/<name>.jar (read-only), run
   the device's `/system/bin/dex2oat` with `--dex-location=/system/framework/<name>.jar`,
   `--instruction-set=<primary-arch>`, `--boot-image=/system/framework/boot.art`,
   write the odex to /data/local/tmp, unmount. /system is never written.
   Sanity-check with the device's `oatdump --oat-file=` for a known symbol.
9. **Deploy systemless:** Magisk module dir with `module.prop` +
   `system/framework/<name>.jar` + `system/framework/oat/<arch>/<name>.odex`,
   zip, `adb shell magisk --install-module`, reboot. Stacked rollbacks:
   disable module in Magisk → Android safe mode (skips modules) → pinned
   stock boot image via fastboot.
10. **Prove it end-to-end with a foreign-signed APK** that the gate denied
    before; ground truth = `pm path <pkg>`, plus `logcat -d -s PackageManager`
    for the gate's own ladder logs.

## Pitfalls

- **sdat2img v4 transfer-list format:** the command word and the range-count
  are SPACE-separated, ranges comma-separated (`new 2,0,1024` = count 2,
  pair 0–1024). Splitting on commas alone parses nothing and yields a 0-block
  image.
- **Pin every deploy script to the device serial** with shell wrappers
  `adb() { command adb -s "$SER" "$@"; }` and `fastboot() { command fastboot
  -s "$SER" "$@"; }` defined before first use — bare `fastboot flash` with
  multiple devices attached errors out at best. Device-DETECTION loops must
  grep the full list (`command fastboot devices`), not the pinned wrapper.
- Only replace the odex for the arch the device actually runs (primary ABI —
  check `ro.product.cpu.abi`); a 64-bit system_server never reads oat/arm.
- Keep `CLASSPATH`/`ANDROID_ROOT`/`ANDROID_DATA` env set when invoking
  dex2oat over adb, and capture its stderr to a file — on RAM-tight devices
  it can OOM; retry with `--compiler-filter=verify-at-runtime`.
- A stale odex beside the new jar makes ART fall back to interpreting the jar
  (slow boot) or ignore your patch — always ship jar + odex from the SAME
  build in one module.
- Never patch odex-era framework for a device you don't own, and never ship
  the patched artifacts (they embed the vendor's keys) — disclosure rules of
  the paired-device-testing skill apply.
