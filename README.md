# PS4 144Hz Minimal Patch kernel 6.15.4 Crashniels

Minimal 1920x1080@144Hz experimental kernel candidate for PS4 Pro
(MN864729 bridge) based on the original
[crashniels/linux](https://github.com/crashniels/linux) `ps4-linux-6.15.y`
tree used by the known-working 1080p120 KHEOPS kernel.

## Status

- Source: `crashniels/linux`, branch `ps4-linux-6.15.y`,
  commit `22a65a956e3421b55ad3451f58c053c6467f729f`.
- Config: the `.config` shipped with the known-working
  `6.15.4.src-crashniels_KHEOPS_FullLTO-v2.1_ntsync_1080p120Hz_belize` release.
- Patch: `0001-ps4-144hz-minimal.patch`,
  SHA-256 `8d3410bf868c8bcc29a0aafa130eb8c848bc621173639e82d2687e5c66f3f464`.
- Build: one out-of-tree `bzImage` build, recorded in `BUILD.md`.
- **Not hardware-verified.** A successful compile does not prove 144 Hz
  works on the console. See `RISKS-AND-ROLLBACK.md`.

## Patch scope

The patch touches only `drivers/gpu/drm/amd/amdgpu/ps4_bridge.c`:

1. Adds `mode_1080p144`, the exact EDID detailed timing:
   1920x1080, 333210 kHz, H 1920/1928/1960/2000,
   V 1080/1095/1103/1157, `+hsync -vsync` (143.9974 Hz).
2. Exposes it from `ps4_bridge_get_modes()` after 60 Hz and 120 Hz; it is not
   marked preferred, so 60 Hz stays the default.
3. Accepts exactly that timing in `ps4_bridge_mode_valid()` through an
   exact-field comparison; no arbitrary non-CEA mode is admitted.
4. In `ps4_bridge_mode_set()`, marks the mode and reuses the existing VIC 63
   (1080p120) bridge path, so `ps4_bridge_enable()` no longer aborts on VIC 0.
   The two VIC-derived register writes (`0x7217`, `0x7227`) then use 63.

Unchanged: 60 Hz and 120 Hz mode definitions, CEA VIC 16/4/63 handling, the
vendor check, DP transmitter enable/disable, DP training, PLL/clock selection,
DPM, and every other file.

## Layout

| Path | Purpose |
| --- | --- |
| `source/` | Pinned crashniels tree (local only, not committed) |
| `config.base` | Unmodified known-working `.config` |
| `build/` | Out-of-tree build output (local only) |
| `firmware/` | The 12 files in `CONFIG_EXTRA_FIRMWARE` + licenses + hashes |
| `0001-ps4-144hz-minimal.patch` | The only source change |
| `PROVENANCE.md` | Source/config/firmware/toolchain identity and hashes |
| `AUDIT.md` | Phase 1 findings, timing verification, go/no-go |
| `BUILD.md` | Build command and artifact hashes |
| `RISKS-AND-ROLLBACK.md` | Test and rollback plan |
| `.github/workflows/build.yml` | Manual, hash-pinned CI build |

## Build

```sh
git init source
git -C source remote add origin https://github.com/crashniels/linux
git -C source fetch --depth 1 origin 22a65a956e3421b55ad3451f58c053c6467f729f
git -C source checkout --detach FETCH_HEAD

git -C source apply --check ../0001-ps4-144hz-minimal.patch
git -C source apply ../0001-ps4-144hz-minimal.patch

mkdir build
cp config.base build/.config
sed -i 's|^CONFIG_LOCALVERSION=.*|CONFIG_LOCALVERSION="-ps4-144hz-minimal-6154-crashniels"|' build/.config
sed -i "s|^CONFIG_EXTRA_FIRMWARE_DIR=.*|CONFIG_EXTRA_FIRMWARE_DIR=\"$PWD/firmware\"|" build/.config

make -C source O="$PWD/build" ARCH=x86_64 LLVM=1 LLVM_IAS=1 olddefconfig
make -C source O="$PWD/build" ARCH=x86_64 LLVM=1 LLVM_IAS=1 -j1 bzImage
```

Requires LLVM/LLD 19.x and `bc`. The CI workflow uses `LLVM=-19` with
clang/lld from apt.llvm.org. See `BUILD.md` for the recorded local run.
