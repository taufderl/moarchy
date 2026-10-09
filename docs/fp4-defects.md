# Fairphone 4 — open defects

Things reproduced on a real handset and not yet fixed. The closed half moved to
[`fp4-fixes.md`](./fp4-fixes.md) — nine entries, kept because several of them
record why something was wrong for a reason that was not the obvious one.

The camera and the sensors both left this file on 2026-09-24 — see D6, D7 and
D24 in the fixes. Both had been recorded as fixed once already and were not,
which is the reason each entry there keeps the wrong turns as well as the fix. D5, D12, D25 and D27 left on 2026-09-29; see the fixes.

Sources: what the handset does, the
[postmarketOS wiki](https://wiki.postmarketos.org/wiki/Fairphone_4_(fairphone-fp4))
for what upstream considers working, and the handset owner.

**Status vocabulary.** `OPEN` — reproduced, not fixed. `OPEN — important` —
blocks ordinary use. `UNSUPPORTED` — no driver exists; needs writing, not
configuring. `WONTFIX` — understood and deliberately left.

| id | what | status |
| --- | --- | --- |
| [D8](#d8) | GPS runs but never reaches a fix; no A-GPS assistance | **OPEN** |
| [D11](#d11) | The phone drops into EDL after repeated reboots | **OPEN — important** |
| [D19](#d19) | The fingerprint reader is an Egis part with no Linux path | **UNSUPPORTED** |
| [D21](#d21) | Bluetooth carries music but not call audio | **OPEN** |
| [D22](#d22) | The touchscreen controller logs recurring i2c failures | **OPEN** |
| [D23](#d23) | The camera's CSI PHY supplies are undescribed, and a clock sticks on | **OPEN** |
| [D29](#d29) | s2idle aborts in ~2s: serial-console RX wakeup (irq 172). Fixed by mainline `d0cd9c8d0fd5` (qcom-geni force-suspend), VALIDATED on FP4 (holds 21s). udev rule is the stopgap until the kernel carries it. Deep-collapse depth cap is a separate open bug | **abort FIXED (mainline fix validated); depth open** |
| [D30](#d30) | The quickshell before-sleep lock traps a locked-password account: it asks PAM, a locked password can never authenticate, so the first sleep locks the user out | **OPEN — important** |
| [D31](#d31) | Some boots have no sound card: the ADSP booted before the PD locator it queries (qcom_pd_mapper) was loaded. Fix applied (modprobe softdep), verifying over more boots | **FIX APPLIED -- verifying** |

---

## What upstream reports

From the [postmarketOS wiki](https://wiki.postmarketos.org/wiki/Fairphone_4_(fairphone-fp4)),
kept here so this file can be read without it. `Y` works, `P` partial, `N` no.

| | | | | | |
| --- | --- | --- | --- | --- | --- |
| flashing `Y` | usbnet `Y` | emmc `Y` | sdcard `Y` | screen `Y` | touch `Y` |
| 3d `Y` | cameraflash `Y` | bluetooth `Y` | gps `Y` | sms `Y` | mobiledata `Y` |
| fde `Y` | otg `Y` | accel `Y` | magnet `Y` | light `Y` | proximity `Y` |
| haptics `Y` | battery `P` | audio `P` | camera `P` | wifi `P` | calls `P` |
| nfc `N` | fingerprint `N` | hdmidp `N` | fossbootloader `N` | | |

Three of those are behind this tree rather than ahead of it. Upstream records
audio as partial ("the built-in microphones are not working", "speaker audio on
the earpiece is distorted") and calls as "possible to place and receive... but
there will be no audio" — all three work here, see
[`fp4-fixes.md`](./fp4-fixes.md) D3, D13, D14, D16, D17.

Upstream was ahead of us on the sensors until 2026-09-24; accelerometer,
magnetometer, light and proximity now work here too (D7, in the fixes).

Upstream's Wi-Fi note matches D5 exactly and is tracked as
[pmaports#2841](https://gitlab.postmarketos.org/postmarketOS/pmaports/-/work_items/2841).

**The parts, as upstream identifies them.** Useful when a driver has to be
found for something:

| | | | |
| --- | --- | --- | --- |
| Display / touch | HX83112A | Amplifier | AW88264(A) |
| Audio codec | WCD9380 | Microphones | AWC2718M06CX |
| Vibration | AW8695 | Earpiece | BM24-10DS/2 |
| Sixaxis | LSM6DSOQTR | Light / proximity | TCS37013H |
| Magnetometer | AK09918 | ToF | VL53L4 |
| Wi-Fi / BT | WCN3988/3990 | NFC | ST21NFCD |
| Charger / fuel gauge | PM7250B | Camera flash | PM6150L |
| Fingerprint | *not identified upstream either* | | |

**Two things worth keeping.** GPS is enabled with

```
mmcli -m any --location-enable-gps-nmea
```

and the serial console is on test points given in Fairphone's own repair
blueprint — `RXD = TP1102`, `TXD = TP1104`, `GND = TP4810`.

---

## D8 — GPS runs but never reaches a fix; no A-GPS assistance {#d8}

**Status: OPEN, REFRAMED 2026-09-28.** The receiver runs and streams NMEA but
never reaches a fix indoors because it has no assistance data. The earlier
conclusion below - that the assistance *indication path* is firmware-dead and
there is "no config-only fix" - is now shown to be **only half right**: the
failure is real, but it is **ModemManager-specific, not firmware**. A direct
libqmi/qmicli client drives the same QMI-LOC assistance queries that MM fails.
Predicted-orbits (xtra) can be *transferred* into the modem in full from
userspace; the one remaining wall is the modem rejecting the assembled xtra at
its final validation step. See *Reframed 2026-09-28* below.

No kernel GNSS device exists and none is needed: the receiver is the modem's,
reached through ModemManager, which reaches the QMI Location service (id 16,
present on qrtr). `mmcli -m any --location-enable-gps-nmea --location-enable-gps-raw`
turns it on, and NMEA then flows -- confirmed 2026-09-24, polling
`--location-get`:

```
$GPGSA,A,1,,,...        automatic mode, fix type 1 = NO FIX
$PQWM1,2437,384234620,1,7,...   Qualcomm proprietary; the receiver is running
(no $GPGSV)             no satellites tracked -- every test has been indoors
```

So the receiver is alive and emitting. Two things stop a fix:

- **No A-GPS assistance.** ModemManager logs, every session:

  ```
  couldn't load supported assistance data types: Failed to receive
  indication with the predicted orbits data source
  ```

  Without predicted orbits / xtra, there is no ephemeris head start, so a
  first fix is a true cold start: 12-15 minutes of open sky, the phone still,
  before `$GPGGA` carries coordinates. GNOME Maps and browser geolocation
  time out long before that, which is what "could not determine exact
  location" was -- not a failure of the receiver, a receiver that had not
  finished yet.

- **A ModemManager crash on rapid toggling.** Cycling location off/on/off/on
  quickly hit `loc_register_events_ready: assertion failed (!priv->loc_client)`
  and MM restarted. A single on or off is safe (verified); the Location quick
  toggle does single actions, so normal use does not trigger it. It is an
  upstream MM bug in the QMI LOC client, worth reporting.

**A-GPS MSB / SUPL was tried directly, 2026-09-24 — it hits the same wall.**
`mmcli --location-status` shows the modem advertises `agps-msa` and `agps-msb`,
not just raw/nmea, so network-assisted GPS (ephemeris fetched from a SUPL
server, phone computes the fix) looked like a path the earlier note missed.
It is not, on this build:

```
mmcli -m any --location-set-supl-server=supl.google.com:7275
  → Aborted: Failed to receive indication with the server update result
mmcli -m any --location-enable-agps-msb ...
  → Aborted: Couldn't enable 'agps-msb': Failed to receive operation mode indication
```

Both fail with the *same* "indication never arrives" signature as the
predicted-orbits error above — and this was **not** a connectivity problem:
the modem was `connected`, packet service `attached` at the
time, so it had its own cellular data path for assistance. `gps-raw` and
`gps-nmea` do enable; only the assistance modes fail. This confirms the root
cause is the modem's QMI-LOC assistance/indication path (izat/xtra), not a
missing config toggle and not the network — the LOC service answers for the
raw receiver but never completes the assistance handshake.

### Reframed 2026-09-28: the assistance query works; injection reaches finalization

The A-GPS tests above were all run *through ModemManager*. Driving QMI-LOC
directly (qmicli / a small libqmi-GIR client over `qrtr://0`, coexisting with
MM via qmi-proxy) tells a very different story. **Every step MM fails, a direct
client completes** - the modem stayed `connected`/`attached` throughout, and
the worst-case recovery is a `systemctl restart ModemManager`, never a reboot.

- **The predicted-orbits data-source query works.**
  `qmicli -d qrtr://0 --loc-get-predicted-orbits-data-source` returns exit 0
  and the live xtra server list:
  `https://path{1,2,3}.xtracloud.net/xtra3Mgrbej.bin`. So the modem's LOC
  assistance path answers a direct client fine - MM's "couldn't load supported
  assistance data types: Failed to receive indication…" is **not** a dead
  firmware handshake.
- **Likely MM root cause: a missing TLV.** The data-source indication carries
  the server list but **omits `allowed_sizes`** (max file / max part size). MM
  needs those to chunk the download, so when they are absent it gives up and
  sets `SupportedAssistanceData = NONE` - which also disables MM's own
  `InjectAssistanceData` D-Bus method. qmicli, meanwhile, exposes no xtra
  inject at all. That is why *both* stock paths dead-end.
- **The xtra file is reachable and current.** `xtra3Mgrbej.bin` is ~64 KiB,
  `Last-Modified` refreshed roughly hourly (a signed XTRA3 file). The filename
  is stable; only the server order rotates.
- **Predicted orbits can be transferred into the modem from userspace.** A
  ~90-line libqmi-GIR injector (`inject_xtra_data`, chunked, lock-step on the
  per-part indication) pushes the whole file: **every part is acknowledged
  `indication_status = SUCCESS`**. The generic `inject_predicted_orbits_data`
  (with `format_type = XTRA`) is **`NotSupported` (QMI error 94)** on this
  firmware; the XTRA-specific `inject_xtra_data` is the supported message.
- **The wall: finalization is a non-functional stub.** Every part transfers
  (per-part indication `SUCCESS`, verified in a `QMI_DEBUG` trace - the sent
  Part Data bytes match the file exactly, so no transit corruption), but the
  **terminal** indication is always `GENERAL_FAILURE (1)` with a **fixed detail
  code 2** and validity stays `missing`. This was isolated exhaustively and the
  result is invariant across: xtra **v1/v2/v3**; part sizes 1024 (the modem's
  hard max - 1025 gives `ArgumentTooLong`) and even-division 997; UTC time
  injected (clock NTP-synced); GNSS engine stopped and started via MM;
  `register_events` for the inject/engine events beforehand; and 1-based part
  numbers (0-based -> `MalformedMessage`). The `detail=2` is constant regardless
  of `total_parts` (41/60/64/65 all report 2), so it is a reason code, not a
  part number. The generic `inject_predicted_orbits_data` (0x0025) is
  `NotSupported` (QMI 94) - the LOC service simply does not implement it.
  Conclusion: on this MPSS build (`MPSS.HI.2.5.1 BITRA`, Nov 2023) the QMI-LOC
  xtra path answers queries and buffers the transfer but its **finalization is
  not implemented** for a mainline client. Android reaches the same modem
  through the proprietary izat/loc-HAL, not this QMI-LOC message, so "Android
  does it" does not translate to a mainline userspace fix here.

**Where this leaves a fix.** Predicted-orbits (xtra) A-GPS is **not reachable
from mainline userspace on this modem firmware** - not because the assistance
path is dead (it answers queries fine), but because the QMI-LOC xtra
*finalization* is a stub. The remaining paths are all heavy: (1) a modem
firmware that implements QMI-LOC xtra finalization (unknown which, if any);
(2) reverse-engineer the proprietary izat/loc-HAL sequence Android uses and
reimplement it (large, uncertain - this is roughly what the FP6 pmOS GPS
bring-up required); (3) capture the exact `detail=2` meaning from Qualcomm QMI
headers to confirm it is "unsupported/disabled" rather than a missing step.

The **one lever that works** is coarse assistance: `--loc-inject-time` succeeds
and `--loc-inject-position-*` is available. That only turns a cold start into a
warm one (~30 s) *once almanac/ephemeris is cached*, so it cannot produce a
first indoor fix, but a small helper that injects time + a coarse (cell/IP)
position on GPS-enable is a real, shippable mitigation for repeat fixes. Not
built yet - deferred pending a decision on whether GPS is worth it given no
xtra.

The injector and data-source reader are kept at
`scripts/fp4-gps-xtra-inject.py` (diagnostic - reaches the finalization stub;
not a working fix). Operational caution: heavy qmicli LOC activity can make MM
briefly re-enumerate the modem (seen once, index 0 -> gone -> back at 0,
self-recovered, data reattached) - no reboot needed, but expect a short blip.
Modem restored to as-found after testing (assistance data deleted, MM location
back to `3gpp-lac-ci`).

**A caution for testing:** do not probe the engine with `qmicli --loc-*` while
ModemManager owns it. They share one LOC session; a `qmicli --loc-stop` tears
down MM's session too, and following NMEA through qmicli came back silent while
MM saw the same stream fine. Read location through `mmcli --location-get`, not
qmicli, unless MM's location is disabled first.

---

## D11 — the phone drops into EDL after repeated reboots {#d11}

**Status: OPEN as a cause, RECOVERABLE as a symptom.** Leading theory (A/B retry exhaustion) ruled out 2026-09-24. Seen three times on
2026-09-23. The first two needed a physical power-button hold. The third did
not, because it no longer has to -- see *Recovering without touching the
phone* below.

The phone stops booting and appears on the host as

```
Bus 003 Device 040: ID 05c6:900e Qualcomm, Inc. QUSB_BULK_SN:<serial>
```

which is the SoC's emergency download mode: powered, enumerating, but running
no OS. No fastboot, no adb, no network.

### It is not the reboot argument

The first occurrence followed `systemctl reboot --reboot-argument=bootloader`,
and this entry originally blamed that. **The second occurrence followed an
ordinary `systemctl reboot`**, so that explanation is wrong and is recorded
here only because the correction matters.

### Ruled out: an exhausted A/B retry counter

This was the leading theory. It is measured, on 2026-09-24, and it does not
hold. `qbootctl-mark-successful.service` is active and runs every boot:

```
qbootctl[556]: SLOT _b: already marked successful
qbootctl[556]: SLOT _b: Marked boot successful
systemd[1]: Finished Tell the bootloader this boot worked.
```

and the slot state is stable across boots:

```
SLOT _b:  Active: 1   Successful: 1   Bootable: 1
SLOT _a:  Active: 0   Successful: 1   Bootable: 0
```

So `qbootctl -m` **does** succeed despite the missing `slot_suffix` -- it takes
the slot from the partition table instead -- and every boot is marked
successful. A retry counter that is reset on every boot cannot exhaust. The
hypothesis is wrong.

There is a real but separate bug here: the boot cmdline carries no
`androidboot.slot_suffix`, so `qbootctl` logs `Couldn't find cmdline arg` and
`Unable to read boot slot property` on every run before falling back. It works,
but blind to what the bootloader actually chose. Worth adding the arg to the
boot image for correctness; it is not the EDL cause.

### Cause: still unidentified

With retry-exhaustion out, there is no confirmed cause. What the occurrences
have in common is a session of many reboots around flashing, not ordinary use;
each was plain EDL (a Sahara `HELLO`), never a ramdump, so the application
processor was not crashing into a debug image. Catching the trigger needs an
EDL event with early-boot instrumentation, which a random fault makes hard.
Left open, and cheap to live with now that recovery needs no physical access.

### Findings 2026-10-09 (sm6350-7.2.y kernel)

Three ramdumps in about 13 reboots in one session, all from a plain
`systemctl reboot`. What is now measured rather than assumed:

- **It IS a ramdump, not plain EDL.** Read on hardware: the device's Sahara
  `HELLO` reports `mode=2 (MEMORY_DEBUG)` and it serves a memory table (OCIMEM,
  AOP RAM, PMIC PON history, reset status, and all 6 GiB of DDR in three 2 GiB
  regions). So the earlier "plain EDL, never a ramdump" above is wrong for these
  episodes: something crashed, and RAM is preserved.
- **A watchdog bite produces exactly this.** Opening `/dev/watchdog0` and not
  feeding it gave `05c6:900e` at 30 s. The FP4's `qcom,kpss-wdt` maxes out at
  ~31 s (20-bit counter, 32 kHz clock), so systemd-shutdown's 10 min reboot
  watchdog request fails (`Invalid argument`) and runs at the 30 s default.
- **Linux cannot turn the ramdump mode off.** `qcom_scm.download_mode` reads
  `off`, writing `off` succeeds silently, and a bite still gives `900e`: the DT
  has no `qcom,dload-mode` register and the SCM call has no effect.
- **The reboot watchdog is not the (only) trigger.** With
  `RebootWatchdogSec=off` (verified: the previous shutdown never armed it) the
  second reboot still ended in `900e`. The change was reverted: with remote
  recovery available (below), a bitten hang is recoverable and an unbitten one
  is not.
- **Timing from the host's USB log:** a normal reboot reaches the next boot's
  USB gadget 31.6 s after the old one disappears; the bad one showed `900e`
  24.2 s after (a crash surfaces as `900e` within ~3 s), i.e. ~21 s after the
  gadget dropped -- late in shutdown or early in the next boot, where the
  journal cannot see.
- **The crashing boot's state cannot be read from the journal** (journald dies
  at "Sending SIGTERM to remaining processes"; good and bad shutdown logs end
  identically). Hence ramoops, now applied: kernel package pkgrel 11 reserves
  `ramoops@b0000000` (1 MiB) and enables `PSTORE_RAM/_CONSOLE/_PMSG`; after the
  next episode the crashed kernel's console tail is in `/sys/fs/pstore/`.

### Instrumentation to diagnose it (prepared 2026-09-26, applied 2026-10-09 as kernel pkgrel 11)

The reason the cause stays unidentified is that **nothing captures the moment**:
`/sys/fs/pstore/` is empty because the RAM backend is off. In the vendored kernel
config (`pkgbuilds/linux-moarchy-sm6350/config`) `CONFIG_PSTORE=y` but
`CONFIG_PSTORE_RAM`, `_CONSOLE` and `_PMSG` are all **not set**, and neither the
mainline `sm7225-fairphone-fp4.dts` nor `sm6350.dtsi` reserves a ramoops region.
Enabling ramoops would let the *next* EDL episode leave the previous boot's
console/panic tail in `/sys/fs/pstore/`.

Two ready ways to enable it (both need a kernel rebuild):

1. **Config + DT node (cleaner):** set `CONFIG_PSTORE_RAM=y`, `_CONSOLE=y`,
   `_PMSG=y` (this `select`s `REED_SOLOMON*` via olddefconfig -- re-vendor the
   resolved config to keep `prepare()`'s diff clean), and add a ramoops
   reserved-memory node to the fp4 dts. **The address matters: `0xb0000000` is
   NOT usable (see the hardware test below), despite showing as System RAM in
   `/proc/iomem`.**
2. **Config + cmdline (no DT, fits the no-kernel-patch stance):** the same config
   change, plus `memmap=0x100000$<addr> ramoops.mem_address=<addr>
   ramoops.mem_size=0x100000 ramoops.console_size=0x40000 ramoops.pmsg_size=0x40000
   ramoops.record_size=0x20000` on the cmdline (image/boot/android-bootimg.sh).

**Tested on hardware 2026-09-30 (INCONCLUSIVE -- boot did not come up, cause not
yet isolated).** Built the shipped commit `16337c9dd` with
`PSTORE_RAM/_CONSOLE/_PMSG=y` (`kernelrelease` matches shipped, so on-disk modules
load) plus a `ramoops@b0000000` DT node (1 MiB, no `no-map`), assembled a
transient `boot.img` (`~/Personal/linux-ramoops-d11/`), and `fastboot boot`ed it
twice. Neither came up: the first boot hung, the retry dropped into EDL ramdump
(`05c6:900e`, recovered with `scripts/edl-reset.py`).

**Do not over-read this.** Two *different* failure modes from one image (hang,
then ramdump) is the signature of the known flaky transient boot (D26 spends a
slot retry), not of a deterministic memory fault. And the address is almost
certainly fine: the FP4 **downstream** DT (`lagoon.dtsi` `reserved_memory`) marks
every protected carveout as an explicit `no-map removed-dma-pool` (hyp, xbl,
smem, all the pil_* firmware regions, `removed_region@c0000000`, the display
regions up to `dfps_data@a2300000+0x100000`), and **nothing reserves
`0xb0000000`** -- it is plain HLOS System RAM in both the downstream map and
mainline `/proc/iomem`. So the 2026-09-26 "verified free" note still looks right;
the ramoops node did not obviously crash anything.

**Next (isolation, not a new address):** re-boot the SAME kernel with the ramoops
node REMOVED. If it comes up cleanly, the two failures were flaky transient boots
and `0xb0000000` is usable (just retry the ramoops boot until it takes). If it
also fails, the fault is the `PSTORE_RAM` build or the transient method, not the
address. The build and the transient-boot workflow are otherwise proven.

**Why it is documented and not applied:** the config change requires a kernel
rebuild this environment can't verify (a bad config fails the build), the DT node
hits the same kernel-delivery wall as D23 (the kernel package is upstream-tag +
config-only, no patch step), and the cmdline variant shares D10's unverified
ABL-passthrough risk. So this is teed up for a build + verify, gated on the
kernel-delivery decision below.

**Honest limit of ramoops here:** it captures the *kernel* side -- a panic/oops
or the previous boot's console tail. It will catch an EDL that follows a kernel
crash or dirty shutdown. But "plain EDL, never a ramdump" points at the
bootloader (XBL/ABL) choosing EDL *before* the kernel, which ramoops cannot see;
a full diagnosis may also need the **PMIC PON/POFF warm-boot-reason register**
surfaced (a separate, complementary lead). Ship ramoops as the cheap first
instrument and read `/sys/fs/pstore/` after the next episode.

### Recovering without touching the phone

### Recovering without touching the phone

The documented recovery was a power-button hold, which is no use when the
handset is not in the room -- and this happened at 01:00 with nobody near it.
It is not necessary.

EDL speaks Qualcomm's Sahara protocol, and Sahara has a reset command that the
device honours *before* any authentication, with no signed programmer
uploaded and nothing flashed. Two 32-bit words out, an acknowledgement back,
and the phone reboots normally:

```
$ scripts/edl-reset.py
edl-reset: reset acknowledged, the phone is rebooting
```

The device replies `08000000 08000000` -- `SAHARA_RESET_RESPONSE`, length 8 --
and comes back on the network about two minutes later. Verified on hardware
2026-09-23 on a handset that had dropped into EDL after an ordinary reboot.

Worth being precise about what this does and does not do: it recovers the
*symptom*. Why the phone enters EDL at all is still unknown, and the A/B retry
hypothesis above is still the thing to test. But an EDL episode is no longer a
dead end that needs somebody in the room, which changes how safe it is to
reboot this phone unattended.

### Recovery

1. Hold **Power for ~15-20 seconds** until the phone goes dark.
2. **Unplug the cable**, hold **Volume Down**, and plug it back in — this is
   the order that works; holding Volume Down with the cable already attached
   does not.
3. In fastboot: `fastboot --set-active=b`, then `fastboot reboot`.

Nothing is written in EDL unless something deliberately flashes it, and both
occurrences recovered with the device intact.

### What to do about it

Until this is understood, **minimise reboots**, and check `qbootctl` after
each one. If the mark-successful path is indeed broken, the fix is either to
put `androidboot.slot_suffix` in the cmdline `android-bootimg.sh` builds, or
to make the unit pass the slot explicitly.

---

## D19 — the fingerprint reader is an Egis part with no Linux path {#d19}

**Status: UNSUPPORTED**, and investigated 2026-09-24 far enough to say what it
would take. Short answer: a driver and a matching stack, both written from
scratch. This is not a configuration problem.

**The part is Egis** (Egis Technology / Egistec). The wiki leaves this row
blank, so it was read off the handset: a scan of the `super` partition returns
73 occurrences of `egis` and 30 of `EGIS` in the vendor libraries. The exact
model did not surface within the time budget; Egis' side-mounted capacitive
parts are the ET5xx/ET7xx families.

**What the hardware looks like**, from this handset's own Android device tree
overlay in `dtbo_a`:

```
fingerprint_gpio {
        compatible    = "qcom,fingerprint-gpio";
        interrupt-parent = <&tlmm>;
        interrupts    = <17 0>;
        fp-gpio-int   = <&tlmm>, <17 0>;
        fp-gpio-reset = <&tlmm>, <18 0>;
        fp-gpio-power = <&tlmm>, <84 0>;
};
```

That node does nothing but reserve three pins. There is no SPI controller and
no SPI device anywhere in the overlay, so the bus binding lives in the vendor
kernel module rather than in the device tree — which is why the earlier note
here, that the SPI controller is not enabled, was looking for something that
was never described in the first place.

The handset also carries two partitions for this part alone: `fpconfig`
(128 KiB) and `fpconfig_persist`, the latter containing the string
`fpconfig2620`. Calibration data, of a shape nothing open knows how to read.

**Why this is harder than the other entries.** The kernel has no fingerprint
subsystem at all; everything goes through libfprint in userspace. libfprint
does have Egis drivers, but they are for the USB laptop parts (`1c7a:0570`,
`0575`, `0576`, `057e`), reverse-engineered one at a time. Its handful of SPI
drivers assume ACPI platforms where firmware handles regulators and interrupts,
which a device-tree phone does not. So closing this needs, in order: the model
identified, a kernel-side driver that powers and clocks the part and hands
frames to userspace, and a matching implementation for a sensor whose
enrolment and matching formats are undocumented.

**Confirmed against Fairphone's published kernel source, 2026-09-24.**
`kernel/msm-4.19` at `int/15/fp4` (from `gerrit-public.fairphone.software`)
carries exactly one fingerprint driver:
`drivers/misc/fpr_FingerprintCard/fpc1020_platform_tee.c` — a **Fingerprint
Cards FPC1020** TEE-platform driver. Read in full, the whole driver does only:
`vreg_setup` (power rails), `select_pin_ctl` (pinctrl), `hw_reset` (reset GPIO),
`device_prepare` (power sequence), `fpc1020_irq_handler` (a "finger touched"
interrupt that wakes userspace), and probe/remove plumbing. There is **no SPI
transfer, no image read, no `read()`/`write()` data path, no template storage
and no matching anywhere in the file.** The `_tee` suffix is literal: the
sensor's SPI bus and every byte of biometric data are owned by a QSEE/TrustZone
trustlet in the secure world. The kernel driver is a light switch and a
doorbell.

This settles the mechanism regardless of the FPC-vs-Egis question. The board
device tree (`kernel/msm-extra/devicetree`, `int/15/fp4`,
`lagoon-fp4.dtsi`) wires the same chip-agnostic `qcom,fingerprint-gpio` node
seen in the handset overlay above (int gpio17, reset gpio18, power gpio84) — it
names no vendor, so the kernel module binds by probe. The published kernel
module is FPC; this handset's `super` partition ships Egis userspace and Egis
calibration (`fpconfig2620`). The FP4 is known to have shipped more than one
fingerprint supplier; whichever this unit is, both use the identical Android
model — a dumb capture sensor plus a proprietary matcher in TrustZone — so the
conclusion does not move.

**Why the sensor being "already programmed" does not help us** (the question
that keeps coming up): the FPC1020 is a capacitive *image* sensor, not a
self-contained matcher. It has no onboard "is this the right finger?" logic to
query — it captures an image and streams it out, and the deciding is done by
the trustlet on the application processor's secure world, which is a signed
blob we cannot load or run on mainline. So there is nothing running on the chip
to simply talk to; the part that authenticates isn't on the chip at all. To get
fingerprint on this Linux stack you would have to *replace that brain* — either
run Qualcomm's secure OS + trustlet (not feasible on mainline), or build a
normal-world path: a from-scratch SPI capture driver (the register protocol is
undocumented, held by the closed TEE driver) plus an open matcher for a
proprietary template format. That is a reverse-engineering research project,
not a port, and it would also discard the security model (any root process
could then read the sensor and templates). Physically not impossible; but a
large research effort for a convenience feature, which is why the on-screen PIN
lock is the right answer here, not a stopgap for something readily fixable.

Nothing here blocks anything else, and no Linux phone this generation has a
working fingerprint reader for the same reasons. Recorded because it was asked
about, and because "no driver exists" is a different answer from "misconfigured".

## D21 — Bluetooth carries music but not call audio {#d21}

**Status: OPEN, re-tiered to a large (Tier-4-sized) task 2026-09-26.** Deeper
research found the gap is more fundamental than "unbridged in WirePlumber": it is
**missing at the kernel/AFE level**. On this phone call audio is DSP-internal --
`q6voiced` holds `VoiceMMode1` open and the DSP routes the voice session straight
to the same hardware backends as media (`QUIN_MI2S_RX` out, `TX_CODEC_DMA_TX_3`
in); the voice audio **never crosses an AP-accessible PCM**. And there is **no
Bluetooth audio backend at all**: the kernel has `# CONFIG_SND_SOC_BT_SCO is not
set`, no BT/AUX-PCM/SCO DAI in the sound card, and no HFP-AG↔modem bridge. So the
DSP has no BT AFE port to route a call to, and userspace has no PCM to loop back.
Two possible architectures, both sizeable: **(A)** add a BT-SCO/AUX-PCM AFE
backend (kernel config + DT DAI + DSP SCO support) + a UCM "Voice Call BT" verb;
or **(B)** a software loopback -- a q6routing voice↔MultiMedia path exposing the
call on host PCMs, bridged to PipeWire's native HFP-AG SCO nodes -- whose
feasibility hinges on whether those voice↔MultiMedia mixers exist on this card.
Either way it is kernel/integration work plus a live paired-headset call to
develop, which only the owner can drive; no repo change fixes it now. Next
code-free steps when the phone is up + a headset paired:
`amixer -c0 controls | grep -iE 'voice|mmode'` (does path B exist?) and, during a
call with the headset connected, `wpctl status`/`pw-dump` for HFP SCO nodes.

**Status (historical): OPEN**, and narrowed on 2026-09-24: not a missing package, an
unbridged path. Full confirmation needs a paired headset and a call.

A2DP (music) works. What the wiki records as "HFP/HSP don't work at all" is the
headset carrying a *cellular* call, and the pieces for it are further along than
"not installed":

- every PipeWire bluez codec is present, HFP included -- `libspa-codec-bluez5-hfp-cvsd`,
  `-hfp-msbc`, `-hfp-lc3-swb`;
- the adapter advertises the **Handsfree Audio Gateway** UUID as well as
  Handsfree, so bluez registers the AG profile a phone needs.

The gap is the bridge. For a headset to carry a cellular call the phone is the
HFP Audio Gateway, and the call's audio has to move between the modem's voice
path (the q6voice PCMs from [D17](./fp4-fixes.md#d17)) and the Bluetooth SCO
link. Nothing wires those together: there is no oFono (one classic AG backend),
and WirePlumber is not routing the modem call PCM to a BT SCO node. That
bridging is the work, and it is integration rather than a config toggle.

Not pursued further yet: it cannot be confirmed without a paired headset and a
live call, and the fix is sizeable. Recorded so the starting point -- codecs and
AG profile present, bridge absent -- is not re-discovered from scratch.

## D22 — the touchscreen controller logs recurring i2c failures {#d22}

**Status: OPEN**, and cosmetic so far.

The touchscreen works. The HX83112A controller nevertheless fails i2c
transactions periodically — 19 occurrences in one boot of roughly two and a
half hours, in bursts of three:

```
gpi 900000.dma-controller: Error in Transaction
geni_i2c 988000.i2c: DMA txn failed:3
geni_i2c 988000.i2c: GPI transfer failed: -5
[HXTP][ERROR] himax_bus_write: i2c_write_block retry over 3
[HXTP][ERROR] himax_mcu_read_event_stack: i2c access fail!
[HXTP][ERROR] himax_touch_get: can't read data from chip!
```

The driver retries three times and gives up on that event, so the visible
symptom would be an occasional dropped touch rather than a dead screen, which
matches the handset being usable. The failure is at the GPI DMA layer beneath
i2c rather than in the touch driver, which points at the i2c controller's DMA
path and not at the HX83112A.

Worth watching rather than chasing: it has not yet been tied to anything a user
would notice.

**Likely fix (kernel-delivery-blocked, low priority), scoped 2026-09-26.** The
failures are at the GPI DMA layer under the `988000.i2c` (GENI) controller, not
in the HX83112A. The standard workaround for flaky GENI/GPI-DMA i2c on Qualcomm
is to take that bus **off DMA into FIFO/PIO mode** -- for a low-bandwidth
touchscreen the CPU cost is negligible and it removes the GPI DMA failure path
entirely. That is a DT change on the `&i2c` node for the touch controller
(drop/adjust its `dmas`, or a `qcom,...` FIFO quirk), so it hits the same
kernel-delivery wall as D11/D23 (the kernel package is upstream-tag, config-only,
no patch step). Cosmetic and not user-visible beyond a rare dropped touch, so it
stays low priority behind the kernel-delivery decision.

**Tested 2026-09-30: the drop-dmas approach is disproven; not DT-fixable.**
Deleting `dmas`/`dma-names` on `&i2c8` does NOT fall back to FIFO here. The geni
i2c driver uses GPI DMA precisely when the SE has FIFO disabled in firmware
(`fifo_disable`, read from the read-only `GENI_IF_DISABLE_RO` register); for this
SE that bit is set, so `setup_gpi_dma()` is mandatory. With the dmas removed the
bus fails probe (`geni_i2c 988000.i2c: error -ENODEV: Failed to get tx DMA ch`),
which takes the touchscreen with it (`hx83112a_probe` backtrace, no input
device). The FIFO path the driver has is gated on that hardware bit, which no DT
change flips. So the GPI-DMA flakiness needs a lower-level fix (GPI DMA driver or
SE firmware), not a device-tree switch. Leave D22 as-is; the driver's retries
absorb it.

---

## D23 — the camera's CSI PHY supplies are undescribed, and a clock sticks on {#d23}

**Status: CSI PHY supplies FIXED 2026-09-30 (sm6350-mainline/linux PR #15),
verified on hardware; the stuck AXI clock remains (an upstream driver issue,
below). The regulator data was UPDATED 2026-09-24 from Fairphone's own board
device tree.** Two separate items, neither with a user-visible symptom.

**Verified 2026-09-30.** The supply-name fix (below) booted on the handset
(7.2.0-dtstest): camss probes, /dev/video0..12 appear, and dmesg shows **zero**
"using dummy regulator" lines for csiphy (was 8). Sent upstream as PR #15;
permanence follows the same upstream-then-pin path as the NFC/mic work. The
stuck AXI clock is unaffected by this and stays open.

**The CSI PHY supplies.** Every CSI PHY rail falls back to a dummy regulator at
probe:

```
qcom-camss acb3000.isp: supply vdd-csiphy0-0p9 not found, using dummy regulator
   ... csiphy0-1p25, csiphy1-*, csiphy2-*, csiphy3-*
```

The `qcom,sm6350-camss` binding defines `vdd-csiphy{0-3}-{0p9,1p25}-supply` and
the FP4 `&camss` node sets none. The earlier note here said the rail identity
"needs data not on hand" — it did, and now it is on hand. The FP4 board device
tree (`kernel/msm-extra/devicetree`, branch `int/15/fp4`,
`qcom/camera/lagoon-camera.dtsi`, cloned from `gerrit-public.fairphone.software`)
names them on each `qcom,csiphy@N` node, under the vendor's own property names
rather than the mainline ones:

```
regulator-names   = "gdscr", "refgen", "mipi-csi-vdd1", "mipi-csi-vdd2";
mipi-csi-vdd1-supply = <&L18A>;   rgltr-min/max 880000 / 1049000   → the 0.9V rail
mipi-csi-vdd2-supply = <&L22A>;   rgltr-min/max 1200000 / 1305000  → the 1.25V rail
```

So, authoritatively:

- **`vdd-csiphy*-0p9` = PMIC LDO L18A** (0.9 V nominal, 0.88–1.049 V range);
- **`vdd-csiphy*-1p25` = PMIC LDO L22A** (1.25 V, 1.2–1.305 V range);
- all four PHYs share the same two rails (both are on the main PMIC, the "A"
  = pm6350 in mainline terms), plus the `cam_cc_titan_top_gdsc` GDSC and the
  SoC `refgen`, which mainline already handles.

**The exact patch, confirmed against the mainline kernel 2026-09-26.** The
mainline tree is `v7.2.0-sm6350` (`sm6350-mainline/linux`). Its camss driver
(`csiphy_res_sm6350[]`) requests eight per-PHY supplies
(`vdd-csiphy{0..3}-{0p9,1p25}`, each `init_load_uA = 80000`), but the FP4
`&camss` override in `arch/arm64/boot/dts/qcom/sm7225-fairphone-fp4.dts` sets
only `vdda-0.9-supply` / `vdda-1.25-supply` -- **names the sm6350 camss code
never reads** (grepped the whole `camss/` dir). So those two lines are dead and
all eight supplies fall to dummy regulators. The mainline regulator labels are
confirmed in the same dts: `vreg_l18a` (0.788-1.049 V, the 0.9 V rail) and
`vreg_l22a` (1.08-1.305 V, the 1.25 V rail), both already powering other
consumers, so enabling them from camss is harmless. The patch:

```diff
 &camss {
-	vdda-0.9-supply = <&vreg_l18a>;
-	vdda-1.25-supply = <&vreg_l22a>;
+	vdd-csiphy0-0p9-supply = <&vreg_l18a>;
+	vdd-csiphy0-1p25-supply = <&vreg_l22a>;
+	vdd-csiphy1-0p9-supply = <&vreg_l18a>;
+	vdd-csiphy1-1p25-supply = <&vreg_l22a>;
+	vdd-csiphy2-0p9-supply = <&vreg_l18a>;
+	vdd-csiphy2-1p25-supply = <&vreg_l22a>;
+	vdd-csiphy3-0p9-supply = <&vreg_l18a>;
+	vdd-csiphy3-1p25-supply = <&vreg_l22a>;
 	status = "okay";
```

**Delivery is the blocker, not the diff** (see the note in D11 and below): this is
a kernel-tree change, and `pkgbuilds/linux-moarchy-sm6350` builds the pinned
upstream tag **config-only, with no patch step** by design. So this patch needs a
decided delivery path -- a patch step in the kernel PKGBUILD, or a moarchy kernel
branch the manifest's `kernel-ref` points at -- the same path the existing
kernel DT work (aw88264/mic/NFC) needs. Correctness/power polish; the camera
works today on the dummies, so it is not urgent. (What the vendor DTS does
**not** give, and what would actually improve image *quality*, is the Qualcomm
ISP tuning -- chromatix/CAMX black-level, lens shading, colour matrices -- which
lives in the gated `camera-devicetree` + proprietary blobs.)

**Autofocus (the main sensor's real limiter), scoped, not attempted:** the main
imx582 has a VCM actuator but no focus control because mainline `qcom-camss`
does not drive actuators and no V4L2 lens subdev binds. Unlocking it needs (a)
identifying the VCM chip (the vendor `qcom,actuator` node names none -- an
I2C/CCI probe or the schematic; likely dw9714/dw9807/ak7375), (b) a mainline
V4L2 VCM driver bound over CCI in the dts, and (c) libcamera `CameraLens` +
megapixels AF support. A multi-component upstream/libcamera effort, not a patch.

**The stuck AXI clock.** Tearing a capture stream down warns every time:

```
gcc_camera_axi_clk status stuck at 'on'
  camss_disable_clocks / vfe_put / vfe_set_power / video_unprepare_streaming
```

The AXI clock does not report itself off when camss releases it. A clock left
running costs power and can block suspend later. This is a driver/clock issue,
independent of the supplies, and belongs upstream with the SoC's CAMSS support;
it is not fixed by anything in this tree.

---

## D28 — the phone never suspends on idle; it only blanks the screen {#d28}

**Status: REOPENED 2026-10-01 as D29 (s2idle does not hold). The wiring below is
correct; the deeper problem is that s2idle never stays asleep.** See [D29](#d29).

**Status (2026-09-27): FIXED, s2idle drain measured (~1 %/h, ~7x better than
screen-blank); pending only the pkgrel-13 reflash to put the same config on the
phone.** The idle->suspend wiring
(steps 1-2) was done 2026-09-26. Two follow-up overnight watches saw zero
suspends, but the cause was NOT the idle logic -- it was that the swayidle under
test had been relaunched over ssh and so had no logind seat, and polkit denies
suspend to a seatless session (see "Root cause" below). The in-session swayidle
is authorized and suspends. The idle action has also been consolidated into one
step (moarchy-idle-action). This is Track A1 in
[`fp4-roadmap.md`](./fp4-roadmap.md) and the reason battery is rated `P`: "screen
off" is not "asleep", so the phone drains while it looks off.

**What is already there** (checked on the handset, read-only):

- s2idle is the supported mode: `/sys/power/state` = `freeze mem`,
  `/sys/power/mem_sleep` = `[s2idle]` (there is no deep/S3 on this SoC in
  mainline, which is normal).
- Wake sources are enabled where it matters: the power-button PMIC pwrkey
  (`pon@800:pwrkey`), all three remoteprocs including `remoteproc1: modem`
  (so an incoming call/SMS can wake the AP), and the RTC alarm (`rtc@6100`).
- The fuel gauge works: `qcom_qg` reports capacity and voltage
  (99 %, 4.39 V). `current_now` reads 0 while on USB, so idle drain cannot be
  measured until the phone is unplugged.
- Nothing is holding a wakelock: no `/sys/kernel/debug/wakeup_sources` entry has
  `prevent_suspend_time > 0`, so s2idle should be enterable.
- `rtcwake` is installed (util-linux 2.42.3) with a working
  `/sys/class/rtc/rtc0/wakealarm`.

**The actual defect.** Nothing triggers a system suspend:

- `logind` has `IdleAction=ignore` (the default), so it never suspends.
- swayidle runs a single rule, `timeout 600 moarchy-idle-blank`, which blanks
  the panel and stops there. The CPU, modem and buses stay fully powered with
  the screen dark.

So after ten minutes the display goes off and the phone keeps running at full
idle power. That is the battery finding.

**What remains, in order:**

1. **Prove s2idle resumes cleanly. DONE 2026-09-26 — it works.** `rtcwake -m mem
   -s 25` (armed detached via `systemd-run` so it did not depend on the ssh
   session), with the owner watching. The phone suspended, woke on the RTC
   alarm, and reconnected on its own; `/sys/power/suspend_stats/success` went
   0 -> 1, fail 0, and dmesg shows a clean cycle:

   ```
   PM: suspend entry (s2idle)
   Filesystems sync: 0.027 seconds
   Restarting tasks: Starting / Done
   PM: suspend exit
   ```

   No failed devices, entry->resume in ~2 s. So s2idle is safe to enable.
2. **Wire idle -> suspend. DONE 2026-09-26, tested live on the FP4.** A second
   swayidle timeout (config/hypr/autostart.lua) runs `bin/moarchy-idle-suspend`
   60 s after the blank, with `before-sleep 'moarchy-lock'` so every suspend path
   wakes to the PIN pad. moarchy-idle-suspend skips Stay Awake and an active
   call/audio (sink/source streams, sleep inhibitors). Shipped by moarchy pkgrel 9.
   Verified on hardware: `moarchy-idle-suspend` suspended to s2idle
   (`suspend_success` incremented, clean resume), and a **single** power-button
   press woke it to the PIN pad, which the PIN unlocked.

   **Also fixed a double-press bug found during this test.** Idle-off used to run
   `moarchy-screen blank`, which blanks WITHOUT setting the lock flag (so touch
   could wake it). But the power button decides lock-vs-wake off that flag, so the
   first press after idle read no flag and re-locked (a no-op blank) instead of
   waking -- you needed two presses, unlike a power-button-off (one). Fix:
   `moarchy-idle-blank` now runs `moarchy-screen lock`, so idle-off leaves the
   same state as power-off and a single press wakes. Trade-off: touch no longer
   wakes from idle (the power button does), which is standard phone behaviour and
   is what removes the two-handler race. Verified: single press now wakes from
   idle.
3. **Measure idle drain** in s2idle vs screen-blank-only, which needs the phone
   unplugged (USB masks `current_now`), same constraint as GPS testing.

**Screen-blank-only drain measured 2026-09-26, unplugged overnight.** With the
system never suspending (confirmed: `/sys/power/suspend_stats/success` = 0, no
suspend lines in dmesg over ~46 h uptime), the battery went **99% -> 34%
overnight** (4.39 V -> 3.72 V). Instantaneous draw ~**0.73 W** (upower), ~8 h to
empty at 34% -- i.e. roughly **~1 day of standby from full**, screen off, doing
nothing. That is the cost of D28: a phone that should get days of s2idle standby
gets about one. The s2idle comparison number still needs step 1 (a proven
resume) before it can be taken. (Aside: the fuel-gauge/charger status
misreports here -- `qcom_qg` status reads Unknown and pm7250b-charger reads
"Charging" while clearly discharging; cosmetic, but it breaks any status-based
UI.)


**Root cause of "zero overnight suspends" (corrected 2026-09-27): a polkit/seat
problem in the test setup, not the idle logic.** Two overnight watches showed
`/sys/power/suspend_stats/success` stuck at its starting value across the whole
night. But the swayidle being watched had, in both cases, been relaunched over
ssh during earlier live debugging -- and an ssh-launched process has no logind
seat. logind/polkit grants `org.freedesktop.login1.suspend` to an **active,
seated** session with no prompt, but requires admin authentication (`auth_admin`)
for a seatless one, which on this locked-password phone can never be satisfied.
So swayidle fired its timeout, ran the action, called `systemctl suspend`, and
was silently denied. The swayidle stderr confirms it:

```
Call to Suspend failed: Access denied as the requested operation requires
interactive authentication. However, interactive authentication has not been
enabled by the calling program.
```

Proof it is the seat, not the code (checked 2026-09-27 with `pkcheck`):

| caller | logind session | `pkcheck ... login1.suspend` |
|---|---|---|
| Hyprland (graphical) | c1, seat0, active | **rc=0 (authorized)** |
| ssh shell | c197/c208, no seat | rc=2, `auth_admin_keep` (denied) |

So the earlier "timer reset" hypothesis (that a 600 s blank reset swayidle's
clock so a 660 s suspend timeout never fired) was wrong: swayidle does fire, and
the graphical session is authorized. **The lesson: idle-suspend cannot be tested
by relaunching swayidle over ssh.** A hand test must put swayidle in the
graphical session -- e.g. `hyprctl dispatch 'hl.exec_cmd("swayidle ...")'`, which
Hyprland runs as its own child on seat0 (`cgroup: session-c1`, `pkcheck rc=0`).

**Code change kept regardless:** the idle action is consolidated into one
`bin/moarchy-idle-action` (lock, then `moarchy-idle-suspend`) driven by a single
swayidle timeout, replacing the two-timeout design -- one atomic action, no
dependence on a second timeout. `config/hypr/autostart.lua`, moarchy pkgrel 13.
This is a simplification, not a fix for a proven bug in note 9.

**Valid test DONE, s2idle drain measured (2026-09-27, on battery, unplugged).**
swayidle launched in-session via `hl.exec_cmd` (`cgroup: session-c1`,
`pkcheck rc=0`), running `timeout 600 moarchy-idle-action ...`; guards all clear.
It suspended once at ~04:16 (last awake 04:13 @ 62%) and held s2idle until the
power-button wake at ~09:22 @ 57% -- `suspend_stats/success` 3 -> 4, fail 0.

**Result: 62% -> 57% = 5% over ~5.1 h = ~1 %/h in s2idle**, vs the
screen-blank-only ~7 %/h (~0.73 W) measured earlier -- about **7x** slower, i.e.
roughly **4 days** of standby from full instead of ~1. It also re-suspended on
its own after the morning wake (idle-suspend fires repeatedly, not just once).
(Caveat: the fuel gauge misreads for a few minutes after resume -- a 57%->49%
jump in 5 min awake is the qcom_qg `status=Unknown` re-settle, not real drain; the
across-suspend delta is the reliable figure.) Step 3 is complete; D28 is fixed,
pending only the same config reaching the phone via the pkgrel-13 reflash.

**Note for the reflash:** the image built overnight (2026-09-27) carries the old
two-stage config -- the change landed after that build started. Rebuild the image
(moarchy is now pkgrel 13) before/with tomorrow's reflash. After reflash, the
valid confirmation is simply: boot, leave the phone idle and unplugged, and watch
`suspend_success` climb -- do NOT relaunch swayidle over ssh (it would be denied).

## D29 -- s2idle enters but does not stay asleep {#d29}

**Status: OPEN (found 2026-10-01, on the flashed 0.5.0 public build).** The
suspend path works but the SoC never stays in s2idle: it enters and exits in
~0.5-1.5 s every time. This is why battery is `P` and why D28's overnight figures
do not reproduce. It is a local/runtime problem, NOT an upstream gap.

**Measured** (over ssh):

| trigger | conditions | s2idle held |
| --- | --- | --- |
| `echo mem` | on USB | ~2 s |
| `echo mem` | unplugged | <1 s |
| `systemctl suspend` (root) | WiFi off | 0.47 s |
| `echo mem` | screen blanked + WiFi off | 1.45 s |

`suspend_stats/success` increments each time, but `pm_wakeup_irq` is empty and no
registered wakeup source increments -- not a clean wakeup-source wake. During the
frozen window the churn is IPI + `arch_timer` + `apps_rsc` (the RPMh RSC) + geni
`gpi-dma`/UART.

**Ruled out** (with evidence): USB charger/typec, WiFi (D5 power-save-off is a red
herring here), the display, phantom touch input (D22 -- input quiet, touch IRQ
flat), the stay-awake flag, blocking sleep inhibitors, active audio, and broken
idle-notify (a probe swayidle fires at its timeout, so Hyprland delivers idle).

**Upstream supports deep sleep on this SoC.** SM6350/SM7225 (pre-Hamoa Qualcomm)
reaches s2ram-equivalent depth *through* s2idle + RPMh; a Jan-2026 lkml RFC
("soc: qcom: rpmh-rsc: Register s2idle_ops...") assumes these targets hit s2ram
depth in s2idle (it only fixes NVMe context loss; the FP4 has UFS, so it does not
apply). `sm6350.dtsi` is fully wired: `cluster_pd` references `cluster_aoss_sleep`
(`arm,psci-suspend-param = <0x4100b244>`), the deepest AOSS cluster idle state.

**Peripheral-hold hypothesis RULED OUT by elimination (2026-10-01).** `cx`/`mx`
sit "on" at perf 256, but stopping the likely holders one by one and cumulatively
did **not** change the ~1 s exit: USB autosuspend 0.9 s, +BT off 0.9 s, +cdsp+adsp
stopped 0.9 s, +modem stopped 1.0 s (baseline 1.4 s). So no peripheral subsystem
(USB, Bluetooth, the adsp/cdsp/modem remoteprocs) is what pulls it out of s2idle.

**So it is kernel-internal:** the s2idle loop exits in ~1 s with no registered
wakeup source and empty `pm_wakeup_irq`, independent of every peripheral.

**CONFIRMED via `qcom_stats` on the PRODUCTION kernel (2026-10-01) - no debug
kernel needed.** `CONFIG_QCOM_STATS=m` ships in the stock config; `sudo modprobe
qcom_stats` exposes `/sys/kernel/debug/qcom_stats/`. The deep-sleep counters are
all zero since boot (uptime 56 min): `aosd` (AOSS) Count 0, `cxsd` (CX collapse)
Count 0, `ddr` (self-refresh) Count 0. So the SoC has **never once reached
system-wide deep sleep** - that is the whole defect, measured directly.
`power-domain-cpu-cluster0` is "on" (never power-collapses).

**Holder of `cx` NOT yet pinned (two earlier conclusions retracted - see below).**
Verified by controlled tests (2026-10-01, prod kernel): with the phone **unplugged
AND the display genuinely idle** (`dpms` off, `disp_cc_mdss_mdp_clk` enable_count
= 0, zero atomic commits in a 6 s passive ftrace), **`cx`/`mx` are still pinned at
perf 256 and a real `echo mem` still never deep-sleeps** (`aosd`/`cxsd`/`ddr`
stay 0). So the deep-sleep blocker is a continuous `cx` NOMINAL vote from a
subsystem that none of the black-box toggles released.

**Ruled out as the holder (by controlled test):**
- **Display / MDP clock.** ftrace shows the kernel DOES release the mdss clocks
  when the compositor is idle (`clk_bulk_disable` via `mdss_runtime_resume`'s
  counterpart; enable_count 0 at idle). The clock is only on while Hyprland is
  committing frames. `cx` stays 256 even with it fully off. (This retracts the
  earlier "MDP clock leak on dpms-off" conclusion, which was taken while the
  screen had bounced back ON - a `dpms("off")` request itself triggers a commit
  that re-enables the panel, which is a *separate* compositor bug: dpms-off does
  not stick.)
- **USB cable** (unplugged, `cx` still 256) and the **dwc3 controller** (unbound
  `a600000.usb`, USB master clock and `cx` unchanged).
- **modem / adsp / cdsp / Bluetooth** (stopped individually + cumulatively
  earlier; no change - though that run was screen-on-confounded).
- Display-clock-leak upstream fixes do not apply anyway: the 2020 DPU
  system-suspend fix is already in our kernel; the 2026-07 dual-MDSS-PD series is
  SM8750-only; no cont_splash/simplefb here.

**Process note / two retractions.** D29's cause was mis-called twice from
*confounded single readings*: first "`0xb0000000` crashes the SoC" (really flaky
transient boots), then "MDP clock leak holds cx" (readings taken with the screen
bounced back on). Lesson: control BOTH confounds (unplugged + display-idle) and
trace the actual `cx` perf-state vote before concluding.

**cx consumer NAMED (2026-10-01, prod kernel, `pm_genpd_summary`).** Reading
`/sys/kernel/debug/pm_genpd/pm_genpd_summary` with the phone unplugged + idle, the
`cx` domain lists exactly one consumer voting the NOMINAL corner:

```
cx                              on                              256
    884000.serial                   active                      256   SW
    ae01000.display-controller      active                      64    SW
    8804000.mmc                     suspended                   0     SW
    genpd:0:4080000.remoteproc      suspended                   0     SW
```

`884000.serial` is `ttyMSM0` (the geni QUP UART). It is **runtime-active for
essentially all of uptime** (`power/runtime_active_time` 22,563 s vs
`runtime_suspended_time` 15 s over a 6 h boot) and votes `cx` perf 256. The
display-controller only votes 64, so even a fully idle display leaves `cx` pinned
at 256 by the serial port - which is exactly why every display/MDP test above left
`cx` at 256. **The serial port is the dominant `cx` holder.**

Two things keep `ttyMSM0` active: (1) it is a registered kernel console (`EC` in
`/proc/consoles`; no explicit `console=` in `/proc/cmdline`, so it comes from the
DTB `chosen`/default), and (2) moarchy runs `serial-getty@ttyMSM0.service`
(`agetty` on it since boot). **The instrumented test below shows only (1) matters -
the getty is NOT the holder.**

**Driver mechanism (source-confirmed, `qcom_geni_serial.c` @ v7.2.0-sm6350).** The
`cx` vote is an **OPP vote**: `geni_serial_set_rate`/`geni_serial_resources_on`
call `dev_pm_opp_set_rate(uport->dev, clk_rate)` (and `dev_pm_opp_set_level` by
baud), which requests the `cx` corner. It is dropped in exactly one place -
`geni_serial_resources_off()` calls `dev_pm_opp_set_rate(uport->dev, 0)` - and that
runs **only** from `qcom_geni_serial_runtime_suspend` (`SET_RUNTIME_PM_OPS`).
Runtime suspend only fires when the port is idle; the **console role** keeps the
port in use, so `runtime_suspend` never runs and the OPP/`cx` vote never falls
(matches the live `runtime_active_time` ≈ full uptime).

**Instrumented test 2026-10-02 (self-healing probe, `qcom_stats` + ftrace). This
REFUTES the earlier "getty is the holder / dropping
it hangs the box / deep-idle slowdown" write-ups - all three were wrong.** The
probe stopped `serial-getty@ttyMSM0` *and* `pkill`ed `agetty`, waited 14 s, then
ran `echo mem` with a +12 s rtc wakealarm, tracing throughout, then auto-restored.

- **Phase B (getty stopped + agetty killed, +14 s): `cx` still 256,
  `884000.serial` still `active`.** Its `runtime_active_time` rose by the full
  ~14,860 ms while `runtime_suspended_time` did **not** move (stayed 15,470 ms) -
  the port was runtime-active the entire 14 s. **So killing the getty does nothing
  to the pin; the registered-console role holds the port active on its own.**
- **Phase C (`echo mem`, rtc +12 s): returned in ~1.98 s, not 12 s** - the D29
  early-exit - and `aosd`/`cxsd`/`ddr` stayed **0**. s2idle still never reaches
  deep sleep, and it exits in ~2 s *independently of the cx pin*.
- **The earlier "userspace hang / deep-idle slowdown" was a measurement artifact:**
  the `qcom_stats` counters prove the SoC never deep-slept, so the ~90 s sluggishness
  was not deep idle. The likely cause is the **function tracer** on
  `dev_pm_opp_set_rate`/`_genpd_set_performance_state` - those fire on every
  clock/perf change system-wide, so tracing them crawls everything; the system
  recovered exactly when restore set `current_tracer=nop`. The phone was never
  wedged and never needed a reboot (it was not rebooted; it is up 1h56m).

**ROOT CAUSE CONFIRMED (2026-10-02, instrumented, two independent confirmations).**
The ~2 s exit is an **armed wakeup IRQ firing the instant s2idle idles the CPUs**.
Traced via the `suspend_resume` tracepoint: the abort happens between
`dpm_suspend_late` end and the `dpm_suspend_noirq`/resume turnaround (no
`machine_suspend`/idle-loop time at all - s2idle never enters). A `func_stack_trace`
on `pm_system_irq_wakeup` caught the single call, from
`irq_pm_handle_wakeup <- handle_fasteoi_irq <- gic_handle_irq <- cpuidle_enter`,
and a kprobe on its argument named the IRQ:

```
irq=172   ->   172:  msmgpio  64  Edge  884000.serial:wakeup
```

**IRQ 172 is the serial console's RX wakeup GPIO (`884000.serial` = `ttyMSM0`, the
geni debug UART, routed through `msmgpio` GPIO 64).** It is armed as a wakeup during
`suspend_device_irqs` and fires immediately, so `irq_pm_handle_wakeup` sets
`pm_abort_suspend`, `pm_wakeup_pending()` is true at the post-noirq check, and the
suspend bails. The handler never runs (the IRQ is masked by the wakeup path), which
is why `/proc/interrupts` shows count 0 for it and why `pm_wakeup_irq` reads empty
after resume (resume clears it) - those two red herrings sent the earlier passes
looking at wakeup *sources* instead of an armed wakeup *IRQ*.

**Confirmation:** `echo disabled >
/sys/devices/platform/soc@0/8c0000.geniqup/884000.serial/power/wakeup`, then
`echo mem` with a +20 s rtc alarm -> **suspend HELD for 21.08 s** (vs ~1.3 s with
it enabled), and the only wake was `irq=116 pm8xxx_rtc_alarm`, i.e. the intended
alarm. So disabling the serial console's wakeup fixes the abort outright.

**It is two stacked serial-console problems, both now correctly placed:**
1. **(primary, the abort) the RX wakeup IRQ 172** aborts every suspend in ~2 s.
   Fixed by disabling `884000.serial` wakeup.
2. **(secondary, depth) even once s2idle HELD for 21 s, `aosd`/`cxsd`/`ddr` stayed
   0** - the SoC held in s2idle but did not reach the RPMh-tracked deep collapse.
   This is where the earlier `cx`-pinned-at-NOM finding belongs: the console's OPP
   vote caps the depth. It is real but second-order; the abort had to be fixed
   first to even observe it.

Both point at the same thing: the serial console does not quiesce for suspend.
Earlier mis-calls on this defect (MDP clock, "getty is the holder", "USB/VBUS",
"deep-idle hang") are all retracted above - this is the first cause backed by a
suspend that actually held.

**Fix (implemented for the primary abort; secondary depth still open).** A udev rule
disables the console's wakeup at boot:
`moarchy-device-fp4/82-fp4-serial-wakeup.rules` ->
`ACTION=="add|change", SUBSYSTEM=="platform", KERNEL=="884000.serial",
DRIVER=="qcom_geni_serial", ATTR{power/wakeup}="disabled"` (installed to
`/usr/lib/udev/rules.d`, `moarchy-device-fp4` pkgrel 5). **Verified via the real
udev path 2026-10-02** (not just a manual echo): rule installed -> `udevadm
control --reload` + `trigger` flips `884000.serial/power/wakeup` `enabled`->`disabled`
-> `echo mem` **held 21 s** to its rtc alarm (was ~1.3 s). Ships in the next image;
no kernel change needed. This is the local stopgap - see the upstream fix below.

**Both D29 layers are one kernel bug, and the upstream fix is already in flight.**
The spurious wake (irq 172) and the depth cap (`cx`/OPP not dropping) have the same
cause: a geni UART kept runtime-active (a console) never runs its runtime-suspend
callback during *system* suspend, so `geni_se_resources_off()` is never called -
and that one function is what applies the **sleep pinctrl** (RX `bias-pull-up`,
which the FP4 DT `qup_uart1_sleep_rx` ships specifically "to avoid ... spurious
wakeups") **and** drops the **OPP/`cx` vote** (`dev_pm_opp_set_rate(dev, 0)`).
`qcom_geni_serial_suspend()` only re-tags the interconnect; it does not deactivate
resources. **Upstream patch (under review, NOT merged):** "serial: qcom-geni: add
force suspend/resume to system sleep callbacks", Praveen Talari (Qualcomm), 2026-07
- calls `pm_runtime_force_suspend()` in suspend / `pm_runtime_force_resume()` in
resume, which invokes the runtime callback (hence `resources_off`: sleep pinctrl +
OPP drop) during system sleep. It skips only when `no_console_suspend` is set; FP4
does not set that, so it should fix **both** layers for us. Its stated motivation
("resources ... not gated ... prevents the platform from reaching its lowest idle
state") is exactly this defect.

**VALIDATED on FP4 hardware 2026-10-02 (patch fixes the abort; depth is a SEPARATE
bug after all).** Talari's patch was cherry-picked onto the pinned kernel (branch
`d29-patch-test`, kernel `pkgrel 2`), built in CI, and transient-booted
(`fastboot boot`, boot-only, modules unchanged since `uname` is identical) with the
udev workaround removed and **serial wakeup ENABLED** - the clean test. Result:
`echo mem` **held 21.8 s** to its rtc alarm (was ~1.3 s), and a kprobe on
`pm_system_irq_wakeup` showed the only wake was `irq 116 pm8xxx_rtc_alarm` (intended)
- **irq 172 no longer aborts suspend.** So the patch fixes the abort on FP4 exactly
as well as the udev rule. **But `aosd`/`cxsd`/`ddr` stayed 0** even with the patch
dropping the serial's OPP/`cx` vote - so the deep-collapse depth cap is **NOT** the
serial after all (this retracts the "secondary = serial OPP vote" guess above): some
other `cx` consumer, or a separate RPMh/cpuidle condition, keeps the SoC out of the
deepest state. The abort - the actual "s2idle won't hold" defect and the battery win
- is what both fixes address; the depth cap is now a distinct open item.

**The patch is ALREADY MERGED TO MAINLINE (checked 2026-10-02): commit
`d0cd9c8d0fd5` "serial: qcom-geni: add force suspend/resume to system sleep
callbacks" is in torvalds/linux.** (Confirmed via the Aug-2026 follow-up "serial:
qcom-geni: Fix unbalanced pm_runtime_enable" by Abel Vesa, which carries
`Fixes: d0cd9c8d0fd5` and its own `Reviewed-by: Praveen` + `Tested-by: David
Heidelberg` (OnePlus 6T) - that follow-up only matters for `no_console_suspend`,
which FP4 does not set, but carry it too for correctness.) So this is **not** an
upstreaming task - no `Tested-by` to send, no new patch. The under-review v2 we
test-carried on `d29-patch-test` is byte-equivalent to the merged commit; use the
**mainline commit(s)** for the real adoption.

**Next (adopt, don't upstream):** (1) in moarchy, either bump the `sm6350-mainline`
kernel to a base that already contains `d0cd9c8d0fd5` (+ the unbalance fix), or
backport those two mainline commits into `linux-moarchy-sm6350` referencing their
SHAs - then drop the udev stopgap (`82-fp4-serial-wakeup.rules`); (2) flag it to
Luca Weiss (sm6350-mainline + the pmaports `linux-postmarketos-qcom-sm6350` aport)
so the fork/pmaports pull it, with our FP4 validation as the justification - a note/
issue, not an MR of an under-review patch. Separate open item: the deep-collapse
depth cap (what still holds `cx`/blocks `cxsd`), and the compositor not keeping the panel
dpms-off.

## D30 -- the quickshell before-sleep lock traps a locked-password account {#d30}

**Status: OPEN -- important (found 2026-10-05, on a full flash of the
`fp4-dev-all` dev image).** After a full flash the account ships with a *locked*
password (`passwd -S` reports `L`; tty1 autologin does not consult one, by
design -- see `docs/structure.md` I8 and D-note in `bin/moarchy-system-lock`).
The quickshell shell's own lockscreen ("omarchy lock", PAM config
`omarchy-lock-password`) is triggered on `before-sleep` by
`swayidle -w ... before-sleep moarchy-lock`. It authenticates through PAM, and a
locked password can never satisfy PAM, so the first time the phone sleeps it puts
up a PIN/password prompt that **nothing the user types can clear** -- a full
lockout. `loginctl unlock-sessions` does not dismiss it (the lock is a secure
ext-session-lock client that only releases on a successful PAM auth), so recovery
needs the serial console.

This is the *same* trap `bin/moarchy-system-lock` already guards against for the
swaylock path: it checks `passwd -S` and, when the state is not `P`, blanks the
panel (`moarchy-screen lock`) instead of taking a real ext-session-lock, exactly
because "a locked password authenticates nothing". The quickshell before-sleep
lock path has **no equivalent guard**, so it reintroduces the lockout the shell
script was written to avoid.

**Recovery (serial):** `printf 'moarchy:<pin>\n' | sudo -n chpasswd` over
`/dev/ttyACM0`, then type `<pin>` on the lockscreen. (One gotcha while doing
this: do not leave two `cat /dev/ttyACM0` readers running -- they split the
byte stream and both the command echo and its output come back garbled.)

**Proper fix, one or both:**
1. Guard the quickshell / `moarchy-lock` before-sleep path to skip a real lock
   when `passwd -S "$(id -un)"` is not `P` (blank instead), mirroring
   `bin/moarchy-system-lock`. This is the robust fix: it makes a locked-password
   device un-lock-out-able regardless of how the lock is triggered.
2. Have first-boot setup actually establish a PIN so the password is `P` from the
   start -- the "setup wizard" that is currently missing. `moarchy-firstboot`
   (groups/autologin) and `moarchy-user-setup` (app configs/theme) both run and
   stamp `firstboot-done` / `user-setup-done`, but **neither sets a PIN**, so
   there is no first-run step that moves the password off `L`. A device left on
   the shipped locked password is the exact condition that (1) must handle.

Until fixed, any full flash leaves the phone one sleep away from a serial-only
lockout.

---

## D31 -- some boots have no sound card: the ADSP's APR audio services never register {#d31}

**Status: OPEN -- important (found 2026-10-08, on the mic-fix kernel `d7f57bf`
flashed over the fp4-dev-all rootfs).** Intermittent: the boot straight after the
flash had a working card and captured audio; a later cold boot came up with
`/proc/asound/cards` = `--- no soundcards ---` and stayed that way.

It looks like D10 but is not D10. `deferred_probe_timeout=60` is on the cmdline
and the ADSP was **early**, not late:

```
[   16.569564] remoteproc remoteproc0: remote processor adsp is now up
[   78.910054] platform 33c0000.pinctrl: deferred probe pending: (reason unknown)
[   78.853687] platform 3370000.codec: deferred probe pending: va_macro: unable to get macro clock
```

The difference is one layer down. `apr`, `qrtr` and `qcom_pd_mapper` are loaded,
but **no APR service device ever appears**, so `q6afe`, `q6afe_clocks`,
`q6afe_dai` and `snd_q6dsp_common` are never autoloaded. Without q6afe-clocks the
LPASS clocks do not exist, the LPI pinctrl (`33c0000.pinctrl`) cannot probe, and
everything behind it (both macros, both SoundWire controllers, the sound card)
stays deferred until the timeout gives up. Writing `33c0000.pinctrl` to
`/sys/bus/platform/drivers_probe` afterwards does not bring it back, because the
missing supplier is the q6 clock service, not the pinctrl itself.

So on these boots the audio protection domain on the ADSP never announces its
services, although the ADSP remoteproc reports `running`.

**Candidate fix, untested:** upstream `b8e6fa9877f6` ("arm64: dts: qcom: sm6350:
Add memory-region for audio PD", Luca Weiss, on `sm6350-7.2.y` after the
`v7.2.0-sm6350` tag) reserves a remote heap for the ADSP's audio PD dynamic
loading and adds the LPASS/ADSP-heap VMIDs for ownership transfer. The pinned tag
lacks it. Bumping the kernel pin to the branch head picks it up; whether it cures
this needs a multi-boot count, since the failure is intermittent.

**2026-10-08, on the bumped kernel (sm6350-7.2.y @ 5ba18a1da713):**

- **`b8e6fa9877f6` does not prevent it.** In a 5-boot loop on the kernel that
  carries it, boots 1-3 had the card and all three q6 modules; boot 4 came up
  with no card and the identical signature. (The loop stopped there because
  that boot also never got an IPv4 lease; WiFi associated, only IPv6 came up.
  Whether that is related is open -- WiFi's QMI services go over the same QRTR
  bus as the audio PD's service registry.)
- **The APR bus is empty on a bad boot:** `/sys/bus/apr/devices` lists
  nothing, with `apr`, `qrtr` and `qcom_pd_mapper` loaded and the ADSP up at
  ~15 s. So the audio PD's services are never announced, rather than announced
  late.
- **Restarting the ADSP recovers it without a reboot:**

  ```
  echo stop  | sudo tee /sys/class/remoteproc/remoteproc0/state
  echo start | sudo tee /sys/class/remoteproc/remoteproc0/state
  ```

  Within 15 s `q6afe`, `q6afe_clocks`, `q6asm` and `q6adm` autoload, the
  card registers and `devices_deferred` empties -- the stuck devices probe
  even though the 60 s deferred-probe window has long expired. So this is a
  boot-time race in the audio PD's first start, not a lasting fault, and it
  points at a cheap mitigation (if no card ~90 s after boot, restart the ADSP
  once; the sensor core shares the ADSP, so iio-sensor-proxy blips).

**Root cause (2026-10-09): a module-load race, not the DSP.** On a DSP's
start, remoteproc adds a "pd-mapper" auxiliary device, and the separately
loaded `qcom_pd_mapper` module binds to it and starts the service-locator
(servloc) QMI server that the ADSP firmware queries for its audio PD. The ADSP
reaches that QMI service over the QRTR link that `qrtr_smd` provides. Nothing
orders those modules against `qcom_q6v5_pas` (no softdep anywhere upstream);
udev loads them on demand. Measured with the `module_load` tracepoint, armed
before udev coldplug, on 6 boots: `qcom_pd_mapper` loaded **40-235 ms after**
"adsp is now up" every time, and `qrtr_smd` after it too. The ADSP usually
waits long enough; when it does not, its audio services never register (no
APR devices, no `ssctl`) and there is no card.

**Fix:** `softdep qcom_q6v5_pas pre: qrtr qrtr_smd qcom_pd_mapper`
(moarchy-device-fp4, `/usr/lib/modprobe.d/moarchy-fp4-pd-mapper.conf`), so
modprobe/udev load them before the remoteproc driver. Verified on hardware:
across 10 reboots the order flipped to `qrtr_smd` and `qcom_pd_mapper` loading
~150-250 ms **before** "powering up adsp", and all 10 boots had the sound card.
No ADSP restart, so call audio is untouched. Before the fix D31 hit roughly
one boot in 10-15, so 10 clean boots supports the fix but does not yet prove it;
it moves to fp4-fixes.md after more boots in normal use.

**Workaround:** reboot, or restart the ADSP as above. **Diagnose a bad boot:** `lsmod | grep -E 'q6afe|apr'`
(apr present, q6afe absent = this defect) and
`sudo cat /sys/kernel/debug/devices_deferred` (the 33c0000.pinctrl chain).
