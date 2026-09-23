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

## Four adaptations to the vendored tree

All four are required. Each was measured, not guessed.

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
still arriving and scores low coverage (max 35/100 over a whole run). 700 ms is
what the hand-built harness used, and it lets the operator press firmly and roll.

```diff
-#define EL721_LIGHT_SETTLE_US (180 * 1000)
+#ifndef EL721_LIGHT_SETTLE_US
+#define EL721_LIGHT_SETTLE_US (700 * 1000)
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

`geometry.js` carries the Ultra's overlay proportions. On this 2560x1600 panel
they put the target at x=2424, where a pressing finger only ever reached
coverage 35 — the fingertip was clipping the sensor. x=2300 is where the harness
reached 88–100. The target is also widened to 200 px so a fingertip lands fully
on the sensor.

```diff
-    const diameter = Math.round(shortAxis * 14.8 / 196);
-    const inset = shortAxis * 16.7 / 196;
+    const diameter = Math.round(shortAxis * 200 / 1600);
+    const inset = shortAxis * 260 / 1600;
```

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
  (`fod_state` reads `pressed|released X Y SEQ`).

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
- **SELinux logs denials** for the lease broker writing
  `/run/gts9u-fingerprint-ui` (`permissive=1`, so nothing is blocked yet). It
  needs a proper label.

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

### 5. `libfprint/drivers/el721.c` — retry a no-match in a fresh session

The identify path hands the matcher one frame per attempt and has no coverage
feedback, so a frame that is not good enough reads as a plain `NO_MATCH`
(result 32) rather than `BAD_QUALITY` (39/41) — and cannot be told apart from a
different finger. Reporting `FPI_MATCH_FAIL` on that first frame is what made
verification a ~20 %-per-press lottery.

The driver now reports a **retry** instead, which fprintd answers by restarting
the verify. Two parts matter:

```diff
+      if (self->match_retries < EL721_MATCH_RETRIES)
+        {
+          GError *retry = fpi_device_retry_new (FP_DEVICE_RETRY_GENERAL);
+          ...
+          el721_complete_match_retry (FP_DEVICE (self), current, retry);
```

and, because fprintd restarts **without claiming the device again**, the QTEE
session survives — and the TA's matcher answers once per identity per session,
repeating that answer for every later capture in the same session. Measured: 21
captures from one press inside one session returned one identical reply and none
matched, while the same finger matched across separate sessions. So
`operation_prepare` closes the session and opens a new one per action:

```diff
-  /* ... only restore the sensor power that open() deliberately dropped. */
-  return set_sensor_power (self, TRUE, error);
+  if (!set_sensor_power (self, TRUE, error))
+    return FALSE;
+  el721_qtee_close (self->qtee);
+  self->qtee = el721_qtee_open (EL721_FIRMWARE, error);
+  if (!self->qtee || !el721_qtee_prepare (self->qtee, error))
+    return FALSE;
```

Verified: 10 attempts in one action produced 9 retries, 9 session rebuilds and
9 fresh actions. The retry budget (`EL721_MATCH_RETRIES`, 9) is the driver's,
because fprintd has no retry limit of its own.

## Reading the reply: the capture-step count

A useful quality signal falls out of the identify reply: how many
`CAPTURE_STEP` (87) iterations run before `CAPTURE_SUCCESS` (6).

- a good frame terminates on the first step — `steps=4,5:0,87:0,6:0,0:0`
- a frame the TA cannot use runs the full four — `steps=4,5:0,87:0,87:0,87:0,87:0,6:0,0:32`

In the run that matched, the successful attempt was the only one-step capture.
This is the closest thing to the coverage feedback the identify path lacks, and
it is what to watch when tuning capture.

## Known gap: per-frame capture quality

The retry machinery is in place and verified, but what still limits verification
is how often a single capture is good enough: one measured action had 10
attempts and **not one** good frame, and the budget ran out (the client then sees
a real `verify-no-match`). Across separate actions the rate has measured between
roughly one in five and one in ten.

The no-match scores are all near `0x8A...` (9070848, 9085184, 9087232, 9089280)
while real matches scored 83538176 and 284854528 — so the score field is
meaningless on a no-match, exactly as the vendored header warns. Never read a
verdict from it; read `result` (0 = match, 32 = no-match) and `matched_slot`.

This is a capture problem, not a plumbing one: a still finger covers the sensor
poorly (still-finger runs measured coverage 2–35, while rolling reached 100), so
pressing **and rolling** is what produces a good frame. The open questions are
whether a longer settle gives the roll time to land, and whether the retry budget
should be larger — both are cheap to measure now that the step count is visible.
