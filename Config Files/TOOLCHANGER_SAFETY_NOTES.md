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
