# Smart docking (WIP)

Goal: make grab/dock diagnose-and-retry instead of ram-and-hope. Two observed
failure modes, same root cause (open-loop push to a fixed `select_x`, then a
blind servo, with no verification):
- **too shallow** → the locking pin can't rotate → head falls off after pull-out
- **too deep** → carriage rams the dock's back wall → X/Y lose steps (corrupts
  position, and on CoreXY poisons everything after)

## Mechanism (confirmed on-machine)
- The **servo locking-pin rotation** latches the head; it only rotates if the
  carriage is pushed in far enough (X-depth is a *precondition*, not the latch).
- **`carriage_tool_sensor` after the pull-out** is ground truth that the pin
  actually latched — distinct from "head is touching."
- The pull-out already happens (`G0 X{select_x + 8}`), so verifying is nearly
  free: read the sensor there.

## Why this may fix more than grab reliability
PH1 is the **reference head**: the print-start `TAP` sets the first-layer Z with
PH1, and `TOOL_LOCATE_SENSOR` measures every other head's offset relative to PH1.
So if PH1's *seated position on the carriage varies pickup-to-pickup* (the same
inconsistency behind falls-off-vs-wall), that variation propagates into:
- **non-repeatable tool-offset calibration** (the reference moves), and
- **±1 mm first-layer Z each print** (TAP runs on an inconsistently-seated PH1).
Consistent seating is therefore a candidate root cause for all three symptoms,
not just grab reliability. Worth testing PH1's seated-Z repeatability directly.

## Roadmap
1. **`DOCK_PROBE` (this branch)** — instrument latch-success vs *extra* seat depth
   past nominal, safely (reduced current so a wall hit slips, re-home each step,
   bounded to +1 mm). Nominal is the empirically-found working point; reliability
   headroom is DEEPER, so we sweep nominal → +1 mm and find the deepest seat that
   latches cleanly. Run `DOCK_PROBE PH=1..5`.
2. **Tune two numbers** from the probe + a reduced-current test:
   - reduced X/Y current for the seat push (normal is 2.2 A) that still seats the
     pin but *slips* at the wall instead of skipping steps;
   - retry depth-step + max.
3. **`_SMART_SELECT` / `_SMART_DOCK`** — wrap the existing motions with:
   precheck (dock has head / carriage empty) → seat at reduced current → servo →
   pull-out verify → targeted retry (re-servo first, then deeper, bounded) →
   pause + `_TC_LOG` on give-up. Anti-ram via reduced current and/or StallGuard
   (TMC5160 X/Y, spreadCycle) as a hard-stall tripwire — StallGuard is reliable
   for the *hard wall hit*, unlike gentle seating.

## Servo feedback
The `[servo toolchanger]` is a standard open-loop PWM servo — no position/load
readout. We monitor its *outcome* (pull-out test). Real servo feedback would be a
hardware mod (cam microswitch at full rotation, pot tap, or servo-power current
sense). `DOCK_PROBE` can be extended to sweep servo dwell/double-tap if the probe
shows the servo (not depth) is the flaky part.
