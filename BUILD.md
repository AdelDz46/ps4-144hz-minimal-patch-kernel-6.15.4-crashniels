# Build record

One build was run. No duplicate builds were launched.

## Environment

| Item | Value |
| --- | --- |
| Host | `qba@cachyos-ps4` |
| Date | 2026-10-28 (started 15:06:05, finished 17:26:10 +0100) |
| Elapsed | ≈ 2 h 20 min |
| Compiler | `ClangBuiltLinux clang version 19.1.7 (cd708029e0b2869e80abe31ddb175f7c35361f90)` |
| Linker | `ld.lld` 19.1.7 (LLVM 19.1.7 tree) |
| Host tools | `bc` 1.08.2 (local package dir; system `bc` absent) |
| Parallelism | `-j1` |

## Commands

```sh
export PATH=<bc-dir>:<llvm-19.1.7>/bin:$PATH
export LD_LIBRARY_PATH=<llvm-19.1.7>/lib:<llvm-19.1.7>/lib/x86_64-unknown-linux-gnu

# source is a worktree at the pinned commit with the patch applied
git -C source apply ../0001-ps4-144hz-minimal.patch

cp config.base build/.config
sed -i 's|^CONFIG_LOCALVERSION=.*|CONFIG_LOCALVERSION="-ps4-144hz-minimal-6154-crashniels"|' build/.config
sed -i "s|^CONFIG_EXTRA_FIRMWARE_DIR=.*|CONFIG_EXTRA_FIRMWARE_DIR=\"$PWD/firmware\"|" build/.config

make -C source O="$PWD/build" ARCH=x86_64 LLVM=1 LLVM_IAS=1 olddefconfig
make -C source O="$PWD/build" ARCH=x86_64 LLVM=1 LLVM_IAS=1 -j1 bzImage
```

Log: `build/build.log` (run with shell redirection, no pipeline that could mask
the exit code). Log ends with:

```
  OBJCOPY arch/x86/boot/vmlinux.bin
  BUILD   arch/x86/boot/bzImage
Kernel: arch/x86/boot/bzImage is ready  (#1)
make[1]: Leaving directory '.../build'
make: Leaving directory '.../source'
```

Result: **success**, 0 `error:` lines.

## Resulting kernel

| Item | Value |
| --- | --- |
| Release | `6.15.4-ps4-144hz-minimal-6154-crashniels+` |
| Build string | `6.15.4-ps4-144hz-minimal-6154-crashniels+ (qba@cachyos-ps4) #1 SMP PREEMPT_DYNAMIC Wed Oct 28 17:24:36 CET 2026` |
| `bzImage` | `/home/qba/ps4-144hz-minimal-patch-kernel-6.15.4-crashniels/build/arch/x86/boot/bzImage` |
| `bzImage` size | 16,729,088 bytes |
| `bzImage` SHA-256 | `c5845a3e668e3d364ca51781ea1eda86802c743265b266cb2656fb12845c6c80` |
| `bzImage` type | Linux kernel x86 boot executable, bzImage |
| `vmlinux` size / SHA-256 | 63,135,224 bytes / `6579452211e6f7a23028cf4df89a59edc016ba2067571ee22a33d8dc36eb0e74` |
| `System.map` size / SHA-256 | 4,868,033 bytes / `5a4fd1f2fe846abada0ec79bb433ef6f47c3276c05bbf6cd333a153b9e79c334` |
| Resolved `build/.config` SHA-256 | `2683c18d830f23a60b1050b210ab9e91dfdb457af663f9ec804b04df0d8e8791` |

The `+` suffix is the kernel's standard "tree has local modifications" marker,
because the patch is applied on top of the pinned commit. It is not
`CONFIG_LOCALVERSION_AUTO` (which is off).

## Warnings

42 warning lines, all in pre-existing code categories:

- `ps4.h:32` — `struct pci_dev` visibility (vendor code).
- `ps4_bridge.c` — `no previous prototype` for `ps4_bridge_mode_set`,
  `get_modes`, `detect`, `mode_valid`, `register`; unused `mode_480p`,
  `mode_720p`. These already exist in the unpatched driver.
- 3 benign Perl locale warnings.

No warning originates from the added `ps4_bridge_is_1080p144()` helper, the new
`mode_1080p144` table, or any added logic. `CONFIG_WERROR` is not set.

## Modules

`CONFIG_MODULES=y` but the resolved config contains **zero `=m` symbols**, so
`bzImage` is the only kernel artifact needed — matching the known-working
1080p120 release, which also ships only `bzImage`. No `modules` target output
is required for this configuration.

## Reproducibility notes

- Source, patch and config are pinned by hash (see `MANIFEST.txt`).
- The local LLVM archive checksum matches but its GPG signature verified BAD;
  the toolchain is trusted only by checksum here.
- The CI workflow uses clang/lld 19 from apt.llvm.org, not this exact archive,
  and embeds different absolute paths and build timestamps. A CI `bzImage` is
  therefore expected to differ byte-for-byte from the hash above.
- No hardware test was performed. Compile success is not evidence of 144 Hz
  operation.
