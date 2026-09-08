# kansas-modem-unlock

Plug-in device framework (flash → root → carrier unlock) with interactive
guidance. Ships with **one verified device** (below); other devices plug in
as data files only after hardware-proven verification (see
`devices/README.md`). Nothing here works on a device it wasn't proven on —
refusal is a feature. A confirmed second-device port (nevada, Moto G
Play 2026, locked→LOADED on the lab unit) lives in `NEVADA.md`.
run wizard.py --experimental thats the auto installer

## The 5-minute path

```bash
cp config.json mylab.json   # point firmware_files at YOUR images
python wizard.py --dry-run --work work-wizard     # walk everything, touch nothing
python wizard.py --work work-wizard               # detect -> flash -> root -> unlock
python wizard.py --work work-wizard --phases verify  # health re-check anytime
# nevada, hands-free past rooting (confirmed port, see NEVADA.md):
python postroot.py --config mylab-nevada.json --work work-nevada --experimental --patches nevada_table.json
```

State resumes from `work-wizard/state.json`; `--phases unlock` (or any
subset) runs just that slice. Every run starts with `setup`: environment
sweep, device + RSA-authorization polling, battery and disk gates.

## Hands-free vs hands-on (what the CLI does vs what you do)

Automated: env checks, RSA wait loops, APK installs (from your configured
copies), opening apps/screens for you, reboot waits, backups, builds,
flashes, health checks, photo/receipt bookkeeping.

You, when asked with exact steps: enable Developer options + USB debugging,
tap Allow on the RSA prompt, in-app taps the tool cannot script (it opens
the screen and verifies the result), cable tricks for fastboot, photos of
screens (screenshots are broken on some builds), typed confirmations.
Nothing advances on assumption: every handoff verifies before continuing. Every destructive step needs typed
confirmation; `--dry-run` performs the whole ceremony against logs only.

## Nevada end-to-end: zero to flashed (step by step)

Every command, in order, for the confirmed nevada port (Moto G Play 2026
XT2615V). Stop at any refusal — refusal means something isn't proven safe
yet, not that you should push harder.

**0. What you need on the table.** Python 3.10+, `adb` + `fastboot`
(`python bootstrap.py --yes` fetches official platform-tools if missing),
YOUR factory firmware folder (RETUS package with `md1img.img`), a bootloader
already unlocked, Magisk or KernelSU installed with root working, one steady
USB cable, and 30%+ charge (flashing refuses below 30% unless you explicitly
own the risk). ~30 unhurried minutes.

```bash
git clone -b nevada-port https://github.com/crabcakes97/kansas-modem-unlock
cd kansas-modem-unlock
```

**1. Plug in, authorize, check the phone is really there.**

```bash
adb devices                        # exactly 1 device, status "device"
```

On the phone (first time only): tap Build number 7x → Developer options →
USB debugging ON → plug in → tap Allow + Always-allow on the RSA prompt.

**2. One-time root grant (the only tap the tool cannot do for you).**

```bash
adb shell 'su -c id'               # expect: uid=0(root) ...
```

If it says Permission denied: on the phone open Magisk/KernelSU → Superuser
→ allow Shell (or set auto-allow), then re-run. Nothing proceeds without
`uid=0` — re-run until you see it.

**3. Point the config at YOUR factory image.** Edit `mylab-nevada.json`:

```json
"firmware_files": { "md1img_stock": "/path/to/your/RETUS/md1img.img", ... }
```

Then read-only recon (touches nothing):

```bash
python unlock.py status
```

Expect: model `moto g play - 2026`, sku `XT2615V`, your baseband string,
`root : OK (su -> uid 0)`, fingerprint `model=False sku=False baseband=False`
with the kansas REFUSE notice (correct — kansas bytes never touch nevada).

**4. Audit your factory image (intact-factory proof).**

```bash
python -c "import imageaudit; print(imageaudit.render(imageaudit.audit_image('/path/to/your/RETUS/md1img.img')[1]))"
```

Expect: `entries=23 layout_ok=True integrity=True`, all three triples
`hash_ok=True`. Anything else: stop, your file is corrupt or wrong.

**5. Run the hands-free flow (backup + build, NO flash).**

```bash
python postroot.py --config mylab-nevada.json --work work-nevada \
  --experimental --patches nevada_table.json
```

What it does, with zero prompts: fingerprint report → battery report →
build (stock audit + 8 old-byte gates + byte-exact diff + CERT2 re-sign +
signed re-parse) → root re-check → live modem backup pull (~200MB, a few
minutes) → stops and prints the flash command. Expected tail:
`build OK`, `root: OK`, `backup done`, `done. Nothing flashed`.

**6. Prove the backup matches factory (your revert must be real).**

```bash
python -c "
import hashlib, glob
b = sorted(glob.glob('work-nevada/backups/md1img_a_backup_*.img'))[-1]
h = hashlib.sha256(open(b,'rb').read()[:76617232]).hexdigest()
print('MATCH' if h == '3383f93c2d1be9d36b3e291a67415c6c13345fe06843e8a95720e017021d3ba5' else 'MISMATCH: STOP')"
```

MISMATCH means the live modem isn't stock — stop and investigate before
anything destructive.

**7. Charge to 30%+.** Check with `adb shell dumpsys battery | grep level`.
Below 30%, flashing refuses unless you add `--charge-anyway` (a mid-flash
power loss is how modems get bricked — the override is your risk, logged).

**8. Flash (the one destructive step).** Preflight re-verifies: unlocked
bootloader, backup present, signed image present. Slot A only; slot B and
preloader/lk/gpt/efuse/nvram are never named.

```bash
python postroot.py --config mylab-nevada.json --work work-nevada \
  --experimental --patches nevada_table.json --flash
```

Expect: `Sending 'md1img_a' ... OKAY`, `Writing 'md1img_a' ... OKAY`,
reboot, ~90 s wait, then verify output.

**9. Verify (non-optional).**

```bash
python unlock.py verify
```

Acceptance, all four: baseband string alive, `sim state: LOADED`,
`remain ... [5]` (untouched), `modem EE count: 0`. If SIM still shows
`NETWORK_LOCKED`, the lock is asserting on a path this build doesn't cover —
revert (next step) and report; do not re-flash hoping.

**10. Way back (any time, tested path home).**

```bash
python unlock.py revert --backup work-nevada/backups
python unlock.py verify
```

**11. After.** A modem OTA silently restores the stock lock — re-check SIM
state after any update. Never `fastboot -w`, never erase, never flash slot B.

## Architecture

- `wizard.py` — interactive runner + device/session layer (Ux),
  state/resume, profile detection and gating. No patch bytes inside.
- `devices/<id>.json` — the entire device definition: fingerprint gates,
  never-flash list, flash plan (user-supplied images + magic checks),
  root recipe, modem patch table, verify acceptance values. Schema v1.
- `flash_mod.py` — denylist (profile + global sacred list), magic checks,
  sha pinning, per-partition typed confirms, post-flash health check.
- `root_mod.py` — guided root with machine-verified gates around a human
  middle (manager app + on-device patch); re-baselines once su passes.
- `unlock.py` + `patches.py` — the proven modem flow, reused by the wizard
  behind a drift guard (profile table must equal patches.py or refuse).
- `sign_mtk_cert.py`, `parse_mtk_certs.py` — vendored re-sign helpers.
- `config.json` — local paths + strictness (never committed with contents).

## Your own key (bootloader phase)

`python unlock.py bootloader` guides an official-style unlock: it shows
get_unlock_data output, you fetch YOUR key from the vendor portal, and
paste it at a hidden prompt. The key travels straight into the one
fastboot command (never shell), is never printed, logged, or written
anywhere, and is wiped from memory after. Plain truth: unlocking usually
factory-resets the phone (vendor behavior) - back up first - and this tool
performs no bootloader patching of its own.

## Experimental mode (red banner, your risk)

`--experimental` enables:

- Untested profiles in the wizard: detection still runs, but flashing
  proceeds only with YOUR patch table (--patches TABLE.json). The tool
  ships zero offsets for hardware it has not proven.
- Custom modem firmware: audit the factory image (stored vs recomputed
digests must match = intact-image proof), apply your table with old-byte
gates, then the byte-exact audit - every differing byte must sit inside
a declared entry with exact old-to-new content, or the build is refused
(tested: merged runs, wrong-old-byte, hostile-extra-byte).

Byte-for-byte means: built equals stock everywhere EXCEPT the declared
set - verified by diff, not by trust. imageaudit.py implements both gates
with stdlib only; RSA checks run where the vendored verifier supports
them, hash consistency is load-bearing either way.

---

## Fresh machine: CachyOS installer (`setup-cachyos.sh`)

Reproduces this exact toolchain on CachyOS/Arch: pi trio pinned 0.85.1
(server+client included — background subagents fail without them),
pi-subagents 0.65.1 via the `~/.pi/agent/npm` project shape, android-tools
(adb/fastboot, official repos, on PATH by install), node/npm, base-devel,
git, and the python sim packages. Then run `pi auth` yourself — auth,
sessions, and keys are never carried over, by design:

```bash
./setup-cachyos.sh [--yes] [--no-upgrade] [--with-opencode] [--with-java] [--no-python-deps]
```

## Production setup (no manual tool installs)

`python bootstrap.py --yes` fetches official Google platform-tools
(~15-50MB, HTTPS-only dl.google.com) into `vendor/` (git-ignored, never
committed) and verifies them three ways: zip integrity, both binaries
present, `adb version` / `fastboot --version` execute. The wizard offers
the same download automatically on first run (`setup` phase) unless
`tools.auto_download` is off. Strict setups pin an exact URL + sha256
(`config tools.pinned` or `bootstrap.py --url/--sha256`); note Google
rotates the `-latest` zips per release, so there is no stable vendor
checksum to pin against — HTTPS + structure + execution is the check,
stated plainly.

## GUI: plug-and-go window (`manager_gui.py`)

```bash
python manager_gui.py
```

Same backend, clickable: device picker with states, six phase tabs
(Connect → Prep → Flash → Root → Unlock → Verify), command-echo log,
package checklist with ordered per-item-confirm uninstall, firmware file
pickers, typed-confirm dialogs wired into every destructive call, plus a
SIMULATE toggle that streams the dry-run rehearsal into the log instead
of touching hardware. UI conventions follow the classic ADB-manager
pattern (threaded calls, no frozen window). Nothing runs on launch.

## Self-containment (what it brings vs what you bring)

Brings: every script, the patch tables, the re-signer, the audit gates,
the GUI, the docs. Zero pip/npm dependencies (stdlib + tkinter only);
platform-tools self-bootstrap on first run (then works offline).

You bring: Python 3.10+, your firmware images, your APKs, an unlocked
bootloader, root, one device, cable, charge. Firmware/APKs are never
bundled (copyright, size, device-specificity) and never fetched — the
tool verifies what you point it at and refuses the rest.

## Supported devices

| Profile | Status | Notes |
|---|---|---|
| `kansas` — Moto G 5G (2025) XT2513V, MT6835 P247.01.339R | verified-live | SIM NETWORK_LOCKED→LOADED proven, remain 5, EE 0, revert tested |
| `nevada` — Moto G Play (2026) XT2615V, MT6835N W1WNS36.18-114-1 | confirmed-lab-unit | 8/8 table (CATI + decode + emu), locked→LOADED, remain 5, EE 0; revert staged, foreign-SIM open (see `NEVADA.md`) |

Adding one: `devices/README.md` (schema + 6-item hardware-proof checklist).
Untested drafts run audit/status only — flashing refuses.

---

## Standalone modem tool (same guarantees, no wizard)

Second device? The nevada port (Moto G Play 2026, confirmed on lab unit)
runs the same pipeline through `unlock.py --experimental custom` — full
story in `NEVADA.md`.

SIM-lock evaluation patch tool for **one exact device + modem build**:

- Phone: Motorola Moto G 5G (2025) **XT2513V** (`kansas`, Tracfone/Visible)
- Modem: MT6835, baseband `MT6835_NR17.RC.MP.V40.2.P247.01.339R`
- Stock md1img: 75,697,504 bytes, sha256 `371671d3…ab3272`

Provenance: developed against a live lab unit of exactly this build
(SIM state `NETWORK_LOCKED` → `LOADED` for a foreign SIM, remain counter 5/5,
zero modem exceptions, fully reversible). No personal data in this repo:
patch offsets/bytes are identical on every unit of the build; no IMEI,
serial, key, certificate, QR, ICCID, IMSI, or matching ID anywhere
(verified by audit before release).

## What this is NOT (read before anything)

- **Not universal.** It refuses any model, SKU, baseband, image size/hash, or
  bootloader state that isn't the fingerprinted build above. Offsets are
  build-specific; applying them elsewhere is how modems get bricked, so the
  tool says no instead. "Any phone" support does not exist here on purpose.
- **Not a bootloader unlock or root tool.** It *requires* both and checks:
  `securestate: flashing_unlocked` in fastboot + `su → uid 0` over adb.
  Those are per-device processes with their own key ceremonies — out of scope.
- **Never touches identity.** No IMEI read or write, no NCK entry or trials,
  no attempt-counter interaction, no protect/nvram/nvdata writes, never slot B,
  never preloader/lk/gpt/efuse. The patch changes the lock *evaluation*
  (verdict logic), never credentials.
- **Not persistent across modem OTAs.** A carrier/modem update rewriting slot A
  restores stock lock silently. Re-check SIM state after any update.

## What it does (32 bytes)

Eight same-footprint patches to the md1img modem image, then CERT2 re-sign
(accepted by the unlocked bootloader), flashed to **slot A only**:

| # | Site | Change | Effect |
|---|------|--------|--------|
| 1 | SML legal-rule entry | `LI a0,1; JRC ra` | Network-link (cat0) evaluation returns LEGAL |
| 2 | SP check entry | `LI a0,1; JRC ra` | Per-category SP-family check returns pass |
| 3–8 | `smu_check_sml` verdict gauntlet (6×) | branch → `NOP; NOP` | Every arrival falls through toward the allow arm |

Total delta vs stock: 32 bytes, image size unchanged. Old bytes are verified
before each write; any mismatch aborts (wrong build = instant refuse).

## Prerequisites

1. The exact device + build above (the tool verifies; don't argue with it).
2. Bootloader unlocked + root (see above).
3. Your own factory `md1img` for this build (integrity-checked by sha256).
4. `adb` + `fastboot` (platform-tools) and `python` on PATH.
5. One device attached, steady cable, charged battery, ~30 minutes.
6. A full backup first (the tool enforces: no backup, no flash).

## Usage

```bash
python unlock.py status                                  # read-only audit
python unlock.py backup --out backups/                  # pull live slot A (200MB)
python unlock.py build --stock FACTORY_MD1IMG --out work/patched.img
python unlock.py flash --image work/patched.signed.img --backup backups/
python unlock.py verify                                 # baseband/SIM/remain/EE
python unlock.py revert --backup backups/               # back to stock backup
python unlock.py full --stock FACTORY_MD1IMG --work work/  # backup+build, then flash explicitly
```

`flash` and `revert` require typing YES. `verify` after every flash is not
optional in spirit: baseband alive, SIM state as expected, remain counter
unchanged at 5, zero modem exceptions in dmesg.

## If something looks wrong

- Flash rejected / boot odd / unexpected SIM state: `unlock.py revert`,
  then `unlock.py verify`. Revert path is why backups are mandatory.
- Fastboot always reachable via Vol-Down+Power (cable-insert trick if looping).
- Never `fastboot -w`, never erase partitions, never flash slot B.

## Files

- `unlock.py` — the tool (stdlib only).
- `patches.py` — fingerprint + 32-byte patch table.
- `sign_mtk_cert.py` — CERT2 re-sign helper (vendored, unmodified).
- `LICENSE` — MIT.

## Legal / safety notes (short)

Unlocking a phone you own for interoperability (other carriers, private/test
networks) is lawful in many places (e.g. US Unlocking Consumer Choice Act),
but rules differ by country and carrier contract — your responsibility to
check yours. This tool changes lock evaluation only; it does not alter device
identity, bypass stolen-device blacklists, or touch network authentication.
Radio operation stays subject to your national regulator (power, bands,
equipment authorization); a lab belongs on cables/attenuators or licensed
spectrum, not on hope. No warranty; you flash at your own risk.

## Appendix: full session log (every command behind the nevada port)

How the port was actually built, in order — including dead ends. Secrets
that appeared in the live session (sudo password, GitHub token) are
redacted here; re-authenticate yourself wherever a command needs it.

**A. Repo + environment recon.**

```bash
git status --short --branch; git log --oneline -5
ls -la; cat work-wizard/state.json; cat config.json
which adb fastboot; adb version; fastboot --version
adb devices -l; ls vendor/   # -> no vendor dir; system platform-tools used
```

**B. Device fingerprint (read-only).**

```bash
adb shell getprop ro.product.model            # moto g play - 2026 (NOT kansas)
adb shell getprop ro.boot.hardware.sku        # XT2615V
adb shell getprop gsm.version.baseband        # MT6835N_NR17.RC.MP.V18.6.6.P133.02.337R
adb shell getprop gsm.sim.state               # LOADED,NOT_READY at first
adb shell getprop ro.boot.slot_suffix         # _a
adb shell getprop sys.boot_completed          # 1
adb shell getprop ro.build.display.id         # W1WNS36.18-114-1
adb shell getprop ro.boot.flash.locked        # 0 = unlocked
adb shell getprop ro.boot.verifiedbootstate   # orange
adb shell 'getprop | grep remain.count'       # [5]
adb shell dumpsys battery | grep level        # 11-14% the whole session
timeout 15 adb shell 'su -c id'               # hung first (grant pending)
adb shell 'ls -l /system/bin/su /system/xbin/su /sbin/su; which su'
adb shell 'pm list packages | grep -iE "ksu|magisk|kernelsu"'
adb shell 'ls -l /dev/block/by-name/md1img_a'
timeout 10 adb shell 'su --version'           # 30.7:MAGISKSU
timeout 10 adb shell 'magisk -v'; adb shell 'uname -r'
timeout 10 adb shell 'ls -l /data/adb/'       # Permission denied pre-grant
adb shell 'getprop | grep -iE "gsm|sim|operator|carrier"'  # Visible 311480 home SIM
```

**C. Firmware on disk.**

```bash
ls ~/ ~/Downloads; find /home/cameron -maxdepth 3 -iname "*.img"
# -> ~/Downloads/RETUS/ factory package (md1img.img, init_boot.img, ...)
python3 -c "import hashlib; ..."  # md1img.img: 76617232 B, sha256 3383f93c...
```

**D. Prove kansas bytes miss on nevada (the refusal that started the port).**

```bash
python3 -c "import imageaudit; ..."  # audit RETUS md1img: 23 entries, integrity True
python3 -c "import patches; ..."     # 0/8 kansas offsets match; kansas version absent
python3 -c "..."                     # old-byte hit counts across image (4B patterns coincide)
```

**E. Gauntlet relocation hunt.**

```bash
python3 -c "..."  # hexdumps around 0x1a17c40/0x1a17c30
# -> 6/6 kansas gauntlet bytes present as a block at 0x1a17c30..0x1a17c72
python3 -c "..."  # pair-spacing scan over 141e2412/a3349110 hits: NO pair at
                  # kansas spacing 0x10a16 -> entries did NOT move as a block
python3 -c "from capstone import ..."  # RISCV decode: garbage (wrong ISA)
python3 -c "..."  # MIPS32/MICRO decode: garbage (capstone has no nanoMIPS)
python3 -c "..."  # strings scan: smu_check_sml, simlock handlers present
```

**F. Lab repo (the method source).**

```bash
git clone --depth 1 https://github.com/Articfox291/kansas-modem-lab /tmp/opencode/kansas-modem-lab
grep -n "sl_Check\|legal_sim_rule\|smu_check_sml" docs/MODEM_CODEMOD.md  # read whole dossier
grep -n "def load_cati" -A 40 sim/emu_engine.py; grep -n "def \|cati" sim/decomp.py
grep -n "def decode_bytes" sim/decode_tables.py; grep -n "def main\|--va\|legal" sim/interp.py sim/chain.py
```

**G. CATI symbol extraction from YOUR image (the key break).**

```bash
python3 -c "import lzma; ..."   # md1_dbginfo + md1_dbginfodsp are LZMA -> CATICTNR
# cati_parse.py written: parses (prev,start,name,start,next) records -> 68,438 symbols
python3 /tmp/opencode/cati_parse.py smu_check_sml sml_sl_Check legal_sim_rule
# -> smu_check_sml VA [0x91a1789e,...]; sml_sl_Check VA [0x907398ee,...];
#    legal_sim_rule absent; full name custom_check_link_sml_legal_sim_rule VA [0x90728a68,...]
# gauntlet VAs land inside CATI smu_check_sml extent -> base 0x90000000 confirmed
```

**H. Decode every site with the lab tables.**

```bash
python3 -c "...decode_tables.decode_bytes..."  # entries: SAVE frames, clean decode
# legal entry 141e4412 (kansas 141e2412, 1B frame diff); sl entry 491e0821
python3 -c "..."  # 6 gauntlet sites: BNEIC a0/s3/a3, BEQIC s5,7, BEQC zero,s3
                  # mnemonic-identical to kansas semantics
```

**I. Emulation differential (lab interp engine, overlay, no file writes).**

```bash
# nevada_diff.py written: monkeypatches image loader to our md1rom carve,
# runs stock vs force-pair overlay per function
python3 /tmp/opencode/nevada_diff.py
# legal: stock 24-step a0=0, patch 1-step a0=1 PASS
# sl_Check: stock a0=0, patch 1-step a0=1 PASS
# smu: documented SWM engine gap, stock+patch identically -> live test decider
```

**J. Table builds (wrong arithmetic caught by gates, then fixed).**

```bash
python3 -c "..."  # first shift attempt -0xF26C8A printed wrong VAs -> discarded
python3 -c "..."  # correct shift +0x89376, all 6 old-bytes verified in-image
python3 -c "..."  # wrote nevada_table.DRAFT.json (6 entries), then 8/8 with
                  # entry old bytes read from image (141e4412, 491e0821)
```

**K. Runner + config + profile.**

```bash
# postroot.py written (zero prompts; --flash/--charge-anyway/--experimental/--patches)
# mylab-nevada.json, devices/nevada.json (untested-draft, zero patch bytes) written
# unlock.py: load_table accepts {"patches": [...]} metadata wrapper
python3 -m py_compile postroot.py unlock.py; grep -n "input(" postroot.py  # must be empty
```

**L. Enforced builds.**

```bash
echo EXPERIMENTAL | python3 unlock.py --experimental custom \
  --stock ~/Downloads/RETUS/md1img.img --patches nevada_table.DRAFT.json \
  --out work-nevada/custom_work.img   # 6 runs declared, re-sign, re-parse OK
```

**M. Backup fallback fix + full hands-free run.**

```bash
# unlock.py cmd_backup: su-dd fallback added (adbd cannot read block devs)
python3 postroot.py --config mylab-nevada.json --work work-nevada \
  --experimental --patches nevada_table.DRAFT.json </dev/null
# -> BLOCKED on su grant first run; after on-phone Allow tap: root OK,
#    backup 200MB in 21s via su-dd, build OK, "done. Nothing flashed"
python3 -c "..."  # live-prefix sha256 == factory sha256 MATCH
```

**N. Flash.**

```bash
adb devices; adb shell dumpsys battery | grep level   # 14%, USB powered
adb shell 'su -c id'                                  # uid=0 still granted
adb shell getprop gsm.sim.state                       # NETWORK_LOCKED (baseline)
adb shell 'getprop | grep -E "sim.operator|operator.numeric"'  # empty = no registration
python3 postroot.py --config mylab-nevada.json --work work-nevada \
  --experimental --patches nevada_table.json --flash --charge-anyway </dev/null
# Sending OKAY, Writing OKAY (~3 s), reboot, 90 s wait
python unlock.py verify   # LOADED, remain [5], EE 0, baseband alive
```

**O. Phone control + housekeeping.**

```bash
sudo apt install scrcpy          # needs YOUR password on YOUR terminal
scrcpy --version; adb devices
cp /tmp/opencode/cati_parse.py /tmp/opencode/nevada_diff.py .
mkdir -p /tmp/opencode/kansas-modem-lab/nevada_port  # copies of port files
```

**P. Branch + PR.**

```bash
git remote -v; git branch --show-current; git log --oneline -3; which gh
git config user.name "crabcakes97"; git config user.email "eddiebob146@gmail.com"
git checkout -b nevada-port
git add unlock.py .gitignore postroot.py mylab-nevada.json devices/nevada.json \
  nevada_table.json cati_parse.py nevada_diff.py NEVADA.md README.md
git diff --cached unlock.py .gitignore   # review before commit
git commit -m "..."; git remote add fork https://github.com/crabcakes97/kansas-modem-unlock.git
# push needs auth: git -c http.extraHeader="AUTHORIZATION: basic $(printf 'crabcakes97:%s' "$TOKEN" | base64 -w0)" push fork nevada-port
# (Bearer form fails for git-over-HTTPS; Basic works. Never store the header.)
# PR opened in browser (token lacked PR scope):
# github.com/Articfox291/kansas-modem-unlock/compare/main...crabcakes97:nevada-port
```
