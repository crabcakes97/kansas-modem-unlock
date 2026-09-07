# Factory mode (nevada XT2615V) — no-cable entry attempts

Goal: reach factory mode **without the Motorola factory cable**, because
factory mode ungates protected fastboot commands — chiefly
`fastboot oem ramdump enable` (full DRAM incl. modem-shared regions) and
`fastboot oem config unprotect`.

Status: **not there yet — phone boots `ro.bootmode=normal` every time.**
Everything below is redoable step by step.

## Why factory mode matters

- `fastboot oem ramdump` answers `command restricted` in normal fastboot.
- LK strings prove the gate: `Entering factory mode`, `kpd_hw_factory_key`,
  `mmi,factory-cable`, `bootmode UTAG is set to factory`,
  `(valid values are "fastboot", "factory")`, `fb_mode_set` / `fb_mode_clear`,
  `unprotect <name> / protect <name>`, `fastboot_after_ramdump`,
  `factory_kill_timeout` (auto-shutdown if USB drops in factory — **keep the
  cable plugged in**).
- Never touch anything mentioning BLOW / reblow / `BROM DIS` fuses. Status
  reads are safe; blowing is permanent and kills blankflash recovery.

## Method 1 — `fb_mode_set factory` (tried, wedged, needs redo)

```bash
adb reboot bootloader
timeout 30 fastboot oem fb_mode_set factory
```

Observed: no output, bootloader stops answering (USB still enumerates).
Recover: **hold Power 12s** → forced reboot (boots normal if the write
didn't commit). If the screen stays black: Vol-Down+Power 12s → fastboot,
then `fastboot oem fb_mode_clear` + `fastboot reboot`.
Redo with variations: `fb_mode_set` with no args (dumps UTAG XML — proves
the command parses), then retry `factory` once more; on wedge, power-cycle.

## Method 2 — `reboot factory` from rooted Android (tried, ignored)

```bash
adb shell 'su -c "reboot factory"'
```

Observed: plain reboot, `ro.bootmode` still `normal`. Reason string is
ignored by init/LK. Redo cost is one reboot; harmless.

## Method 3 — UTAG `bootmode` via `oem config` (syntax unknown, redo)

- `fastboot oem config` (no args) streams the whole UTAG table — long output,
  capture with `timeout 200 fastboot oem config | head -c 300000 > utags.txt`.
  (Two attempts were aborted mid-stream; it needs minutes, not seconds.)
- Single reads (`oem config bootmode`, `… factory-cable`, `… fastboot_mode`,
  `… fullramdump`, `… ramdump`) returned empty — syntax unconfirmed.
- Suspected write path once syntax is known:
  `fastboot oem config unprotect bootmode` → set `bootmode=factory` →
  `fastboot reboot` → every bootloader entry is factory-privileged.
- LK usage fragment to decode: `<value of .chosen><parent utag name>=`.
  Next RE step: disassemble the `oem config` handler around the usage string
  at lk.bin `0xD4EEB` to get exact argv order (248 adrp refs to that page —
  needs narrowing, not brute force).

## Method 4 — `adb reboot meta` (not yet tried)

Meta mode ≠ factory mode (NVRAM/calibration access, no ramdump), but free
and harmless. Worth one cycle for modem NVRAM visibility.

## Method 5 — DIY factory cable (fallback, needs a sacrificial cable)

Motorola factory cables signal via the USB ID pin; exact resistor value for
nevada is unconfirmed — do not guess. Confirm from the TWRP nevada tree or
measure a known-good cable before cutting anything.

## Once factory mode is reached (verify first!)

```bash
adb shell getprop ro.bootmode        # expect: factory
fastboot oem ramdump                 # expect: usage text, NOT "restricted"
```

Then: `fastboot oem ramdump enable` (may reboot — save state first),
`fastboot oem config unprotect <name>` as needed, full UTAG dump to file,
and `mrdump_*` output controls. To leave: `fastboot oem fb_mode_clear`
(if it exists as a command) + `fastboot reboot`, confirm `ro.bootmode`
back to `normal`.

## Attempt log

| # | method | result |
|---|---|---|
| 1 | `fb_mode_set factory` | hung bootloader, power-cycle → normal boot |
| 2 | `reboot factory` (root) | normal reboot, `ro.bootmode=normal` |
| 3 | UTAG full dump | aborted mid-stream twice (slow output) |
| 4 | single UTAG reads | empty output, syntax unconfirmed |
