# Nevada port (experimental) — Moto G Play 2026 XT2615V

Experimental port of the modem-unlock flow to a second device. Status:
**boots, flashes, locked→LOADED once on the lab unit — NOT proven.**
Foreign-SIM confirmation and revert test are still open. Everything below
is the full method so anyone can reproduce or refute it.

## Device + firmware

- Phone: Motorola Moto G Play 2026, `nevada` (`ro.product.model`
  `moto g play - 2026`), SKU `XT2615V`, build `W1WNS36.18-114-1`
- Modem: `MT6835N_NR17.RC.MP.V18.6.6.P133.02.337R` (same MT6835 family as
  kansas, different build)
- Stock md1img: 76,617,232 bytes,
  sha256 `3383f93c2d1be9d36b3e291a67415c6c13345fe06843e8a95720e017021d3ba5`
  (RETUS factory package; full-parse audit: 23 entries, integrity True)
- Profile: `devices/nevada.json` (`untested-draft` — the wizard refuses to
  flash it; it carries **zero** patch bytes by rule)

## How the table was found (so you can check the work)

1. The kansas 8 shipped offsets miss everywhere on this image (0/8,
   size + sha + version literal all differ) — direct application refused.
2. The 6-branch `smu_check_sml` gauntlet is byte-identical, shifted
   `+0x89376` vs kansas (`nevada = kansas + 0x89376`), gaps included.
3. The 2 function entries were located by name in the image's own CATI
   symbol table (`md1_dbginfo`, LZMA — see `cati_parse.py`): 68,438 symbols,
   `custom_check_link_sml_legal_sim_rule` @ VA `0x90728a68`,
   `sml_sl_Check` @ VA `0x907398ee`, `smu_check_sml` @ VA `0x91a1789e`
   (modem VA base `0x90000000`, `img_off = VA - base + 0x200`).
4. Every site round-trips through the lab's nanoMIPS decode tables; the six
   branches decode mnemonic-identical to the kansas ones
   (`BNEIC a0/s3/a3`, `BEQIC s5,7`, `BEQC zero,s3`).
5. Both entry patches execute in the lab's simulator (`nevada_diff.py`,
   overlay, no file writes): stock returns deny in 24 / 2727 steps,
   patched returns 1 in 1 step. The `smu` full-function run stops at the
   engine's documented SWM gap (stock and patch identically) — live
   behavior is the decider there, same standard kansas was held to.

## Run it (non-interactive, hands-free past rooting)

```bash
cp mylab-nevada.json mylab.json   # or pass --config mylab-nevada.json
# point mylab firmware_files.md1img_stock at YOUR factory md1img first
python postroot.py --config mylab-nevada.json --work work-nevada \
  --experimental --patches nevada_table.DRAFT.json
```

That does fingerprint, full-parse audit, old-byte gates, byte-exact build,
re-sign, live backup (with factory-hash comparison), and stops. The one
destructive step needs all of: `--flash`, 30%+ battery (or `--charge-anyway`
on your explicit risk), backup present, unlocked bootloader:

```bash
python postroot.py --config mylab-nevada.json --work work-nevada \
  --experimental --patches nevada_table.DRAFT.json --flash
python unlock.py verify            # LOADED, remain [5], EE 0
python unlock.py revert --backup work-nevada/backups   # way back
```

`postroot.py` never prompts: the one-time Magisk/KernelSU "allow Shell" tap
happens on the phone, then everything runs (bounded waits, fail-fast).

## Live result (one lab unit, local receipts only)

- Before: `NETWORK_LOCKED`, remain 5, no registration.
- Flash `md1img_a`: Sending OKAY, Writing OKAY (~3 s). Slot B untouched.
- After: baseband alive, `LOADED`, remain 5, zero modem exceptions.
- Live backup prefix hashes exactly to the factory sha above.
- Receipt: `work-nevada/FLASH_RECEIPT.txt` (git-ignored, stays on your disk).

## Open items (do these before calling it proven)

1. Foreign-SIM test (non-Verizon-family): stock `NETWORK_LOCKED` →
   patched `LOADED`, remain 5, EE 0.
2. Revert round trip back to stock behavior.
3. OTA relock watch: any modem update restores stock lock silently.
4. Keep charge 30%+ for any flash; never flash what you cannot revert.

## Files

- `nevada_table.DRAFT.json` — the 8/8 table (offsets are file offsets).
- `devices/nevada.json` — fingerprint + policy, no patch bytes.
- `postroot.py`, `mylab-nevada.json` — runner + lab config.
- `cati_parse.py`, `nevada_diff.py` — symbol extraction + emu differential.
- `unlock.py` (`--experimental custom`) — the enforced build pipeline.
