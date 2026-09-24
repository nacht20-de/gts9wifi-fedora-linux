# The fingerprint stack that works

The gts9wifi under-display reader (EgisTec EL721) enrols and authenticates
through the real libfprint/fprintd stack. Verified on 2026-09-24: all 17 of 17
`EL721_ENROLL_STAGES` accepted with coverage climbing to 100, a template stored
under `/var/lib/fprint/fedora/el721/gts9u-el721/`, and then `verify-match`
(`matched_slot=3`, score 284854528).

This file records the arrangement, because most of it lives in the gitignored
`ubuntu-galaxy-tab-s9-ultra/` tree — that is the Tab S9 **Ultra** port, vendored
here as a reference, and this port runs four of its files with local changes.

## The layers

| Layer | What it is |
| --- | --- |
| `egis_el721` | Kernel driver: 3.3 V rail, enable/reset, `/dev/esfp0`. Tracked here as `kernel/files/egis_el721.c`. |
| `fts1ba90a` | Kernel touchscreen driver. Tracked here as `kernel/files/fts1ba90a.c`. Supplies the fingerprint press edge. |
| `panel-samsung-ana38407` | Panel driver. Owns `fod_mode`/`fod_circle`/`fod_brightness`; `fod_mode` is HBM. |
| `libfprint` + `el721` driver | Fedora 44's own libfprint 1.94.100 with the vendored EL721 driver added. Built natively on the tablet. |
| `fprintd` | Fedora's fprintd 1.94.5. |
| `gts9u-fingerprint-overlay@agcarbajo` | GNOME Shell extension: draws the white target and pulses the UI lease. |
| `ubuntu-gts9u-fingerprint-ui` | Root D-Bus broker that turns the Shell's availability into the lease file fprintd reads. |

The trustlet needs a **UI lease** before it will arm: `/run/gts9u-fingerprint-ui/ready`
must read `ready <CLOCK_MONOTONIC microseconds>`, no more than 2 s ahead. The
Shell extension asks the broker, and the broker writes it. The extension shows
the white target only while the driver publishes `/run/gts9u-fingerprint/active`.

## Seven adaptations to the vendored tree

All seven are required. Each was measured, not guessed.

### 1. `libfprint/drivers/el721.c` — point at this tablet's touch controller

The reference hardcodes the Ultra's Goodix controller; this tablet's is the
FTS1BA90A.

```diff
-#define EL721_TOUCH "/sys/bus/i2c/devices/13-005d"
+#ifndef EL721_TOUCH
+#define EL721_TOUCH "/sys/bus/i2c/devices/10-0049"
+#endif
```

### 2. `libfprint/drivers/el721-touch-light.h` — settle long enough for a finger

180 ms covers the Shell observer and a few compositor frames: it is a *light*
settle, not a finger settle. A capture that early lands while the fingertip is
still arriving and scores low coverage. 1200 ms gives a **still** finger time to
settle and flatten against the glass, which matters because a still press cannot
make up for an early capture by moving. Measured with a firm still press: at
700 ms every attempt ran the TA's full four capture steps and an action of 10
attempts produced no match at all; at 1200 ms one-step frames appear and matches
arrive.

```diff
-#define EL721_LIGHT_SETTLE_US (180 * 1000)
+#ifndef EL721_LIGHT_SETTLE_US
+#define EL721_LIGHT_SETTLE_US (1200 * 1000)
+#endif
```

### 3. `libfprint/drivers/el721.c` — action timeout long enough for an enrolment

`EL721_ACTION_TIMEOUT_US` is a whole-action deadline (checked against
`action_started`), not a per-capture one, so 90 s must cover all 17 stages —
about 5 s per press. 300 s is comfortable.

```diff
-#define EL721_ACTION_TIMEOUT_US (90 * G_USEC_PER_SEC)
+#ifndef EL721_ACTION_TIMEOUT_US
+#define EL721_ACTION_TIMEOUT_US (300 * G_USEC_PER_SEC)
+#endif
```

### 4. `gnome-shell/extensions/gts9u-fingerprint-overlay@agcarbajo/` — this panel

`geometry.js` originally carried the Ultra's overlay proportions, which on this
2560x1600 panel put the target at x=2424, where a pressing finger only ever
reached coverage 35 — the fingertip was clipping the sensor. It was then moved
to x=2300 with a widened 200 px target, where a hand-built harness reached
88–100. Both were superseded on 2026-09-24 by measurements of the reader itself
(see "Where the reader is" below): the target now sits on the measured centre
and is sized to Samsung's own FOD active area.

**Everything must stay a fraction of the panel.** The Shell hands the extension
the *logical* monitor, and this panel runs at scale 2, so the logical axes
(1280x800) are half the physical ones. An intermediate revision of this file
expressed the geometry in physical millimetres and divided by those logical
axes, which doubled the inset and parked the target in the bottom-right corner —
the tap that diagnosed it landed at display (2160, 1600) against a sensor at
(2360, 800). Anything written here in pixels must be divided by the panel's own
dimensions first, never by `monitor.width`/`height`.

```diff
-    const diameter = Math.round(shortAxis * 14.8 / 196);   /* Ultra's overlay */
-    const inset = shortAxis * 16.7 / 196;
+    const PANEL_SHORT = 1600;          /* physical panel, portrait */
+    const PANEL_LONG = 2560;
+    const PX_PER_MM = 1000 / 92.55;
+    const SENSOR_CX = 800;             /* centred across the short axis */
+    const SENSOR_INSET_MM = 18.5;      /* from the short edge */
+    const SENSOR_DIAMETER_MM = 14.8;   /* Samsung's FOD active area */
...
+    const x = 1 - inset / PANEL_LONG;
+    const y = cx / PANEL_SHORT;
+    const size = Math.round(diameter / PANEL_SHORT * shortAxis);
```

`extension.js` passes an optional third argument so the target can be moved
without a logout — the Shell caches extension modules (proved with an atime
probe: `disable`/`enable` does not re-read `geometry.js`), so editing this file
otherwise costs a login. The override is read from
`/run/gts9u-fingerprint/sensor-geometry` as `cx cy diameter` in **portrait panel
pixels** (e.g. `800 2360 160`), cached for a second, and only accepted when it
could describe this panel's reader: on the short axis, in the last third of the
long axis, and no wider than half the short axis. Anything else falls back to
the measured defaults — which is what makes the stale logical-units file
(`400 1180 80`) written during the debugging above harmless. `/run` is tmpfs, so
the override does not survive a reboot.

**Measured result of the corrected geometry (2026-09-24).** Enrolment completed
with coverage 100 and all 17 samples accepted in about two minutes, where the
previous two attempts stalled at coverage 6 with 2 of 17 stages over 20 minutes
— because the sensor had been sitting one pixel inside the illuminated circle.
A still-press verify then matched on the first press (`matched_slot=3`,
`steps=4,5:0,87:0,6:0,0:0`, where a single `87` iteration is the good-frame
signature and four of them is the failure signature), and the GNOME lock screen
unlocked with the fingerprint.

### Where the reader is, and why the target matters

The sensor's centre is at portrait **(800, 2360)** on the 1600x2560 panel —
centred across the short axis and 200 px (18.5 mm) in from it, i.e. landscape
**(2360, 800)**. Three independent sources agree: the LineageOS overlay for this
tablet (`DaViDev985/device_samsung_gts9wifi`:
`physical_fingerprint_sensor_center_screen_location_y = 2360px`,
`pixel_pitch = 92.55`); the touchscreen's own `fod_state`, which reports real
presses at y 2357; and the stock firmware's device tree (`sec,max_coords =
1752x2800` on the touch node, `samsung,support-optical-fingerprint` on the panel
node). Samsung exposes the same geometry in millimetres at
`/sys/class/fingerprint/fingerprint/position` — `marginBottom, marginLeft,
areaWidth, areaHeight, imageSize, activeArea, draggingArea` — with the sysfs
class provided by `fingerprint_sysfs.ko` and the attribute registered by the
touchscreen driver. This port's ported FTS driver has no such attribute and the
stock values are not in the device tree, so the numbers above come from the
overlay plus measurement.

**The panel has no local FOD illumination.** Measured 2026-09-24 with the Shell
overlay disabled and the panel held in FOD mode: cycling `fod_mode` 10 s on /
10 s off makes the whole screen pulse, and the DDIC's `fod_circle` command
lights no visible region at all. The only bright area on screen during a
fingerprint action is therefore this extension's white target plus the shade
that dims everything outside it — which is why the target's position and size
are load-bearing: the sensor has to sit inside the illuminated circle. At the
old geometry (centre y 2300, 200 px, so 2200–2400) the sensor (centre 2360,
~79 px, so 2321–2399) was inside by a single pixel at its far edge.
`fod_circle_store` refuses to enable the circle unless `fod_mode` is already on
(`if (enable && !ctx->fod_mode) ret = -EINVAL`), so the circle cannot be tested
in isolation without a kernel change; Samsung's own local-HBM scheme — a mask
brightness for the non-FOD area plus the DRM `finger_mask` connector property —
is what would remove the whole-screen flash while keeping the sensor lit.

`metadata.json` also needs this tablet's Shell, and `extension.js` needs one
GNOME 50 fix — GNOME 50 dropped the `affectsInputRegion` parameter of
`addTopChrome()`, and the extension fails to enable with
`Error: Unrecognized parameter "affectsInputRegion"`. All three actors are
already `reactive: false`, so the parameter is redundant.

```diff
-        Main.layoutManager.addTopChrome(this._shade,
-            {trackFullscreen: true, affectsInputRegion: false});
+        Main.layoutManager.addTopChrome(this._shade,
+            {trackFullscreen: true});
```

```diff
-    "shell-version": ["46"],
+    "shell-version": ["46", "49", "50"],
```

GNOME Shell caches extension ES modules **in process**, so every one of these
extension changes needs a fresh Shell — a log out and back in on Wayland.
Disabling and re-enabling re-runs the cached code and changes nothing.

## The kernel half: where the press edge comes from

Samsung reports a fingerprint press as a **low-power gesture** frame
(`FTS_GESTURE_EVENT`, stype `FTS_SPONGE_EVENT_PRESS`), and the controller only
runs that scan while the display is **off**. With the panel lit, no gesture
frame ever arrives: measured with the driver armed and a finger held on a lit
reader, `fod_state` stayed at sequence 0 for an entire 300 s action. The
vendor's `FTS_MODE_PRESS` makes this worse, because it also suppresses ordinary
contact reporting — the one channel that still carries the contact.

So `fts1ba90a.c` claims the reader without entering the vendor's FOD mode, and
`fts1ba90a_fod_contact()` takes the press edge from the **ordinary coordinate
event**:

- `fod_enable` still reads back `1`/`0` — that readback is all the libfprint
  driver checks — but writing `1` clears the vendor FOD mode instead of setting
  it;
- a press bumps `fod_sequence` and sets `fod_pressed`, which is exactly the edge
  the driver polls `fod_state` for;
- the frame is **consumed, never reported to input**, so pressing the target
  cannot also open the desktop's long-press context menu over the illuminated
  circle;
- `fod_rect` and `fod_state` exist with the interface the driver expects
  (`fod_state` reads `pressed|released X Y SEQ`); `fod_rect` is a kernel-side
  filter only — see "The sensor window" below.
- a contact outside the sensor window is **not** consumed: it is handed to
  input as an ordinary touch, so the screen stays usable while a prompt is
  armed.

Deploy it by swapping the loadable module — no reflash:

```sh
cp kernel/files/fts1ba90a.c ~/kbuild-gts/linux-7.2/drivers/input/touchscreen/
make -C ~/kbuild-gts/linux-7.2 ARCH=arm64 LLVM=1 KBUILD_MODPOST_WARN=1 \
     drivers/input/touchscreen/fts1ba90a.ko
zstd -19 -f fts1ba90a.ko -o fts1ba90a.ko.zst
cp fts1ba90a.ko.zst /lib/modules/$(uname -r)/kernel/drivers/input/touchscreen/
depmod -a && modprobe -r fts1ba90a && modprobe fts1ba90a
```

The `vmlinux.o is missing` / unresolved-symbol warnings from modpost are the
known benign artifact — the symbols resolve against the running kernel.

## Runtime requirements

- **fprintd needs the TEE device nodes.** Fedora's unit sets `DevicePolicy=auto`
  with an allow-list that omits `/dev/tee0`, and the denial surfaces as
  `cannot open the QTEE object namespace: Operation not permitted` — which reads
  like a TEE fault and is not one. A drop-in with
  `DeviceAllow=/dev/tee0 rw` (plus `/dev/spcom`, `/dev/qsee_ipc_irq_spss`,
  `/dev/dma_heap/qcom,secure-sp-tz`, `/dev/esfp0`, `/dev/k250a`) and
  `ReadWritePaths=/var/lib/gts9u-fingerprint` fixes it. It belongs in
  `rootfs/overlay`, not `/run`.
- **polkit gates enrolment.** `net.reactivated.fprint.device.enroll` is denied
  for a transient user scope; it needs an active graphical session or a rule.
- **The panel must be bright enough and awake.** Screen blanking mid-capture
  latches the trustlet.
- **SELinux**: both runtime paths are labelled `gts9u_fingerprint_runtime_t` by a
  CIL module shipped at `rootfs/overlay/usr/share/gts9wifi/selinux/`. This port's
  systemd and services run in `kernel_t` / `kernel_generic_helper_t` rather than
  `init_t` / `unconfined_service_t`, so no `*_var_run_t` interface applies and
  **both** writes were denied — the broker's lease and the driver's active-lease
  file — invisibly, because the port runs permissive. Install with
  `semodule -i gts9u_fingerprint.cil` and
  `restorecon -R /run/gts9u-fingerprint /run/gts9u-fingerprint-ui`. Verified: 0
  denials on the type for either writer, with the file correctly labelled.

  Two traps found while doing this. The module must be **CIL**, not `.te`: on
  this system `checkmodule` cannot resolve even `unconfined_t` from the devel
  headers, so the traditional module toolchain does not work here. And
  `SELinuxContext=` on the unit is *not* a fix — the transition itself is denied
  (`nnp_transition`, because the unit sets `NoNewPrivileges`), which permissive
  hides and enforcing would not.

  The port is still nowhere near enforcing: this boot logged 33,265 denials,
  most of them `{ search }` on `unlabeled_t` for `/usr/bin` and `/usr/lib64`,
  i.e. the rootfs is largely **unlabelled**. That is a port-wide item, not a
  fingerprint one.

## The trustlet latch

The trustlet latches on the first frame it **rejects** (result 39 `BAD_QUALITY`).
While latched, every later reply carries status `0x88000000` and `coverage=0`,
which looks exactly like an illumination failure. Two consequences:

- a clean session reports `status=0` on opcodes 4/5/87; `0x88000000` there means
  degraded, and any coverage measurement taken in that state is meaningless;
- **a new session clears it.** No reboot is needed — contrary to an earlier
  reading, a fresh fprintd action starts healthy.

Because a rejected first frame latches the session, the first press matters:
place the finger deliberately, and vary placement between presses. Roughly three
near-identical frames in a row also end the attempt with result 70.

### 5. `libfprint/drivers/el721.c` — a retry only for a frame the TA rejects

The identify path hands the matcher one frame per attempt and has no coverage
feedback, so a frame that is not good enough can read as a plain `NO_MATCH`
(result 32) rather than `BAD_QUALITY` (39/41) — and cannot be told apart from a
different finger.

An earlier version of this adaptation therefore retried *any* no-match inside
the same action, rebuilding the QTEE session each time and bounding the loop
with its own `EL721_MATCH_RETRIES` budget. **That was reverted, and should not
be reinstated.** fprintd turns a retry into its own translatable notice —
*"Place your finger on the reader again"* — which the user read as the reader
rejecting a press that was fine ("before it never told me to place my finger
again"), and it arrives exactly while the driver is rebuilding the secure
session with the touch trigger disarmed, so the press that follows is dropped.
A no-match is now **terminal**: the action ends and the lock screen starts a
fresh verify. `operation_prepare` is back to restoring only the sensor power
that `open()` deliberately dropped.

What remains is a retry for the case the TA *does* flag — `BAD_QUALITY`:

```c
if (reply.result == 39 || reply.result == 41)
  {
    GError *retry = fpi_device_retry_new (FP_DEVICE_RETRY_GENERAL);
    ...
    action_cleanup (self);
    el721_complete_match_retry (FP_DEVICE (self), current, retry);
  }
```

libfprint permits exactly **one** result per action, retries included, so the
retry has to clean up first and let fprintd start the next action; rearming here
left `result_reported` set and made the next physical contact violate that
contract. Enrolment reports the same retry through `fpi_device_enroll_progress`,
which is what a marginal fingertip needs — one measured 17-stage run took 55
samples.

### 6. `libfprint/drivers/el721.c` — hold the light across a press, drop it per sample

HBM on this DDIC is a **global** panel change, not a spot: entering FOD mode
moves DCS `0x51` from the user's brightness to the FOD level
(`ANA38407_FOD_BRIGHTNESS` 1623), and the GNOME Shell overlay compensates with a
black shade whose opacity is derived from the backlight ratio. That shade can
only follow *after* the panel has changed — the overlay polls `fod_mode` every
50 ms — so every HBM transition brightens the whole display for about three
frames before the shade catches up. Fewer transitions, fewer flashes.

Two rules, each measured against the alternative:

- **an early release keeps the light** (`touch-to-light held; finger released
  early`). Dropping HBM and re-entering it on the next press is two flashes where
  one would do, and the press is over by then anyway.
- **an enrolment capture drops the light** — `fod_mode` goes 1 → 0 → 1 once per
  sample. This is the opposite of what the retry-era code did, and it was
  measured: holding HBM across a whole enrolment stalled coverage at 6 with 1–2
  of 17 stages accepted over 55 samples, while cycling it per sample reached
  coverage 100. It is also the operator's only cue that the reader took its
  sample — the white circle going dark — which the user noticed the moment it
  disappeared.

An earlier version took a `keep_light` parameter through `udfps_end` so that
retry restarts inside one action would not re-enter HBM. That went with §5: with
no in-action retry, every action end really is an end, so `udfps_end` drops the
light unconditionally again.

The panel's own 15 s watchdog still applies: it returns the panel to normal
brightness if the driver stops refreshing, which bounds a held light even if
userspace dies.

### 7. `libfprint/drivers/el721-identify-wire.h` — a press counts even if the finger was already down

The capture rule used to be `opcode == 4 && edge && pressed && !was_pressed`, and
the poll seeds `finger_present` from the panel state the moment the action's
target becomes available. That combination silently threw away the **first
press of every action**: a user putting their finger down as the prompt appears
is already "pressed" when the driver arms, so the press event the controller
reports for it was rejected as a duplicate.

Measured on the tablet, in one unlock attempt:

```
02:00:21  contact pressed=1 released=0 sequence=694 delta=1 capture=0   <- press ignored
02:00:22  contact pressed=1 released=0 sequence=696 delta=1 capture=1   <- only after lifting
02:00:32  contact pressed=1 released=0 sequence=699 delta=1 capture=0   <- retry, ignored again
02:00:41  contact pressed=0 released=1 sequence=700 delta=1 capture=0
02:00:42  contact pressed=1 released=0 sequence=701 delta=1 capture=1
```

The user had to lift and press again for *every* attempt, which is the reported
"I have to press two times" — and with 2–4 attempts needed to match, it is also
what made an unlock fail. The rule no longer consults the latch:

```diff
-  return opcode == 4 && edge && pressed && !was_pressed;
+  (void) was_pressed;
+  return opcode == 4 && edge && pressed;
```

A finger that is merely *resting* emits no event at all, so `edge && pressed`
still means exactly one capture per press edge — the guarantee the latch was
there to protect. The cost is that a tap *outside* the sensor also burned one
retry, because this port's kernel trigger published a FOD press edge for any
ordinary panel touch. That is now fixed by gating the trigger by the sensor
window — see the next section.

## The sensor window: a kernel-side filter, never a firmware one

Consuming every contact while the prompt was armed had two measured
consequences: the screen was dead to the user (a tap on "use password" did
nothing), and every such tap burned a verify attempt and reported a no-match —
**101 of 311 captures in one evening were nowhere near the sensor**, one of
them at raw `x=325 y=1580`. The trigger now gates presses by the sensor window
the libfprint driver publishes and hands everything else to input. Three
properties make that safe.

**The window is never forwarded to the controller.** The vendor driver writes
it to the controller at sponge offset `0x4b`, and that write killed the reader
once already: the controller then applies its own fingerprint handling inside
the window, which it only reports through the low-power gesture scan it runs
with the display *off*, so the ordinary coordinate channel carrying the contact
goes silent. Measured with a valid window written, the driver armed at
02:26:32 and logged **no contact at all for the next 28 s** while the finger
pressed the sensor. `fod_rect_store()` therefore stores the window and never
touches the i2c bus — verified by disassembling the built module, where
`fod_rect_store` calls only `sscanf` and `_dev_info`.

**A window that cannot describe this panel filters nothing.** The reference
port inherited the Tab S9 Ultra's `854 2732 994 2872` verbatim, whose y range
2732-2872 starts past the bottom of this 1600x2560 panel; gating by it would
silently reject every real press. An impossible window — all zero, inverted, or
past `FTS_MAX_X`/`FTS_MAX_Y` — is treated as "no filter", which also makes the
kernel and driver halves independently deployable: the module can be loaded
while the old driver still publishes the Ultra window.

**The window matches the reader.** `EL721_FOD_RECT` is `680 2240 920 2480`:
the measured reader centre at portrait (800, 2360) with 120 px of margin around
it for where a fingertip actually lands.

Verified on the tablet 2026-09-24, with the trigger armed by a real lock-screen
verify:

| | measured |
|---|---|
| driver publishes the window | kernel logs `fingerprint FOD rect 680 2240 920 2480` |
| an on-sensor press | `EL721 contact pressed=1 ... x=771 y=2355 capture=1` — inside the window, captured, `verify result=0 matched_slot=3` |
| an off-sensor tap **while armed** | reaches input — `ABS_MT_POSITION_X=412 Y=283` appears in the event stream instead of being consumed |
| the press itself | produces **no** input event, so there is no stray click under the circle |

Verified on the tablet: three attempts produced **three** captures from three
presses — one press each, with no discarded press and no lift-and-press — and
the third matched (`verify result=0`, `matched_slot=3`, score 217748224,
`steps=4,5:0,87:0,87:0,6:0,0:0`).

## Reading the reply: the capture-step count

A useful quality signal falls out of the identify reply: how many
`CAPTURE_STEP` (87) iterations run before `CAPTURE_SUCCESS` (6).

- a good frame terminates on the first step — `steps=4,5:0,87:0,6:0,0:0`
- a frame the TA cannot use runs the full four — `steps=4,5:0,87:0,87:0,87:0,87:0,6:0,0:32`

In the run that matched, the successful attempt was the only one-step capture.
This is the closest thing to the coverage feedback the identify path lacks, and
it is what to watch when tuning capture.

## Known gap: per-frame capture quality, and what fixed most of it

This sensor images a **patch** of the fingerprint per frame. Rolling sweeps more
of the print across it (that is how enrolment reached coverage 100), so a rolled
template and a still probe disagree — the probe covers only part of the template
and about four frames in five no-match. Measured with a rolled template and a
still press: 5 attempts to a match, and one action of 10 attempts that never
matched at all.

What closes the gap is enrolling the way the sensor will actually be used:

- **Enrol with still presses**, shifting position and finger part between presses
  (pad, tip, up, down, left, right). Coverage still climbs to 100 because each
  press contributes a different patch, and the resulting template agrees with a
  still probe. Measured: 17/17 stages, coverage 100, zero FATALs.
- **Press on the sensor's centre.** The circle is 200 px across but the active
  area is smaller; presses 50 px off centre measurably hurt.
- **Press firmly and hold ~2 s.**

With a still-press template, two consecutive still-press verifications matched in
**3 and 4 attempts** (capture steps 4,3,1 and 4,4,4,2 — the good frame is the one
that terminates in one step). No rolling needed.

Still-press enrolment has one trap: poor frames from the same spot look alike,
which trips the trustlet's ~3-identical-frames abort (result 70) and loses the
whole action, because the accepted count does not survive a session. Varying the
finger part and position between presses is what avoids it — a run that pressed
the same spot stalled at coverage 6 with 3 accepted and then aborted.

The no-match scores are all near `0x8A...` (9070848, 9085184, 9087232, 9089280)
while real matches scored 83538176 and 284854528 — so the score field is
meaningless on a no-match, exactly as the vendored header warns. Never read a
verdict from it; read `result` (0 = match, 32 = no-match) and `matched_slot`.

## Enrolling from GNOME Settings: why the "Scan new fingerprint" tile did nothing

On GNOME 50 (Fedora 44 ships `gnome-control-center` 50.4) pressing **Scan new
fingerprint** in Settings did nothing at all: no error, no finger list, and no
`EnrollStart` reaching fprintd. The failure is entirely in Settings' UI layer —
the reader, fprintd, polkit and the port's driver are all fine, and
`fprintd-enroll` worked the whole time.

### What was actually happening

Instrumenting the dialog (`patches/gnome-control-center-50.4-fingerprint-add-tile.patch`
started life as this instrumentation) showed the popover being mapped and
unmapped in the same instant:

```
FP-POPOVER add-finger popover MAPPED
FP-POPOVER add-finger popover UNMAPPED
```

The "Scan new fingerprint" tile opens a `GtkPopover` holding one button per
finger. Ten entries in a plain vertical box made the popover taller than the
space available around the tile — the dialog is a bottom sheet 796x644 inside a
1280x800 logical desktop, and the gallery sits at y 290-364 of it — so the
compositor could not place the popup and dismissed it immediately. Because the
popover never stayed mapped, clicking the tile looked like a dead button.

The finger tiles in the same gallery worked because each of their popovers holds
a single entry.

### The fix

`patches/gnome-control-center-50.4-fingerprint-add-tile.patch`, applied through
`Patch0` in the package spec and built with `rpmbuild -bb`:

* the popover's content is wrapped in a `GtkScrolledWindow` with
  `max-content-height: 260` and `propagate-natural-height: true`, so the popover
  is always small enough to be placeable;
* `position: bottom` is dropped so GTK may flip the popover into view, as it
  does for the finger tiles;
* `autohide: false`, so the popover cannot be dismissed by the very click that
  opened it, with `on_enroll_cb()` popping it down once a finger is chosen;
* `on_print_activated_cb()` calls `gtk_popover_popup()` explicitly, so clicking
  the tile body works and not only the small ⊕ icon.

Upstream reached the same conclusion by redesign: GNOME 51 has no
`add_print_popover` at all, replacing it with an `add_finger_button` and
`AdwPreferencesGroup` finger rows on a separate page. This patch is the minimal
backport of that idea to 50.4.

### Building and installing it

```sh
sudo dnf builddep -y /home/fedora/rpmbuild/SPECS/gnome-control-center.spec
dnf download --source gnome-control-center --enablerepo=fedora-source,updates-source
rpm -i --define "_topdir $HOME/rpmbuild" gnome-control-center-50.4-1.fc44.src.rpm
# add Patch0: gnome-control-center-50.4-fingerprint-add-tile.patch to the spec,
# put the patch in ~/rpmbuild/SOURCES, then
rpmbuild -bb ~/rpmbuild/SPECS/gnome-control-center.spec
sudo rpm -Uvh --replacepkgs --replacefiles \
  ~/rpmbuild/RPMS/aarch64/gnome-control-center-50.4-1.fc44.aarch64.rpm
```

The build takes about two minutes on the tablet (780 ninja targets, 8 cores).
`%autosetup -p1` picks the patch up automatically. The original binary is kept
at `/root/gnome-control-center.bak-orig`, and `dnf reinstall gnome-control-center`
restores the packaged build — note that a Fedora update of the package will
replace the patched binary and the tile will break again.

### Verifying it

The dialog's popover is invisible to `fprintd` and to the journal unless it is
instrumented, and a GNOME Shell extension does not need a logout to be toggled.
The quickest check is the one used here: launch Settings on the users panel,
open the fingerprint dialog and click the tile — the finger list should drop
down. Reading the AT-SPI tree is also a reliable way to see whether the popover
actually mapped, since an unmapped popover contributes no accessible nodes.

## Unlock latency: where the time actually goes

Measured on the tablet 2026-09-24 from fprintd's own timestamps, a successful
lock-screen unlock:

| leg | before | after |
| --- | --- | --- |
| press detected -> frame taken | 1.218 s | 0.718 s |
| frame taken -> match reported | 0.462 s | 0.462 s |
| **press -> match** | **1.69 s** | **1.20 s** |

and, separately:

* power key -> the sensor accepts a press: ~4.3 s. Of that, ~2.4 s is the lock
  screen coming up before PAM starts fprintd, then ~0.7 s to Claim, then ~1.2 s
  for the secure-world open (`lookupTA(dualfp) returned 23; the TA was not
  resident and is being loaded` plus the BAUTH prepare and optical blob upload).
  This is the cost of the port's QTEE path, not of the settle.
* match -> the greeter session is gone: ~2.2 s, which is GNOME Shell's own
  unlock transition and not something this port controls.
* match -> `report_verify_status: verify-match`: 3 ms.

So the one leg this port owns is the illumination settle, and it was carrying
about 500 ms of pure margin.

### Why the settle was 1200 ms and why verification does not need it

`EL721_LIGHT_SETTLE_US` exists because a frame taken too early lands while the
fingertip is still flattening: the constant's own note records a capture 180 ms
after the press edge scoring a maximum coverage of 35/100, against 88-100 once
the wait reached 700 ms. 1200 ms is that measurement plus margin.

Enrolment keeps the reference value, because a marginal sample there costs a
whole stage. Verification does not: it needs one frame good enough for the
matcher, and the matcher already retries internally (the `87` CAPTURE_STEP
iterations), so a finger that is still settling keeps being sampled at later,
better-settled moments inside the same action. Verification therefore settles
for `EL721_LIGHT_SETTLE_VERIFY_US`, 700 ms.

Measured after the change: three unlocks matched at 1.20-1.23 s from press,
each on a single `87` iteration. Two presses in the same run failed with
`result=32` and four rejected iterations — but that is the same signature as the
failure seen at 1200 ms earlier the same day, so the shorter settle is not what
causes it. `patches/el721-verify-settle.patch` is the change.

### Tuning it without a rebuild

Both settles read an environment override, which matters because fprintd is
D-Bus activated and restarted per operation:

```sh
sudo mkdir -p /etc/systemd/system/fprintd.service.d
sudo tee /etc/systemd/system/fprintd.service.d/10-settle.conf >/dev/null <<'CONF'
[Service]
Environment=EL721_LIGHT_SETTLE_VERIFY_MS=1200
CONF
sudo systemctl daemon-reload && sudo systemctl restart fprintd
```

`EL721_LIGHT_SETTLE_MS` applies to every action, `EL721_LIGHT_SETTLE_VERIFY_MS`
only to verification and identification, and a value that is not a positive
number of at most 5 s is ignored. Confirm it reached the daemon with
`sudo cat /proc/$(systemctl show -p MainPID --value fprintd)/environ | tr '\0' '\n' | grep EL721`.
Raise it if presses start failing to match; the reference 1200 ms is the
known-good upper bound.

### fprintd sandbox allowances must live outside /run

`fprintd` needs the EL721 sandbox allowances to open the QTEE side of the
stack. They were present only as
`/run/systemd/system/fprintd.service.d/el721-qtee.conf`, which is tmpfs: nothing
shipped it (not owned by any package) and nothing recreated it, so the next
reboot would have dropped every `DeviceAllow` and `ReadWritePaths` entry and the
reader would have failed with the EPERM "cannot open the QTEE object namespace"
error. That hand-written drop-in is now installed persistently as
`/etc/systemd/system/fprintd.service.d/10-gts9u-el721.conf`, and the `/run` copy
is gone, so what is live is exactly the post-reboot configuration.

Note that this is *not* the same file the repo ships for the device package at
`ubuntu-galaxy-tab-s9-ultra/packaging/ubuntu-gts9u-device/usr/lib/systemd/system/fprintd.service.d/10-gts9u-el721.conf`.
That variant also carries `Requires=`/`After=` on
`ubuntu-gts9u-qcomtee.service`, `ubuntu-gts9u-fingerprint-ui.service` and
`ubuntu-gts9u-fingerprint-secure.service`, plus
`ExecStartPre=/usr/libexec/ubuntu-gts9u-fingerprint-secure --wait` and
`StateDirectory`/`RuntimeDirectory`/`ExecStopPost` lines, and those services are
not installed on this tablet. Installing it as-is would make fprintd depend on
units that do not exist, so the minimal proven content was persisted instead:

```ini
[Service]
DeviceAllow=/dev/tee0 rw
DeviceAllow=/dev/spcom rw
DeviceAllow=/dev/qsee_ipc_irq_spss rw
DeviceAllow=/dev/dma_heap/qcom,secure-sp-tz rw
DeviceAllow=/dev/esfp0 rw
DeviceAllow=/dev/k250a rw
ReadWritePaths=/var/lib/gts9u-fingerprint
```

Verify with `systemctl cat fprintd` (only the packaged unit, this drop-in and
the systemd timeout fragment should be listed) and by running `fprintd-verify`
until the journal logs `EL721 verify armed`, which only happens once the
sandbox permits the device open.

A related leftover was removed at the same time:
`/run/systemd/system/fprintd.service.d/debug.conf` set `G_MESSAGES_DEBUG=all`,
which made the journal chatty. Drop-ins under `/run` are easy to leave behind
while debugging; check `systemctl cat fprintd` for them afterwards.
