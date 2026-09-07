# fastboot.md — no-cable factory-mode path (nevada XT2615V)

Goal: factory mode without the Motorola factory cable, to ungate
`fastboot oem ramdump` (full DRAM incl. modem-shared regions).
Phone boots `ro.bootmode=normal`, slot `_a`, unlocked + rooted.

## What the LK exposes (from your pulled LK, `patched lk/analysis-phone/lk.bin`)

- `Entering factory mode`, `kpd_hw_factory_key`, `mmi,factory-cable`,
  `bootmode UTAG is set to factory`, `(valid values are "fastboot",
  "factory")`, `fb_mode_set` / `fb_mode_clear`, `unprotect <name>`,
  `fastboot_after_ramdump`, `factory_kill_timeout`.
- `fastboot oem ramdump enable|disable`, `mrdump_*` output controls,
  `oem usb2jtag`, `oem p2u`, `oem get_socid`, `oem hwid`,
  `oem show_screen`, `oem dump_pllk_log`.
- BROM-DIS fuse status is readable; **never blow/reblow BROM fuses**
  (permanent, kills blankflash recovery).

## Live probe results

| command | result |
|---|---|
| `fastboot oem ramdump` | `command restricted` (the factory gate) |
| `fastboot oem config` (no args) | streams whole UTAG table (very long) |
| `fastboot oem config <name>` reads | empty output (syntax unconfirmed) |
| `fastboot oem hwid list` / `get_socid` | empty OKAY |
| `fastboot oem config unprotect ramdump` | `Not allowed command` |

## fb_mode experiment (the no-cable lever — partial win)

```bash
adb reboot bootloader
fastboot oem fb_mode_set factory     # OKAY (first try wedged: hold Power 12s)
fastboot reboot                       # loops back to fastboot, not system
fastboot oem ramdump                  # STILL restricted
fastboot oem fb_mode_clear            # OKAY
fastboot reboot                       # boots normal, ro.bootmode=normal
```

Learning: `fb_mode_set factory` writes a **persistent force-fastboot**
flag (reboot loops to bootloader until cleared) — so flag writes work —
but it is **not** the factory-mode ungate. `fb_mode_clear` fully restores
normal boot. If `fb_mode_set` ever wedges (USB enumerates, no answers):
hold Power 12s; black screen → Vol-Down+Power 12s → `fb_mode_clear`.

## UTAG store (root reads, all backed up to /tmp — do NOT write)

- `utags` (sdc9, 512KB) + `utagsBackup` (sdc10): identical; identity UTAGs
  (imei, battid, carrier, hwid, slot mbm_ver). **No bootmode/factory entries.
  Never write here.**
- `misc` (sdc1): standard BCB (`bootonce-bootloader` leftovers only).
- `para` (sdc2): 164KB binary, no bootmode/factory strings.
- `boot_para` (sdc21, 1MB): all zeros.
- `reboot factory` (root): ignored, normal reboot.

## Remaining no-cable ideas (untried)

1. Full UTAG dump to file (needs minutes uninterrupted) → find the exact
   ramdump/fullramdump/factory-cable knob names + protected flags.
2. `oem config unprotect <exact-knob>` with exact names from (1).
3. Disassemble the `oem config` handler (usage at lk.bin `0xD4EEB`) for exact
   argv syntax instead of guessing.
4. `adb reboot meta` (meta ≠ factory: NVRAM access, no ramdump).
5. DIY factory cable (ID-pin resistor value for nevada unconfirmed — verify
   before cutting anything).

## Results (live slot-B tests)
- `factory-allow` v1 (dispatcher tbz `0xF3F4`→NOP): `oem ramdump` went from
  `command restricted` to usage text; `ramdump enable` → OKAY.
- `factory-allow` v2 (+config tbz `0xAD88`→NOP): `config unprotect
  <bad-name>` now reaches the handler (`no such utag`) instead of
  `Not allowed command`. Real knob found: **`enable_fulldump=true`,
  already unprotected** — no unprotect needed.
- `oem usb2jtag/p2u/dump_pllk_log/printk-ratelimit`: not supported commands
  (internal strings only, no handlers — cannot be ungated, nothing to point at).
- `getvar cid` → `0x33` = **51 = RetailLocked** (per RETUS signing-info).
  `oem cid_prov_req` returns a device-specific provisioning blob (keep local):
  CID change needs Motorola-signed response — no local flip exists. SIM-lock
  freedom already comes from the modem patches (LOADED), not CID.
- SLA verdict (from boot logs in expdb): `sbc_en=1`,
  `img_auth_required=1` on every image incl. `bl2_ext cert vfy → ok`.
  SLA/DAA cannot be turned off from here (BROM+preloader enforced, fused);
  unlocked fastboot already covers our flash needs — SLA only gates
  BROM-download tools we don't use.
- UTAG `console` reads cleanly on slot B (protected=false, empty value):
  set forms per its description: `enable|true`, `disable|false`, or custom
  `"ttyS0,921600n1"`. Enables kernel serial console (needs UART pads to see;
  pstore console-ramoops already works without it). `tool_by_pass_pwk` and
  `bptools` are NOT utags (code/menu identifiers, not settable values).
- Tools Mode = bootloader MENU item (`Switch_tools_mode` in menu list next
  to `restartbootloader`): Vol keys to highlight, Power to select. No command
  enters it; `reboot bptools` is ignored (normal reboot). Needs fingers.
- GZ-canary test signal: slot-B **fastboot presence** (fastboot needs
  GZ→LK, so USB fastboot = hypervisor accepted+ran). Android on B is NOT
  required and never will be (no system_b).

## Max-access notes (retail build realities)

- No EngineerMode/CQATest/factory apps installed — nothing to launch.
- `adb root`: refused (production build). No adbd root, ever here.
- vbmeta: `fastboot --disable-verity --disable-verification flash
  vbmeta_a/vbmeta_system_a` (RETUS files) boots fine, orange. Direct
  `/system` RW still impossible — system is **erofs** (read-only by design,
  not policy). Magisk systemless overlays remain the way.
- Meta/factory/bptools reboot reasons all ignored (normal reboot).
  Dialer codes need fingers: `*#*#3646633#*#*` (MTK),
  `*#*#2486#*#*` (Moto CQATest, likely absent).

## Safety rules

- Keep USB plugged in (factory_kill_timeout can power off on disconnect).
- `fb_mode_clear` + `fastboot reboot` always gets you home.
- Stock RETUS `lk.img` + `unlock.py revert` stay the deeper safety net.
