# nxnative tutorial — run a Switch game on your iPhone

[English](README.md) | [中文](README.zh-CN.md)

A step-by-step companion to [nxnative](https://github.com/seedds/nxnative), the project that runs Nintendo Switch games **natively** on Apple platforms: the game's own code executes on the Mac/iPhone, on a host that answers Horizon's system calls and implements the Switch's graphics APIs (NVN, Vulkan, NVIDIA's desktop OpenGL) on Metal.

This walkthrough takes a game you own from NSP files to a playable app on your iPhone — including every pitfall hit on a real install (Mario Kart 8 Deluxe 4.0.1 → iPhone 17 Pro, iOS 27.2), and what fixed each one.

> **Use your own games.** You need NSP files and `prod.keys` dumped from your own console. Nothing here distributes games, keys or firmware. Don't share your `prod.keys` — the build only reads it and never prints its values.

## Which games work?

- **The [games table](https://github.com/seedds/nxnative#readme) in nxnative lists every validated title** — 90+ games, from *Breath of the Wild* and *Tears of the Kingdom* to Mario Kart 8 Deluxe, Pokémon, Metroid Dread, Silksong and Hades II. Each has a tested profile and a note describing what it needed.
- **A game not in the table** still builds: the script surveys its binary (`<title ID>_survey/survey.md`), guesses a provisional profile, and produces a best-effort IPA **that may not boot** — bringing up a new title is real work (see nxnative's `docs/agents`).
- The same flow also works **on the Mac alone**: `--platform mac-app` packs a double-clickable `.app`, no iPhone needed at all.

## Prerequisites

| Tool | Notes |
|---|---|
| [nxnative](https://github.com/seedds/nxnative) cloned | all commands run at the repository root |
| Apple-silicon Mac, Xcode | the Metal compiler must match nxnative's toolchain lock (see its `docs/development.md`) |
| Python 3.12+ | with `lz4`, `capstone`, `fonttools` (and `backports.zstd` before 3.14); `pip install nsz` only if your files are NSZ/XCZ |
| xcodegen, .NET SDK 10, zstd | on `PATH` |
| One game's NSPs | base game, optional update and DLC, in one folder |
| `prod.keys` | from your own console |
| [Sideloadly](https://sideloadly.io) | signs and installs the IPA with an Apple ID |
| Apple ID | a free one works: 3 apps, re-sign every 7 days; a paid account lasts a year |
| iPhone + USB cable | Developer Mode enabled (one-time, see troubleshooting #3) |

**Disk layout tip:** keep the NSPs and the finished IPA wherever you like, but keep the build's work folder (`<title ID>_build/`) on the internal APFS disk. An exFAT external drive makes the shader stage ~10× slower (one `metal` invocation per shader, tens of thousands of small files).

## The 30-second version

```bash
# 1. Build — small IPA + separate game image (mandatory for games over ~4 GB)
python3 scripts/build_ipa_from_nsp.py /path/to/game-nsps -k /path/to/prod.keys \
  --romfs separate -o /path/to/Game.ipa
#    (the build prints the exact push_romfs.sh command at the end)

# 2. Install Game.ipa with Sideloadly (your Apple ID)

# 3. Push the game files into the app's sandbox
xcrun devicectl device info apps --device <UDID>          # find the real bundle ID
bash scripts/push_romfs.sh <UDID> <bundle ID> /path/to/<titleID>_build/extract/<version>/RomFS.img
```

A first build takes 2 minutes (*Metroid Dread*) to 10 (*Breath of the Wild*); a rebuild with unchanged inputs about 20 seconds. On the phone, budget the game's size twice-ish: the IPA (~1 GB) plus the game image (Mario Kart 8 Deluxe: 11.3 GB).

## Step 1 — Build the IPA

```bash
python3 scripts/build_ipa_from_nsp.py /path/to/game-nsps \
  -k /path/to/prod.keys \
  --romfs separate \
  -o /path/to/Game.ipa \
  --progress lines
```

What happens: the NSPs are identified by title ID and matched to the game's profile, extracted, their code modules rewritten where Apple's kernel requires it, the shaders translated to Metal, and everything packed with Xcode. Every intermediate lands under `<title ID>_build/` in the NSP folder; the shader cache makes rebuilds fast and survives interruptions.

Useful options:

| Option | What it does |
|---|---|
| `--romfs separate` | keeps the game files out of the IPA (see below) — always use it for iPhone installs |
| `-p v327680` | pack a specific version (default: newest update in the folder) |
| `-l zh-Hans` | game language; change later without a rebuild via `DM_LANG` in the app's `Documents/env.txt` |
| `--app-name`, `--bundle-id` | home-screen name and bundle ID |
| `--allow-other-metal` | proceed when your Xcode's Metal compiler differs from the tested one (read nxnative's note on issue #242 first) |
| `--platform mac-app` | build a double-clickable Mac `.app` instead of an IPA |
| `--survey-only` | just analyze an unknown game |

**Why `--romfs separate`:** iOS's installer never sees archives beyond ~4 GB (App Store's ceiling), and a bundled IPA of a big game is zip64 with a single 10+ GB file inside — the phone reports `PackageExtractionFailed` every time. Separating gives a ~1 GB IPA plus the raw game image, which step 3 copies straight into the app's sandbox.

## Step 2 — Install with Sideloadly

Open Sideloadly, drop in the IPA, sign with your Apple ID, install. Two things to know:

- **If login fails** with `SSLError(SSLEOFError)` against `gsa.apple.com`: Sideloadly ships an old OpenSSL whose TLS handshake gets reset on some networks (a direct China Unicom line does this deterministically; system `curl` still works, so "curl is fine" proves nothing). Route it through a proxy — enable a system proxy and Sideloadly picks it up. The `Fake2-Anisette download failed` warning is harmless.
- **On iOS 26+/27, Sideloadly appends your team ID to the bundle ID**: `org.nxnative.<titleID>` becomes `org.nxnative.<titleID>.<TEAMID>`. That's the ID step 3 needs.

## Step 3 — Push the game files

The app carries the identity of the image it was built for (`romfs.id`) and only accepts a `Documents/romfs.img` that matches — so install the app first, then push once:

```bash
# find the real (team-suffixed) bundle ID
xcrun devicectl device info apps --device <UDID> | grep nxnative

# push (phone unlocked, Developer Mode on)
bash scripts/push_romfs.sh <UDID> <bundle ID> \
  "/path/to/<titleID>_build/extract/<version>/RomFS.img"
```

`push_romfs.sh` copies the image as one file into the app's data container. An interrupted copy is never used (the app checks the image's identity), so just rerun after a failure.

## Day-to-day

- **Every 7 days** (free Apple ID): the signature expires — reinstall the same IPA with Sideloadly. The game files stay; nothing to re-push.
- **Re-push only when the game or version changes** — a new build of the same version keeps the same image identity.
- **Plays with a controller** paired to the iPhone.

## Troubleshooting

| Symptom | Cause → fix |
|---|---|
| Sideloadly login `SSLError(SSLEOFError)` | old OpenSSL fingerprint reset on-path → enable a system proxy |
| Phone: `PackageExtractionFailed` | IPA is zip64 / >4 GB → rebuild with `--romfs separate` |
| No *Developer Mode* toggle on the phone | hidden until the phone pairs with Xcode once (Window → Devices and Simulators), then Settings → Privacy & Security → Developer Mode → on → reboot |
| Build stops on a Metal compiler `[RED]` line | compiler ≠ tested one → `--allow-other-metal` (or install the matching Xcode) |
| `The Xcode build system has crashed`, repeatedly | build folder moved between disks; stale absolute paths → `rm -rf <titleID>_build/ipa/build/ios/pkg/dd`, rebuild |
| Shader stage crawling (hours) | work folder on exFAT → move `<titleID>_build/` to internal APFS (carry `shader/` along; a resumed build restores finished shaders) |
| `devicectl` device `unavailable` | phone locked → unlock |
| `push_romfs.sh`: app not found | wrong bundle ID → look it up with `device info apps`; Sideloadly added your team ID |
| `devicectl device install app` of Sideloadly's big IPA: file not found | Sideloadly purges its signing cache on quit — not a real path; install the small separate IPA instead |

## The files in this folder

- `README.md` / `README.zh-CN.md` — this tutorial (English / Chinese).
- `scripts/build_ipa_from_nsp.py`, `scripts/push_romfs.sh` — the two nxnative scripts this tutorial drives, copied here **for reading** (MIT, see `scripts/LICENSE`). They resolve paths relative to the nxnative tree, so run them from a clone of [nxnative](https://github.com/seedds/nxnative), not from this folder.
- `LICENSE` — this repository's MIT license.
- `SKILL.md` — the same walkthrough as a Claude Code skill (below).

## Use this as a Claude Code skill

`SKILL.md` turns this tutorial into an agent skill: Claude runs the build, looks up the device and bundle ID, and drives the push.

```bash
git clone https://github.com/leezhong/nxnative-tutorial
ln -s "$(pwd)/nxnative-tutorial" ~/.claude/skills/nxnative-tutorial
```

Then in any session: "把马里奥赛车编译装到手机" / "build and install this Switch game on my iPhone".

---

*Verified end-to-end 2026-11-01: Mario Kart 8 Deluxe 4.0.1 (base 1.0.0 US + update + DLC) → 1004 MB IPA + 11.3 GB image → iPhone 17 Pro, iOS 27.2.*
