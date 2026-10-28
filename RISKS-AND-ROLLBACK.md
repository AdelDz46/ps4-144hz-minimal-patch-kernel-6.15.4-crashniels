# Risks, test plan and rollback

**This candidate is not hardware-verified.** A successful build proves only
that the code compiles and links. No console was touched.

## Known technical risks

1. **Bridge timing path is unproven.** The GPU CRTC is programmed with the
   144 Hz custom timing, but the MN864729 enable sequence is fed VIC 63
   (1080p120). If the bridge regenerates HDMI timing from that code, output may
   stay at 120 Hz, or the link may fail. This is the central unknown.
2. **Non-CEA AVI InfoFrame.** VIC 63 is advertised in the InfoFrame while the
   real timing is non-CEA. Some sinks reject or mis-handle that combination.
3. **EDID/display dependence.** The mode is offered unconditionally from
   `get_modes()`, even if the attached display does not support 144 Hz. Manual
   selection is required; the 60 Hz preferred mode remains default.
4. **No fallback automation.** If the mode is selected and the display loses
   sync, recovery depends on the boot loader/kernel fallback path.
5. **Toolchain authenticity.** The LLVM archive checksum matched, but its GPG
   signature verified as BAD. Provenance rests on the checksum only.
6. **Config path deviation.** `CONFIG_EXTRA_FIRMWARE_DIR` was redirected to a
   local copy. Firmware list and content hashes are recorded, but this differs
   from the original build environment path.

## What this build does *not* provide

- No installable boot package, no `initramfs`, no loader configuration.
  `CONFIG_MODULES=y` but no `=m` symbols are enabled, so the kernel is
  self-contained and `bzImage` is the only kernel artifact needed — matching
  the known-working 1080p120 release, which also ships only `bzImage`.
- No change to the existing 120 Hz kernel, which remains the fallback.

## Safe test plan (requires explicit user approval; not performed)

1. Keep the known-working 120 Hz `bzImage` untouched and bootable.
2. Copy the new `bzImage` under a **new, distinct name** on the test media.
3. Boot it, stay in TTY (do not start the graphical session), and confirm the
   kernel release string `6.15.4-ps4-144hz-minimal-6154-crashniels`.
4. Check `dmesg | grep -i ps4_bridge` and `modetest`/`drm_info` for the offered
   1920x1080 mode set (should list 60, 120 and the 144 Hz timing).
5. Select 144 Hz manually. Record: link up/down, reported refresh, pixel clock,
   and whether the bridge logs any error.
6. Watch for the previously observed symptom: signal in TTY, then signal loss
   when the graphical session starts.

## Rollback

1. Power off, restore the original boot entry / `bzImage` from the unchanged
   1080p120 release directory (SHA-256 recorded below).
2. If only the new image was used, no files on the original release were
   modified, so rollback is simply selecting the old boot entry.
3. Never overwrite the known-working image in place.

Known-working fallback image (untouched):
`/home/qba/linux files/6.15.4.src-crashniels_KHEOPS_FullLTO-v2.1_ntsync_1080p120Hz_belize/bzImage`

## Evidence to collect on a future hardware test

- Full `dmesg` from boot through mode set.
- Bridge register readback if a diagnostic build is approved separately.
- Display OSD/refresh readout and a photo of the result.
