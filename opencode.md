# opencode.md — how we got here (nevada XT2615V modem/LK research)

Device: Motorola Moto G Play 2026 (`moto g play - 2026`), SKU `XT2615V`,
codename `nevada`, SoC MT6835 (ARMv8), Android 16, slot `_a`,
bootloader `flashing_unlocked`, rooted (Magisk `su → uid 0`).
Baseband `MT6835N_NR17.RC.MP.V18.6.6.P133.02.337R`, build `W1WNS36.18-114-1`.
Phone LK is bootloader **-111-3**; RETUS factory package is **-114-1**.

## 1. Pulled the live LK from the phone (not lab images)

Found partitions: `lk_a → /dev/block/sdc49`, `lk_b → /dev/block/sdc72`
(16,777,216 bytes each). No `/dev/block/by-name/lk` exists on this
A/B device — any guide naming it is wrong for nevada.

```bash
adb shell 'su -c "dd if=/dev/block/by-name/lk_a of=/sdcard/lk_a_phone.img bs=4096"'
adb pull /sdcard/lk_a_phone.img "patched lk/lk_a_phone.img"
```

- Raw partition sha256 `0b0cd33e…e7fc` (matched live block device).
- Real content 3,351,917 bytes (`lk_a_phone_nopad.img`, sha `c392dbbf…`);
  rest is trailing-zero partition padding.
- Extracted LK payload 1,238,168 bytes (`lk.img`, sha `ad52731a…`):
  AArch64, base `0xffff000050f00000`.
- Saved in `patched lk/` with full analyzer output (`analysis-phone/`).

Phone-vs-stock subimage diff (liblk, 5 partitions each): `lk` payload
same size, different sha (Val unlock patch lives here); `aee` differs;
`bl2_ext`/dtb/dtbo identical. Container delta +1117 bytes is CERT
re-sign area. Phone LK reports `-111-3`, stock `-114-1`.

## 2. Killed four wrong assumptions with byte scans of the phone LK

`modem_auth`, `load_modem_fw`, `mmu_table_init`, `armv7_mmu_init`: **all
absent** (0 hits). Consequences, verified not guessed:

- LK is **not EL3** — it calls *into* ATF/EL3 via SMC. No LK patch yields EL3.
- LK's MMU/EMI-MPU/mblock setup is **rebuilt by the kernel at boot** —
  LK remaps evaporate.
- Modem signature verification is **not in LK** — CERT2 re-sign stays the
  working bypass.
- The 1.1KB modem payload limit is free space inside md1img, not an LK check.

## 3. Found the real modem-load surface (disassembly, capstone AArch64)

- `platform_load_modem` (`…FD0EF2`), `ccci_plat_apply_mpu_setting`
  (`…FD92F4`), `emi_mpu_set_protection` (`…FD0D1A`), `arm64_mmu_*`,
  `mtk_wdt_doe_setup`, `motorola_alloc_mblock`, `ccci_sec_data`,
  `md1_sib_mem`, `md1_bank4_cache_info`.
- Function near `…F49A24` validates the MD table: region id must be
  `0xBC/0x200/0x11C`, size bound `w22 <= w20`; violations log + return NULL.
- DT (`lk_main_dtb`) holds the overlap truth: `emimpu@10226000`,
  `emi_mpu`, `reserved-memory`, `ccci-dpmaif-*`, `md1_ccif`.

## 4. Built `--preset modem-unlock` in val-protocol (committed + pushed)

- Report-only by default: 12 real markers with offsets/VAs, 4 absent names.
- `--modem-size-bypass --modem-allow-unsafe`: 3 same-footprint NOPs,
  old-byte gated, wrong build = instant refuse:

| gate | lk.bin off | old → new |
|---|---|---|
| region-id `b.ne` fail | `0x49A9C` | `21060054` → `1f2003d5` |
| size-bound-1 `b.hi` fail | `0x49AAC` | `88040054` → `1f2003d5` |
| size-bound-2 `b.hi` fail | `0x49B24` | `c8000054` → `1f2003d5` |

- 12 bytes total; full pipeline re-signs `Result: VALID`; other subimages
  identical. Same old bytes on -111-3 and -114-1.
- Also fixed a real analyzer crash (`write_summary` validator/erase-op bug).
- What it does/doesn't do: removes LK's table-validation gate only. No
  signature bypass, no DMA overlap, no watchdog/EL3 changes.

## 5. Signed modem image anatomy (work-nevada/custom_work.signed.img)

- 8/8 nevada patches verified live in the signed image.
- Signed = unsigned + **96-byte CERT2 insert** at `0x2C65354`; all trailing
  bytes shift +96 (verified exact). Never hand-edit signed — rebuild from
  stock via `unlock.py`.
- `sign_mtk_cert.py` updates **only the first CERT2 (md1rom's)** → patchable
  = md1rom entry only (44.4MB). dsp/drdi/dbginfo patches would fail secure boot.
- Space vs reachability: 5,664 zero-caves (2.9MB, largest 189KB) — but
  **zero caves ≥1.1KB within ±128KB of the hooks** (nearest: 28B paddings at
  +227KB; nearest big cave 6.8MB away). Next payload room must come from
  dead-code reclamation near a hook, not distant caves. Zeros ≠ executable
  without MPU changes.

## 6. BROM / factory mode / ramdump (live fastboot probes)

- BROM code-exec/download is closed (SLA/DAA fuses; needs Motorola-signed DA).
  LK only knows the BROM-DIS fuse status. Never blow BROM-DIS fuses.
- Factory mode exists (`Entering factory mode`, `kpd_hw_factory_key`,
  `mmi,factory-cable`) and gates protected oem commands. No factory cable
  on hand → pursuing software paths (bootmode UTAG, `fb_mode_set/clear`).
- `fastboot oem ramdump` → `command restricted` (the factory gate).
  `oem config` (no args) dumps the UTAG list; single-name reads returned
  empty (syntax TBD); `oem hwid list` / `get_socid` return empty OKAY.
- Ramdump (full DRAM incl. modem-shared regions) is the prize: a read
  primitive with no modem exploit. Gated behind factory mode / unprotect.
- BROM dump via ramdump needs arbitrary-phys ranges + still-mapped ROM —
  unproven; BROM *exploit* is a different tool class (host USB), not a Val
  preset. A BROM-*dumper* LK patch could fit Val architecturally if range
  reads prove possible.

## Open next steps

1. Full UTAG dump → find ramdump/fullramdump/factory-cable knobs.
2. `fb_mode_set factory` → factory mode without cable → `ramdump enable`.
3. BROM readability probe at phys `0x0` (zeros/abort = stop).
4. Next modem patch needs a defined target + nearby dead code survey.
