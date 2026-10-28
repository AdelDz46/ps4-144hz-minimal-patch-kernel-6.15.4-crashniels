# Provenance

All hashes are SHA-256.

## Kernel source

| Item | Value |
| --- | --- |
| Remote | `https://github.com/crashniels/linux` |
| Branch | `ps4-linux-6.15.y` |
| Commit | `22a65a956e3421b55ad3451f58c053c6467f729f` |
| Commit date | 2026-02-22 15:48:30 +0100 |
| Commit subject | `Enable OpenVPN support` |
| Local path | `source/` (not committed; CI fetches the pinned commit) |

Why this is the correct baseline:

- The known-working release `6.15.4.src-crashniels_KHEOPS_FullLTO-v2.1_ntsync_1080p120Hz_belize`
  ships `update_version 2.1.txt`, which states:
  `Kernel source 6.15.4 crashniels aeolia-belize`
  and links `https://github.com/crashniels/linux/tree/ps4-linux-6.15.y`.
- `source/` is a clone whose `origin` is exactly that remote and whose HEAD is
  the tip of that branch (`22a65a956...`). It carries no local diagnostics
  patches (unlike the lab trees used for the TX experiments).
- The shipped release reports `6.15.4.src-crashniels-KHEOPS-FullLTO-v2.1_ntsync_1080p120hz`,
  whose base version `6.15.4` matches `Makefile` at this commit.

The much later `feeRnt/ps4-linux-12xx` tree and its consolidated 0010/0011
patches are **not** used anywhere in this project.

## Kernel config

| Item | Value |
| --- | --- |
| File in repo | `config.base` |
| Origin | `.config` inside the known-working 1080p120 release directory |
| SHA-256 | `be4668eb3059eac23a86569cd6fbbf467da54ef9a6cf9c60f19677f0f63b31c0` |
| Original `CONFIG_LOCALVERSION` | `.src-crashniels-KHEOPS-FullLTO-v2.1_ntsync_1080p120hz` |

`CONFIG_X86_PS4=y` and `CONFIG_DRM_AMDGPU=y` are retained unchanged.

### Candidate-only config deltas

Applied only to `build/.config` (never to `config.base`):

1. `CONFIG_LOCALVERSION="-ps4-144hz-minimal-6154-crashniels"`
2. `CONFIG_EXTRA_FIRMWARE_DIR="<project>/firmware"` — the shipped config
   contains `/home/saya/Bureau/.../extra_firmware`, an absolute path from the
   original builder's machine. Only the directory path is changed; the
   `CONFIG_EXTRA_FIRMWARE` file list is untouched.

## Firmware

Source of the local copy: `/home/qba/ps4-kernel-build-preflight/build-inputs/firmware`
(read-only copy; that repository was not modified). The 12 files listed in
`CONFIG_EXTRA_FIRMWARE` were copied to `firmware/`:

```
d3ecac0fde5b66fab282cd97f43c04d77bf4fd6abdbd9c35c5d3d40e2f1f035a  EEPROM_MT7668.bin
a042c6ceb77ae4a0c367e5e5522cbf21f767936b8799a781096e1e72684d957c  EEPROM_MT7668_e1.bin
49fa3e52b26fe49b837187c91765550e32cd6540ea2e09343274245d6630062a  TxPwrLimit_MT76x8.dat
35faff6a613fefcbe33a6f6d8df885be76c546f9e78ba8ab93056a816ad87254  WIFI_RAM_CODE2_SDIO_MT7668.bin
b6c6b63f021168a327a0600119a416c4f5f5198820a3267b54a1259c19da0baf  WIFI_RAM_CODE2_USB_MT7668.bin
977724f3eef0e6f27567a6684bf616b884b82fe948bde0a436ea546a1f9c83ee  WIFI_RAM_CODE_MT7668.bin
465ed634c61b46b34bb21240736f33d3d347fefec7cbfbd1cb6e63334aec4cc9  mediatek/mt7668pr2h.bin
c5b84ce8f70072f9f11173f448d574655ff2506595cb8dc113f6d8110272ad66  mrvl/sd8797_uapsta.bin
b04fee35702d765ca4d84e321ad4cf814b7c8fb9b2aa6784e9154cb06df0cc4a  mrvl/sd8897_uapsta.bin
519083ebb8318d9cfc8eaaaa7470d1a266c8f32275df3c62e3619d6c07f97389  mt7668_patch_e1_hdr.bin
05a619651517ee481ccfa3ed2cd11bd33f497967802233e6f81085036e0ec9d5  mt7668_patch_e2_hdr.bin
828d7d18607c78513a71719f919340a7c91ee8ef68080ad7270a65dd661abbbd  wifi.cfg
```

## Toolchain

| Item | Value |
| --- | --- |
| Archive | LLVM 19.1.7 `clang-built-linux` x86_64 tarball |
| Archive SHA-256 (verified) | `c9f2cd6997bc17fdd241a3594d77ba22d9f4921ef69ac1e14ad86fd6522f18bb` |
| GPG signature | **BAD signature** (key was not authenticated) — checksum match only, not authenticity |
| `clang --version` | `ClangBuiltLinux clang version 19.1.7 (cd708029e0b2869e80abe31ddb175f7c35361f90)` |
| `ld.lld` | LLVM 19.1.7, verified working |
| Host `bc` | GNU bc 1.08.2 from a local package directory (system `bc` absent) |

## Candidate patch

| Item | Value |
| --- | --- |
| File | `0001-ps4-144hz-minimal.patch` |
| SHA-256 | `050e68e1472257a7d4c80936f1716d7eb5492d28c769a4fbb8ed0364d41fd9b9` |
| Size | 126 lines, 1 file, +47 / -10 |
| Applies to | pristine `22a65a956e3421b55ad3451f58c053c6467f729f` (`git apply --check` passes) |
