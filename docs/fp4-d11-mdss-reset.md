# D11 root cause: the MDSS core reset freezes the SoC on the live splash

This is the full record of how D11 ("the phone drops into the Qualcomm ramdump
after reboots") was tracked down on 2026-10-09, so that neither the result nor
the method has to be rediscovered. The defect entry is
[`fp4-defects.md#d11`](./fp4-defects.md#d11); this file is the long form.

## TL;DR

- **Symptom:** on roughly 1 boot in 4, the FP4 never reaches userspace. About
  30 s later the kernel watchdog bites and the phone enumerates as `05c6:900e`
  (Qualcomm Sahara, MEMORY_DEBUG mode) instead of booting.
- **Where:** at ~0.77 s into boot, inside `mdss_probe()` ->
  `msm_mdss_init()` -> `msm_mdss_reset()` (`drivers/gpu/drm/msm/msm_mdss.c`),
  which asserts the display core reset `DISP_CC_MDSS_CORE_BCR`, sleeps 20 ms
  and deasserts it.
- **Why (best explanation):** the bootloader leaves the display running (its
  splash is still being scanned out of RAM by the MDP when Linux probes MDSS,
  and simpledrm is bound to that framebuffer). Resetting the display core with
  a scanout in flight wedges the interconnect; every CPU stalls on its next
  bus access, nothing more is logged, and only the watchdog gets out.
- **Proof:** same kernel and cmdline, `resets` property deleted from the MDSS
  DT node so `msm_mdss_reset()` is a no-op: **20 of 20 boots clean**, against
  3 ramdumps in the first ~5 boots with the reset in place (and 6 in 24, ~1 in
  4, on the normal cmdline). 20 clean in a row is ~0.3 % likely at 1 in 4.
- **But the reset is load-bearing:** without it the panel never lights up
  (backlight on, nothing drawn; DSI command DMA times out). So the fix is to
  make the reset safe, not to remove it. See [Fix](#fix).

## Timeline of evidence

| step | image | boots | ramdumps | where the crashed kernel stopped |
| --- | --- | --- | --- | --- |
| 1 | ramoops kernel (7.2.0-11), normal cmdline | 3 (loop 1) | 3 in a row | `3d6a000.gmu: Adding to iommu group 12` |
| 2 | same | 20 (loop 3) | 1 | same line |
| 3 | same | 24 (loop 4) | 6 | same line, all 6 (8 of 8 total) |
| 4 | same kernel, `ignore_loglevel` + `dyndbg` cmdline | ~5 | 3 | `ae00000.display-subsystem: really_probe: probing driver msm-mdss with device`, all 3 |
| 5 | step 4 + `resets` deleted from the MDSS node | 20 | **0** | -- (display dead, see below) |

The crash rate wanders between runs (1 in 20 to 3 in 5); it is a race, and the
debug cmdline (which slows early boot with console output) made it more likely,
not less.

## How the crashed kernel was read

Nothing on a running system sees this crash: journald is not up at 0.77 s, and
the next boot's journal starts fresh. The chain that worked:

1. **ramoops.** `CONFIG_PSTORE_RAM=y`, `PSTORE_CONSOLE=y`, `PSTORE_PMSG=y`
   (kernel pkgrel 11). The 7.2.y `sm6350.dtsi` *already* reserves
   `ramoops@ffc00000` (1 MiB, `no-map`, console 256 KiB at +0xa0000, pmsg 128
   KiB, ecc 16); only the config was missing. A second node we added at
   `0xb0000000` failed probe with `-22` ("already initialized") and was dropped.
2. **Warm-reboot pstore is NOT reliable here.** After a clean reboot,
   `/sys/fs/pstore/console-ramoops-0` came back with
   `ECC: 237 Corrected bytes, 873 unrecoverable blocks`: something between
   kernels (the bootloader) scribbles on `0xffc00000`.
3. **Read RAM straight out of the ramdump instead.** A watchdog bite drops the
   SoC into Sahara MEMORY_DEBUG with DDR intact. `scripts/d11/sahara-dump.py
   read 0xffc00000 0x100000 out.bin` does one Sahara session (HELLO, HELLO_RESP
   mode 2, `MEMORY_READ_64` = command 0x11, reads of at most 1 MiB, a USB reset
   restarts a wedged session) and then `scripts/edl-reset.py` sends Sahara RESET
   so the phone reboots without anyone touching it.
   `scripts/d11/ramoops-console.py out.bin` decodes the console zone (header
   `DBGC`, start, size, then the text).
4. **The printk ring adds nothing, and why.** `__log_buf` was read too (its
   physical address is `phys(_text) + (__log_buf - _text)`; `_text` and
   `__log_buf` from `/proc/kallsyms`, `phys(_text)` = start of "Kernel code" in
   `/proc/iomem`; `0xa3d72e80` on 7.2.0-11, stable across boots because ABL
   loads the kernel at a fixed physical address and arm64 KASLR is virtual
   only). It holds *less* than ramoops: it is cached memory and a watchdog bite
   does not write caches back, while ramoops maps its region uncached. So
   ramoops is the faithful record, and nothing at all was logged after its
   last line.
5. **Narrowing with dyndbg (no rebuild).** The boot.img v0 header keeps the
   cmdline in a fixed 512-byte field (offset 64) that the image-id hash does not
   cover, so the field was patched in place, kernel and DTB byte-identical:

       ... deferred_probe_timeout=60 ignore_loglevel dyndbg="file drivers/base/dd.c +p; file drivers/gpu/drm/msm/msm_mdss.c +p; file drivers/iommu/arm/arm-smmu/arm-smmu.c +p"

   `dd.c` logs every `really_probe` start and bind; `msm_mdss.c` has one
   `dev_dbg` right after the reset (`mapped mdss address space`).
6. **The loop.** `scripts/d11/reboot-loop.sh` reboots over SSH, and on a
   ramdump reads ramoops + `__log_buf` over Sahara before `edl-reset`, prints the
   crashed console tail, and continues.

## What the crashed kernels show

On a good boot (dyndbg), the sequence after the GPU SMMU probe is:

    arm-smmu 3d40000.iommu: ... preserved 0 boot mappings
    platform 3d00000.gpu: Adding to iommu group 11
    platform 3d6a000.gmu: Adding to iommu group 12          <- normal-cmdline crashes end here (8/8)
    arm-smmu 3d40000.iommu: driver: 'arm-smmu': driver_bound: bound to device
    platform 3d00000.gpu: Retrying from deferred list
    adreno 3d00000.gpu: driver: 'adreno': driver_bound: bound to device
    platform ae00000.display-subsystem: Retrying from deferred list
    platform ae00000.display-subsystem: bus: 'platform': really_probe: probing driver msm-mdss with device
                                                             <- dyndbg crashes end here (3/3)
    msm-mdss ae00000.display-subsystem: mapped mdss address space @...
    ... (MDSS populates DPU / DSI / panel children)
    msm_dsi_phy ae94400.phy: [drm:dsi_pll_10nm_vco_prepare] *ERROR* DSI PLL(0) lock failed
    WARNING: ... clk_core_disable ... dsi0_phy_pll_out_dsiclk already disabled

The successful bind of the GPU SMMU is what kicks the deferred-probe list, and
MDSS (deferred until then) probes immediately after: that is why the normal-
cmdline crashes all stop on the GMU line (the console simply had not printed the
dd.c lines, which are not logged without dyndbg). Between `probing driver
msm-mdss` and `mapped mdss address space`, `msm_mdss_init()` does only:

1. `msm_mdss_reset()`: `reset_control_get_optional_exclusive`, assert,
   `msleep(20)`, deassert (`resets = <&dispcc DISP_CC_MDSS_CORE_BCR>` in
   `sm6350.dtsi`);
2. `devm_kzalloc`, `qcom_ubwc_config_get_data()`, `of_device_get_match_data()`
   (no hardware access);
3. `devm_platform_ioremap_resource_byname()` (no hardware access).

So the freeze is in the reset. Note: the journal's kernel timestamps cannot
resolve the 20 ms hold (on a good boot it shows 0.13 ms between the two lines)
because journald reads kmsg late and in batches; use `dmesg` or the ramoops
console for timing.

## The no-reset test, and why it is not the fix

Image: the dyndbg boot.img with `resets` removed from
`/soc@0/display-subsystem@ae00000` in the appended DTB:

    # boot.img v0: kernel section = Image.gz + appended DTB (FDT magic d00dfeed,
    # found where offset + totalsize == section end)
    fdtput -d noreset.dtb /soc@0/display-subsystem@ae00000 resets
    image/boot/android-image.py bootimg --kernel Image.gz --dtb noreset.dtb \
        --cmdline "<same cmdline>" --pagesize 4096 --out boot.img

(Repacking the *unmodified* DTB with the same tool reproduced the flashed image
byte for byte, so the only difference was that one property.)

Result: 20/20 boots clean. But:

- The panel stays dark with the backlight on. Hyprland runs, DRM reports
  DSI-1 connected and enabled, and the DSI host cannot talk to the panel:

      dsi_cmds2buf_tx: cmd dma tx failed, type=0x39, data0=0xb9, len=8, ret=-110
      panel-himax-hx83112a ae94000.dsi.0: sending dcs data b9 83 11 2a failed: -110

- The `DSI PLL(0) lock failed` error and the `already disabled` clk WARN that
  appear on **every** normal boot are absent without the reset.

So the reset is what lets Linux re-initialise a display the bootloader left
running; it just must not be asserted under a live scanout.

## Upstream context

- The reset was added by Bjorn Andersson in 2022, "drm/msm/dpu: Issue MDSS
  reset during initialization", precisely because the bootloader's splash
  leaves the MDSS half-configured
  ([v4](https://lkml.rescloud.iu.edu/2204.2/06017.html),
  [commit](https://gitlab.freedesktop.org/lumag/msm/-/commit/3e9c146f4997e32d257215b0036b2425c6053ffe)).
- A September 2025 MSM8939 thread notes that a live splash used to be masked
  when DRM_MSM was a module, because before v6.17 the MDSS power domain was
  turned off until the module loaded, clearing the hardware state
  ([thread](https://lkml.iu.edu/2509.1/04530.html)). This kernel has
  `DRM_MSM=y` on 7.2, so the reset meets a live scanout.

## Fix

Status: in progress. The approach: before asserting the MDSS core reset, stop
the scanout and let in-flight fetches drain. See the defect entry for the
current state of the patch and its test results.

## Operational lessons from the day

- **Idle suspend is not a network problem.** The phone suspends after 10 min
  idle (s2idle); it then drops off WiFi, and the USB gadget may stay enumerated
  while the CPU ignores serial input (writes to `/dev/ttyACM0` hit EAGAIN).
  Two "network hiccups" were this. For long remote loops, mask the sleep targets
  for the duration and unmask after:
  `sudo systemctl mask sleep.target suspend.target hibernate.target hybrid-sleep.target suspend-then-hibernate.target`.
- **After a fresh boot WiFi can take minutes** before SSH answers; the loop
  waits up to 10 min and re-scans the /24 in case the lease moved.
- **The serial console needs a raw host tty.** With the host's line
  discipline in cooked mode, the phone's output (including bash's OSC 3008
  escape sequences) is echoed back into the phone's shell and garbles every
  command. `tty.setraw()` on `/dev/ttyACM0`, then Ctrl-C, Ctrl-U and `set +H`,
  makes it usable.
- **Keep dumps out of the repo.** ramoops and `__log_buf` contain the kernel
  command line (serial number, MACs); the loop writes them outside the tree.
