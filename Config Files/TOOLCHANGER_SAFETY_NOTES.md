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

## 2. Resume state can disagree with the physical carriage (FIXED)

Originally `PAUSE`/`RESUME` keyed entirely off `RESUME.extruder_restore_tool`
(the active *extruder*, which always names some tool whether or not a head is
physically mounted) and never consulted `carriage_tool_sensor`, so the software
belief and the physical carriage could diverge across a pause/resume.

**Fixed.** `_PRE_SELECT_CHECK` now records `pending_tool` (the target of an
in-progress change), and `RESUME` reconciles against the physical sensors before
continuing. It acts only when BOTH the carriage switch and the target's dock
sensor AGREE: a head confirmed on the carriage resumes; a confirmed-empty carriage
re-grabs the pending tool and verifies it; any sensor disagreement REFUSES and
stays paused for the operator. So it never resumes into an empty carriage
(headless) and never re-grabs a head that is already held (which would drop it).

## 3. Redundant PH1 dock+re-grab on PH1-first prints (FIXED, slicer-side)

The start G-code primed every used extruder via `_PRIME_PHx` (grab, prime, dock)
and THEN selected the initial tool via `SELECT_PHx` (which itself grabs and
primes), so the initial tool was primed twice and picked up an extra time — when
the initial tool is PH1 it was grabbed three times (TAP, prime, select).

**Fixed** in the Orca profile (`Orca-Slicer-Profiles`, ProForge 5
`machine_start_gcode`): each `_PRIME_PHx` is guarded with `initial_tool != n`, so
the initial tool is primed exactly once — by `SELECT_PHx` — and not redundantly
picked up beforehand. PH1-first drops from 3 pickups to 2.

Not fully eliminated: TAP still grabs PH1, and PH1 must dock to heat safely (ooze
in the dock zone), so a PH1-first print still picks PH1 up twice (TAP + select).
Getting to a single pickup means keeping the head held through heating over the
ooze bucket — geometry/ooze-sensitive, left as a follow-up.
