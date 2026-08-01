# Toolchanger safety — analysis & follow-ups

This branch (`fix/toolchanger-safety`) addresses a head-into-head collision that
can happen when a print or calibration run resumes with a head still mounted.

## 1. Collision guard (FIXED in this branch)

**Bug.** The normal pickup path `SELECT_PHx` → `_PRE_SELECT_CHECK` reads the
physical `carriage_tool_sensor` and docks a stray head *before* moving into a
dock. The calibration pickups `_CAL_SELECT_T0..T4` (used by the print-start TAP
sequence and by `CALIBRATE_TOOL_OFFSETS`) skipped that check and drove straight
into the dock via `_SERVO_SELECT`. If a head was already on the carriage, the
carriage slammed the mounted head into the head it was reaching for.

The start sequence only removes a mounted head with a single `DOCK` near the top
(`G28` → `DOCK` → … → `_CAL_SELECT_T0`). Any resume/restart that re-enters the
setup routine *after* that `DOCK`, with a head mounted, defeats the only guard.

**Fix.** New `_CAL_ENSURE_CARRIAGE_CLEAR` macro, called at the top of every
`_CAL_SELECT_Tn`. It mirrors what `_PRE_SELECT_CHECK` already does: if
`carriage_tool_sensor` reads PRESSED, run `DOCK` first. Protects all callers
regardless of how the routine was entered.

**Hardening (red-team A1/A2).**
- *A1 — closed the bypass.* If `DOCK` cannot resolve which head to dock it
  pauses only while `print_stats.state == "printing"`; run from the console it
  would leave the head mounted and the old code plunged anyway. The guard now
  calls `_CAL_VERIFY_CLEAR` (a **separate** macro, because Klipper renders a
  macro's template before its sub-macro's moves run) which `M400`s, re-reads the
  sensor, and `action_raise_error`s if the carriage is still occupied — aborting
  the whole `_CAL_SELECT` chain so the plunge never happens.
- *A2 — fresh sensor read.* The guard now `M400`s before reading
  `carriage_tool_sensor`, so a value that is stale behind queued motion can't slip
  through.
- *A3 — residual.* Still only as good as the sensor: a false-negative
  `carriage_tool_sensor` (reads RELEASED with a head on) defeats it. Same
  dependency as the stock `SELECT` path — not a regression.

## 1b. Dock reseat + diagnostics (NEW, `toolchanger-extras.cfg`)

`RESEAT_DOCKS` addresses the common "multiple heads read undocked" `DOCK` error,
which usually just means a head isn't pushed fully into its dock. With the
carriage empty and XY homed, it gently pushes each undocked head to its seated
dock X (never past it, so it can't jam) and verifies the switch. The `DOCK`
multiple-undocked error message now points at `RESEAT_DOCKS`.

`_TC_LOG` writes a one-line snapshot (carriage sensor, all dock sensors, active
extruder, homed axes, `can_continue`, `already_selected`) to klippy.log. It is
wired into `DOCK` entry and the multiple-undocked branch, and into `RESEAT_DOCKS`
and the PLR re-pick, so a future failure can be reconstructed from the log.

Requires `[include toolchanger-extras.cfg]` in `printer.cfg` (added).

## 2. Resume state can disagree with the physical carriage (FOLLOW-UP, not fixed)

`PAUSE` sets `RESUME.extruder_restore_tool` from `printer.toolhead.extruder`
(the active *extruder*), which always names some tool whether or not a head is
physically mounted. `RESUME` then keys entirely off that variable
(`{% if extruder_restore_tool >= 0 %}`) and never consults `carriage_tool_sensor`.
So the software belief and the physical carriage can diverge across a
pause/resume, which is how a head ends up mounted while the routine "forgets" it.

Proposed follow-up (separate commit): have `RESUME` reconcile against the
physical sensor — if `carriage_tool_sensor` is PRESSED, `DOCK`/verify before
resuming; if it is RELEASED, don't try to wipe/select a tool that isn't there.
The guard in (1) already prevents the *collision*; this would fix the root
state-tracking mismatch.

## 3. Redundant PH1 dock+re-grab on PH1-first prints (FOLLOW-UP, needs testing)

In the start G-code (Orca profile `machine_start_gcode`), TAP always runs on PH1
via `_CAL_SELECT_T0` → `TAP` → `DOCK`. Then each used extruder is primed
(`_PRIME_PHx`, which picks up, primes, and docks), then the initial tool is
selected (`SELECT_PHx`). When the first print tool **is** PH1, PH1 gets grabbed
three times (TAP, prime, select), each with a dock + ~20 cm +X clearance move +
re-grab — the "releases the tool, moves ~20 cm X+, immediately grabs it again"
behavior.

Why it is not a trivial skip: simply removing the post-TAP `DOCK` would leave PH1
mounted, and `SELECT_PH1`/`_PRE_SELECT_CHECK` would then take the
"already selected → skip" branch — which also skips the prime, `_OFFSET_RESET`,
and the `extruder_restore_tool` bookkeeping that `_SELECT_PHx` performs. So this
needs a small refactor (e.g. an "already-mounted" path in `_SELECT_PHx` that
still runs prime/offset/bookkeeping), and must be validated on-machine. Tracked
here rather than shipped half-done.
