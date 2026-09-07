# totalfinding.md — everything this chat (nevada XT2615V, MT6835)

## Device + ground truth

- Moto G Play 2026, `XT2615V`/`nevada`, MT6835 ARMv8, Android 16, slot `_a`,
  unlocked + Magisk root. Baseband `MT6835N_NR17.RC.MP.V18.6.6.P133.02.337R`.
- No `/dev/block/by-name/lk` on A/B — only `lk_a` (sdc49) / `lk_b` (sdc72),
  16MB each. `md1img_a/b`, `utags` (sdc9) + `utagsBackup` (sdc10),
  `misc`/`para`/`boot_para`, `expdb` (128MB), `cid`, `nvram`, `seccfg`.
- Phone LK = bootloader **-111-3** (AArch64, 1,238,168 B, base
  `0xffff000050f00000`); RETUS factory = **-114-1** (same gate code).
  Pulled live LK over `su`+`dd`, sha-verified vs block device
  (`0b0cd33e…`), trimmed (3,351,917 B, `c392dbbf…`), saved in `patched lk/`.
  **Every binary finding below comes from this pull — no lab images.**

## Four wrong assumptions killed by byte scan

`modem_auth`, `load_modem_fw`, `mmu_table_init`, `armv7_mmu_init`: 0 hits.
Hence: LK ≠ EL3 (calls ATF via SMC); LK memory maps die at kernel boot;
modem signatures verified outside LK; 1.1KB limit is md1img free space.
BROM code-exec/download closed (SLA/DAA fuses; needs Motorola DA).
Fenrir/pwnage patched here per prior findings; sprig = bl2_ext stage
(harder brick class, not an LK preset).

## Real modem-load surface (capstone disassembly of phone LK)

`platform_load_modem` `…FD0EF2`, `ccci_plat_apply_mpu_setting` `…FD92F4`,
`emi_mpu_set_protection` `…FD0D1A`, `arm64_mmu_*`, `mtk_wdt_doe_setup`,
`motorola_alloc_mblock`, `ccci_sec_data`, `md1_sib_mem`,
`md1_bank4_cache_info`. MD-table validator near `…F49A24`: region id ∈
`{0xBC,0x200,0x11C}`, bound `w22<=w20`, else log + NULL return.
DT (`lk_main_dtb`): `emimpu@10226000`, `emi_mpu`, `reserved-memory`,
`ccci-dpmaif-*`, `md1_ccif`, `modem_temp_share` — kernel + LK both consume
it, so LK-only remaps evaporate.

## Val Protocol additions (committed + pushed to crabcakes97/val-protocol)

- `--preset modem-unlock`: report-only (12 real markers + 4 absent names);
  `--modem-size-bypass --modem-allow-unsafe` NOPs 3 MD-validation gates
  (`0x49A9C:21060054`, `0x49AAC:88040054`, `0x49B24:c8000054` → `1f2003d5`;
  12 bytes, re-sign VALID). LK-side parsing only — no sig/DMA/WDT/EL3.
- `--preset factory-allow`: NOPs dispatcher deny `0xF3F4:00010036` and
  config-subcommand deny `0xAD88:a0090036` (8 bytes, VALID). **Proven live
  on slot B**: `oem ramdump` usage (was `restricted`), `ramdump enable`
  OKAY, config reaches UTAG layer.
- Fixed `lk_static_analyzer.py write_summary` crash (validator/erase-op
  copy-paste; erase ops carry `failure_target`).
- Docs: `MODEM_UNLOCK.md` (forensics), main `README.md` sections.

## Signed modem image (work-nevada/custom_work.signed.img)

- 8/8 nevada patches live; unsigned == stock + exactly 32 B.
- Signed = unsigned + 96 B CERT2 insert at `0x2C65354` (trailing shift
  verified exact) — never hand-edit signed, rebuild via `unlock.py`.
- Re-sign covers **md1rom only** → patchable = md1rom (44.4MB).
- Space vs reachability: 5,664 zero-caves (2.9MB, biggest 189KB) but
  **zero ≥1.1KB caves within ±128KB of hooks** (nearest: 28B pads +227KB;
  nearest big cave 6.8MB away). Next payload room = dead-code reclamation
  near a hook. Zeros ≠ executable without MPU changes.

## Slots, fastboot, factory mode, ramdump (live)

- Fastboot lives in LK: slot B works with no system. Patched LK on A would
  boot Android normally (fastboot-only code paths) — A stays known-good
  until B proves everything.
- `fb_mode_set factory` → persistent force-fastboot (reboot loops to
  bootloader; ramdump still restricted — NOT the factory ungate);
  `fb_mode_clear` restores normal. One wedge needed Power-hold recovery.
- `reboot factory` (root): ignored. `misc` = plain BCB; `para` binary, no
  bootmode strings; `boot_para` all zeros; `utags` = identity only (never
  write). Factory cable state is hardware-sensed (`Cannot update`).
- No factory cable on hand; software factory entry still open (UTAG dump
  unfinished — streams for minutes; `oem config` handler argv at lk.bin
  `0xD4EEB` is the RE target).
- Ramdump chain: ungated + enabled live; `enable_fulldump=true`
  pre-existing unprotected; `mrdump_*` not commands; `mrdump_output`
  internal. expdb baselined (sha `427b4f8c…`, logs-only in last ~3MB).
  Full-DRAM destination unresolved (USB write path implied, no fetch
  command found). Trigger (sysrq) + retrieval = next session.

## Key numbers (all verified, not estimated)

- LK gates: 3×4 B modem + 2×4 B factory, old-byte values confirmed on two
  builds. Modem patches: 8×4 B = 32 B. Caves: 5664/2.9MB/189KB-max.
- Slo,B test loop proven twice: flash `lk_b` → `--set-active=b` →
  fastboot works → `--set-active=a` → normal boot, SIM LOADED, root OK.

## Open, in order

1. Ramdump trigger (sysrq) + watch USB/expdb; find output-device selector.
2. UTAG full dump → exact knob names; remaining `tbnz` gates only if needed.
3. Next modem patch = defined target + nearby dead-code survey.
4. scp.img (CM4 bridge core, AP↔MD shared mem) as alternate cross-domain path.
