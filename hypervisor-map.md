# hypervisor-map.md — GenieZone disassembly map (nevada XT2615V, MT6835)

Source: live `gz_a` pull (`pulled rom/gz_a.bin`, 32MB part / 1.6MB image),
`tools/sign_mtk_cert.py` VALID, CERT2-forged boots proven on slot B.
Payload linked linearly from file `0x200`; 1425 self-consistent
adrp+add→string resolutions. 398-name symbol pool present. Live policy
from `pulled rom/gz_log_live.txt` (62KB ring, read-only).

## Image layout

- Container: MTK part image, gz payload `0x200..0x192800`, CERT1 + CERT2.
- No HVC instructions in payload (6 HVC-shaped words are rodata); 5 SMC
  sites, 2 ERET sites. Entry from below is SMC-routed; exit via ERET.

## SMC call sites (file offsets, AArch64)

| site | semantics |
|---|---|
| `0xA568` | generic SMC forwarder: loads x0–x7 from struct, `smc #0`, restores
  (trusty fastcall gateway) |
| `0xB8A8` | SiP `0x8200FF03`, all-zero args, inside EL2 SYNC-exception dispatch
  (nested call down to EL3 runtime; exact service TBD) |
| `0xB954` | PSCI-style power call (`w9` masked `0xffff3fff\|0x8000`) |
| `0x2AED0`, `0x2AEEC` | bare `smc #0; ret` trampoline stubs |

## Exception dispatch (around `0xB838`)

- Reads `ESR_EL2`, table-dispatches (`ldr x1,[x3,w1,sxtw#3]`), per-EC
  handlers; `data_abort_curr/lower_el_handler`, `gz_kernel_exception`,
  `hvc_undefined` in symbol pool.
- Call routing framework: `el3_smc_entry`, `route_smc_to_el3`,
  `trusty_mt_sm_fastcall`, `trusty_mt_smcall`, `trusty_sm_stdcall`,
  `sm_queue_vmcall/stdcall`, `sm_get_vmcall_ret` — guest→EL3 calls pass
  through GZ (the interception point for any future runtime work).

## Memory capability inventory (symbol pool, grouped)

- Page tables: `hyp_mmu_init` (two-pgd + type2, per live log),
  `gzvm_map_pgd`, `gz_mm_el1s2pgt_map/unmap`, `gz_mm_el2pgt_map/unmap`,
  `create_hyp_mmu`, `write_mmu_pt_base_addr`, `gz_dump_vm_tbl`
  (a table-dump primitive by name — trigger TBD).
- Dynamic regions: `gz_mmap_add_dynamic_region*` (incl. specific_valloc),
  `gz_mmap_remove_dynamic_region*`, `gz_vmmap_remap_region*`,
  `gz_vmmap_unmap_region*`, `sys_mmap`.
- Gatekeepers (callers not yet traced): `is_addr_in_emimpu_region`,
  `is_addr_in_mblock_region` — the checks a future preset would NOP.
- Sharing (live-proven in gz_log): `hyp_pmm_reg_share_region`
  (iova `0xfc000000`+64M, `0x1a/1c…`+512M, `0x1e/1f…`+256M),
  `iommu_set_iova_share_region`, `hyp_set_cma_region`, `share_iova`,
  `mtee_mem_srv_body/ioctl` (`TZCMD_MEM_SHAREDMEM_REG` succeeding live),
  KTA `AllocMem/ReferenceMem/UnreferenceMem` handles, `/dev/gz_kree`.
- IOMMU introspection: `hyp_iommu_iova_to_phys`, `get/parse_pte`,
  `iommu_protect_bank`, `map_protect_bank`, `pmm_create_hal_mtk_iommu`.
- Ranges: `hyp_pmm_secure/unsecure_range`, `map_mmio_regions`,
  `platform_init_mmu_mappings`, `platform_mtksmmu_protpgd`,
  `all_mem_region` / `oem_all_mem_region` lists (static map data — parse next).

## Live policy (from gz_log, not static)

- RKP `unmap2`: 33 regions stripped from kr/lk/gz stage-2 (PERM:0):
  `0x70000000`+64M, `0xd0000000`+44M, `0xd3000000`+, `0xd4000000`+54M,
  `0x9fe70000`+, `0x7ec00000`+18M, `0xed780000`+, `0x7fe70000`+.
- Modem memory policy is silent: zero md1/modem/smem strings in the log.

## Open RE targets (in order)

1. Static region lists (`all_mem_region`/`oem_all_mem_region`) — parse
   addresses/sizes/perms from image data.
2. ~~Gatekeeper callers~~ Range hook VALIDATED live (2026-09-07):
   `0x200E4 B.HI` + `0x200EC B.LO` → deny(log)+`-8`; NOPs fall to map call.
   `--preset gz-range` (8 bytes, VALID) flashed to `gz_b`, slot B booted to
   fastboot with ramdump ungated — no regression. Effect (forced pass on
   arbitrary requests) still needs a caller path (hypercall/KREE).
3. SiP `0x8200FF03` callee semantics + `gz_dump_vm_tbl` trigger.
4. KREE ioctl ABI (`/dev/gz_kree`) for no-flash share requests.
