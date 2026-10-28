# Audit report

Date: 2026-10-28. Scope: read-only reconnaissance of the local lab, source
selection, minimal patch review, and one build. No console access.

## 1. Source candidates (Phase 1)

All `ps4-144hz-lab/source/*` trees are at the same commit
`22a65a956e3421b55ad3451f58c053c6467f729f` (`ps4-linux-6.15.y`), but they are
not equivalent:

| Tree | `origin` | Dirty | Content |
| --- | --- | --- | --- |
| `linux-copy` | `https://github.com/crashniels/linux` | clean | **Selected baseline** |
| `linux` | `/home/qba/ps4src` (local) | 8 entries | lab work + diagnostics |
| `lane-probe-copy` | `/home/qba/ps4src` | 6 entries | TX lane-probe experiment |
| `public-check-copy` | `/home/qba/ps4src` | dirty | experiment copy |
| `public-check-copy2` | `/home/qba/ps4src` | dirty | experiment copy |

Only `linux-copy` has the real upstream remote. The others are local clones of
`ps4src` and contain experimental edits (e.g. `ps4diag:` `DRM_INFO` logging,
transmitter probes). `ps4src` itself is a reference tree, not a remote clone.

Selection rationale:

- The known-working release's `update_version 2.1.txt` names
  `https://github.com/crashniels/linux/tree/ps4-linux-6.15.y` as its source.
- `linux-copy` is a direct clone of that URL at the branch tip and is clean.
- The shipped release kernel string (`6.15.4.src-crashniels-KHEOPS-FullLTO-v2.1_ntsync_1080p120hz`)
  is consistent with 6.15.4 source.

**Go.** Baseline identity is proven by remote + commit + release notes.

## 2. Quarantined material (not used)

- Consolidated `0010`/`0011` patches made for `feeRnt/ps4-linux-12xx` — different
  tree, different VIC policy (0011 removes 1080p120).
- TX diagnostic / lane-status probe patches (`0001`–`0004`, `0002b`,
  `RESTORE_ENCODER_LANE_PROBES.patch`) — diagnostic, not minimal.
- `dp_extclk`, `dp_clock` flooring, TX disable/enable skip, DP-training bypass,
  bridge register probing, >140 Hz blanket fixups, refactoring — none included.
- The `feeRnt/ps4-linux-12xx` source tree — not referenced by this project.

Historical `ps4-144hz-lab` sources, patches and outputs were left untouched.
The completed TX lane-probe build output was not read into, reused, or modified.

## 3. Timing verification (independent of the prompt)

Parsed the first detailed timing descriptor of the 1080p144 EDID
(`/home/qba/ramdisk/lib/firmware/edid/1080p144v2.bin`):

| Field | EDID | Patch mode |
| --- | --- | --- |
| Pixel clock | 333210 kHz | 333210 |
| H active / sync start / end / total | 1920 / 1928 / 1960 / 2000 | same |
| V active / sync start / end / total | 1080 / 1095 / 1103 / 1157 | same |
| Flags | `+H -V` (0x1a) | `PHSYNC \| NVSYNC` |

Refresh: `333210000 / (2000 × 1157) = 143.9974 Hz` (exact; rounds to 144.00 Hz).
The patch's exact-field matcher therefore admits only this EDID timing.

## 4. Patch review

`0001-ps4-144hz-minimal.patch` (1 file, +44/−5):

1. `struct ps4_bridge`: adds one `bool custom_mode_valid` flag.
2. `ps4_bridge_is_1080p144()`: exact match on all timing fields and both sync
   flags; no range/`>=` comparisons, so no other mode is admitted.
3. `ps4_bridge_mode_set()`: for a non-CEA mode that matches exactly, sets the
   flag and stores existing VIC 63 (1080p120) as the bridge mode. Every other
   non-CEA mode still hits the original `DRM_ERROR` path.
4. `ps4_bridge_enable()`: uses value 63 for the two VIC-derived bridge writes
   (`0x7217`, `0x7227`) when the custom flag is set; otherwise the original
   `mn_bridge->mode` is written. All other register writes are untouched.
5. `ps4_bridge_get_modes()`: appends the 144 Hz mode after 60 Hz and 120 Hz.
   It carries only `DRM_MODE_TYPE_DRIVER`, so the 60 Hz preferred mode is
   unchanged.
6. `ps4_bridge_mode_valid()`: returns `MODE_OK` only for the exact timing;
   the original VIC allowlist (16/4/63) is untouched.

Not changed: 60 Hz and 120 Hz mode definitions, CEA handling, DP transmitter,
DP training, PLL/clock selection, DPM, vendor check, any other file.

Patch hash: `8d3410bf868c8bcc29a0aafa130eb8c848bc621173639e82d2687e5c66f3f464`.
`git apply --check` and `git apply` both verified against a pristine copy of
the pinned commit; resulting file hash matches the worktree exactly.

## 5. Config review

`config.base` is the known-working `.config` (SHA-256 `be4668eb...`). It keeps
`CONFIG_X86_PS4=y`, `CONFIG_DRM_AMDGPU=y`, `CONFIG_MODULES=y` and the original
`CONFIG_EXTRA_FIRMWARE` list.

Two candidate-only changes in `build/.config`:

- `CONFIG_LOCALVERSION="-ps4-144hz-minimal-6154-crashniels"`
- `CONFIG_EXTRA_FIRMWARE_DIR` redirected from the original builder's
  `/home/saya/...` path to this project's `firmware/`.

**Finding:** without the path fix the build cannot complete, because
`CONFIG_EXTRA_FIRMWARE` files are compiled into the kernel from that directory.
The 12 required files were copied read-only from
`ps4-kernel-build-preflight/build-inputs/firmware` and hashed (see
`PROVENANCE.md`).

## 6. Go/no-go

**Go**, with caveats:

- Source and config identity are proven.
- Patch is minimal, reversible, and confined to `ps4_bridge.c`.
- Build feasibility was already demonstrated by an earlier successful build of
  the same tree with the same toolchain (TX lane-probe output).

Caveats (see `RISKS-AND-ROLLBACK.md`):

- The bridge is programmed with the 1080p120 (VIC 63) register path. The GPU
  CRTC runs the 144 Hz timing, but whether the MN864729 forwards/recreates that
  timing is unknown. This patch removes the two software blockers (mode
  validation and VIC-0 enable abort); it does not prove the bridge can carry
  144 Hz.
- No hardware verification was performed and none is claimed.
