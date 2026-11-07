---
name: nxnative-tutorial
description: Run a Switch game on the user's iPhone or Mac with the nxnative project — build the IPA (build_ipa_from_nsp.py --romfs separate), install it with Sideloadly, look up the real bundle ID with devicectl, push the RomFS image with push_romfs.sh. Switch 2 games build from a .nup or a dump folder with no prod.keys (Mac-verified). Trigger words: nxnative, compile switch game, install switch game on iPhone, port to iPhone, build IPA, re-sign, push romfs, switch 2 game, nup, game dump, 编译 switch 游戏, 把 switch 游戏装到手机, 移植到 iPhone, 推镜像, 重签, switch 2 游戏, 游戏dump.
---

# nxnative: Switch game → iPhone (build, sign, install)

Compile one Switch game's NSPs into an IPA with the nxnative repository, install it on the user's iPhone with Sideloadly, and push the game files into the app's sandbox. Verified end-to-end 2026-11-01 on Mario Kart 8 Deluxe 4.0.1 → iPhone 17 Pro (iOS 27.2); works the same for every game in nxnative's `dm/games/README.md` games table. Switch 2 games: see the section at the end.

## Switch 2 games (.nup or a dump)

A Switch 2 title (program ID `0400…`) builds from **one `.nup` file or a folder of the game's files** (`exefs/` + `romfs/` = the base game) — **not NSPs, and no `-k`/`prod.keys` at all**:

```bash
python3 scripts/build_ipa_from_nsp.py <dump folder> --romfs separate -o <output>.ipa
```

- Everything downstream is the same script; the survey/provisional-profile flow covers unknown titles.
- An **update dump** (e.g. Mario Kart World 1.8.0) packs from a profile *derived* from the base game's — its fixes are build-gated, so boot and play the update on the Mac first. Verified 2026-11-07: Mario Kart World 1.8.0 to a full race on an M1 Pro (first run anywhere; SDK 21.4.0, NVN2 API 231.39, Ampere/sm86 shaders, 7.47 GiB heap ask).
- **All Switch 2 titles are Mac-verified only** — validate on the Mac before any iPhone attempt (iPhone memory limits are open).
- Playing on the Mac: launch the host with `DM_WINDOW=1` **and `DM_AUDIO=1`** (Mac defaults to a silent sink); keyboard WASD/J-K-U-I/Q-E/arrows; save dir on internal APFS (`clonefile`).
- Dumps travel as multi-part RARs, often RAR6 `v6:m1` which **7zz leaves as 0-byte files** — extract with RARLAB `unrar` 7.2+ and check sizes before building.

**Repository**: a clone of nxnative (https://github.com/seedds/nxnative); run every command at its root.
**Device**: the user's iPhone over USB; find the UDID once with `xcrun devicectl list devices`.

## Step 0: pre-flight (all required)

1. **Proxy**: Sideloadly's Apple ID login (gsa.apple.com) dies with `SSLError(SSLEOFError)` on some networks — it bundles an old OpenSSL whose handshake gets reset on-path (deterministic on a direct China Unicom line; system `curl` still works, so that proves nothing). Have the user enable a **system proxy** before any Sideloadly step.
2. **Developer Mode** on the phone (Settings → Privacy & Security). No toggle visible = the phone has never paired with Xcode: Xcode → Window → Devices and Simulators with the phone plugged in, once. Then toggle on, reboot. One-time.
3. **Phone unlocked**: devicectl reports the device `unavailable` while locked.
4. **Work folder on internal APFS**: `<NSP folder>/<title ID>_build/` must not sit on an exFAT external disk — the shader stage drops ~10× (tens of thousands of `metal` invocations over small files). If the NSPs live on such a disk, keep `shader/` and the build folder on the internal disk and symlink into the NSP folder. The per-shader cache survives interruptions.
5. **Is the game supported?** Check `dm/games/README.md`. In the table → build directly. Not in it → the script writes a survey and guesses a provisional profile, and the IPA is best-effort, **may not boot** — tell the user before building.
6. `prod.keys` is only ever a `-k` path. **Never print its contents.**

## Step 1: build

```bash
python3 scripts/build_ipa_from_nsp.py <NSP folder> \
  -k <NSP folder>/prod.keys \
  --romfs separate \
  -o <output>.ipa \
  --progress lines
```

- `--romfs separate` always for iPhone installs: a bundled IPA over ~4 GB is zip64 and the phone rejects it (`PackageExtractionFailed`, every time). Separated: ~1 GB IPA + the image at `<title ID>_build/extract/<version>/RomFS.img`. The build prints the exact push command at the end.
- `[RED]` Metal compiler version gate → add `--allow-other-metal` (the local Xcode differs from the tested one; mention nxnative issue #242).
- `The Xcode build system has crashed`, repeatedly → the build folder was moved/copied across disks; `rm -rf <title ID>_build/ipa/build/ios/pkg/dd` and rerun (the app pack itself is ~20 s).
- Version: `-p v<vnumber>`; default is the newest update NSP in the folder. A new game version needs a rebuild **and** a re-push (the image identity changes).
- No iPhone at all? `--platform mac-app` builds a double-clickable Mac app instead — offer this when the user just wants to play on the Mac.

## Step 2: Sideloadly install (user does this — GUI)

Tell the user: open Sideloadly, drop in the IPA, sign with their Apple ID (a free one works), install. Confirm the system proxy is on (step 0.1). The `Fake2-Anisette download failed` warning is harmless.

## Step 3: look up the real bundle ID

Sideloadly on iOS 26+/27 appends the team ID to the bundle ID (`org.nxnative.<titleID>` → `org.nxnative.<titleID>.<TEAMID>`); pushing with the original ID finds no app:

```bash
xcrun devicectl device info apps --device <UDID> | grep nxnative
```

## Step 4: push the game files

```bash
bash scripts/push_romfs.sh <UDID> <bundle ID from step 3> \
  "<NSP folder>/<title ID>_build/extract/<version>/RomFS.img"
```

Success prints an identity line (`DMROMFSID1 … index=…`). The app only accepts an image with the identity it was built for: **re-push only when the game or its version changed**, never after a reinstall or re-sign.

## Aftercare

- **7-day re-sign** (free Apple ID): when the app stops opening, the user reinstalls the same IPA with Sideloadly (proxy on). Game files untouched.
- A failed/interrupted push is never used (the app checks image identity) — just rerun the script.
- Don't chase `devicectl device install app` on Sideloadly's big signed IPA: Sideloadly deletes its signing cache on quit. The separate-IPA flow is the path.

## Pitfall quick reference

| Symptom | Cause → fix |
|---|---|
| Sideloadly login `SSLError(SSLEOFError)` | old OpenSSL fingerprint reset on-path → system proxy on |
| Phone `PackageExtractionFailed` | zip64 IPA over 4 GB → rebuild with `--romfs separate` |
| No Developer Mode toggle | pair with Xcode once, then the toggle appears |
| Shader stage crawling for hours | build folder on exFAT → move to internal APFS (carry `shader/`) |
| xcodebuild crashing repeatedly | stale `pkg/dd` after a disk move → delete it, rebuild |
| devicectl device `unavailable` | phone locked → unlock |
| Push: app not found | bundle ID got the team ID appended → `device info apps` |
| devicectl install of Sideloadly's cache: no such file | cache purged on quit → use the separate IPA flow |
| Switch 2 dump: 0-byte files after 7zz | RAR6 `v6:m1` → extract with RARLAB `unrar` 7.2+ |
| Mac run: no sound | Mac host defaults silent → launch with `DM_AUDIO=1` |
| Mac run: saves read-only (`clonefile` RED) | save dir on exFAT → put it on internal APFS |
