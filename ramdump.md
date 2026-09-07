# ramdump.md — full-DRAM capture via unlocked fastboot (nevada XT2615V)

Goal: read physical DRAM (AP + modem-shared regions) over USB with no
modem exploit. Status: **command chain proven live, trigger + full
retrieval still open** (see §5).

## 0. Preconditions

- Unlocked bootloader, `lk_b` flashable, stock RETUS `lk.img` on hand.
- 2-gate `factory-allow` LK image built (see §1). Device slot `_a` healthy.

## 1. Patch: `--preset factory-allow` (val-protocol, committed + pushed)

Two NOPs in the oem dispatcher, old-byte gated (wrong build = refuse):

| gate | lk.bin off / VA | old → new | unlocks |
|---|---|---|---|
| dispatcher deny (`tbz` → "command restricted") | `0xF3F4` / `…F0F3F4` | `00010036` → `1f2003d5` | `oem ramdump …` answers usage |
| config-subcommand deny (`tbz` → "Not allowed command") | `0xAD88` / `…F0AD88` | `a0090036` → `1f2003d5` | `oem config …` reaches UTAG layer |

```bash
python3 val-protocol/lk_auto_patch.py "patched lk/lk_a_phone_nopad.img" \
  -o /tmp/lk_factory_allow2.img --preset factory-allow \
  --factory-allow --factory-allow-unsafe
# verify: exactly 8 bytes differ (0xAD88, 0xF3F4), re-sign VALID
```

Fastboot-only code — Android boot untouched.

## 2. Flash + verify the ungate (inactive slot only)

```bash
fastboot flash lk_b /tmp/lk_factory_allow2.img
fastboot --set-active=b
fastboot reboot bootloader
fastboot oem ramdump            # was: "command restricted" → now: usage text
fastboot oem config unprotect enable_fulldump   # reaches handler now
```

Proven live twice. `config unprotect` with bad names answers
`no such utag` (through the gate, vocabulary problem only).

## 3. Enable + knobs (proven)

```bash
fastboot oem ramdump enable     # → "enable full ramdump", OKAY
```

- UTAG `enable_fulldump=true`, already unprotected — nothing to unprotect.
- `oem mrdump_out_set|chkimg|fallocate` are NOT oem commands here;
  `mrdump_output` is NOT a UTAG (internal setting, address TBD).
- `oem help` lists 12 commands; ramdump stays hidden-but-working.

## 4. Baselines (recorded, rooted Android, slot A system)

```bash
# expdb (128MB AEE log/mini-dump store, NOT full-DRAM target):
su -c "blockdev --getsize64 /dev/block/by-name/expdb"   # 134217728
su -c "sha256sum /dev/block/by-name/expdb"              # 427b4f8c… (pre-trigger)
# content: zeros to ~0x7D00000, charger/kernel logs in last ~3MB
```

expdb physically cannot hold 8GB DRAM — it takes logs + LK-stage
memdumps (`bl33_memdump` strings). Full-DRAM destination is unresolved:
strings imply a USB write path (`Failed to write to USB`) but no fetch
command was found — a host-side receiver protocol is the missing piece.

## 5. Trigger (VALIDATED 2026-09-07, logs-only result)

SysRq crash from rooted system (needs `echo 1 > /proc/sys/kernel/sysrq`
first; stock has it `0`):

```bash
su -c "echo c > /proc/sysrq-trigger"   # Kernel panic - not syncing
```

Observed: panic at `sysrq triggered crash` → ramoops capture
(`console-ramoops-0` 262KB + `dmesg-ramoops-0` 9KB, pstore was empty
before) → clean reboot in ~90s. expdb grew +239,602 B (log rotation).
**No DRAM contents anywhere** — kernel crashes yield the 896KB ramoops
window (`ramoops.mem_address=0x48090000 mem_size=0xe0000` on cmdline),
never full memory. `mrdump.ko` is loaded live (159744 B, 18 holders incl.
`ccci_dpmaif`, `ccci_md_all`) — the modem-wired capture path exists in
kernel, but its full-memory output still needs the unresolved
output-device selector (§4). Trigger works; retrieval is the gap.

## 6. Recovery (every step reversible)

- In fastboot with USB: `fastboot --set-active=a` (+ `fb_mode_clear` if set).
- Wedged bootloader (enumerates, silent): hold Power 12s; black screen →
  Vol-Down+Power 12s → fastboot.
- Deep fallback: `fastboot flash lk_b <RETUS>/lk.img`; deeper: `unlock.py
  revert`. Never touch preloader/efuse; never blow BROM-DIS fuses.
