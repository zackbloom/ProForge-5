# ProForge 5 — Orca "Machine start G-code" (streamlined)

Status: SAFE removals only. The big win (eliminating the prime->re-select double
pickup of every head) is deferred to a tested `PRINT_START_PREP` macro, because it
shuffles heads right next to the forks/other docks and must be verified on the
machine, not blind-committed into a slicer file.

Geometry rule to respect: you may only sit at a nozzle's Y with X < 0, or the head
runs into the forks / other heads. Any near-dock motion must be pure-X off, then Y.

## Safe change already applied here
- Removed the standalone `DOCK` right after `G28` — `BED_MESH_SCAN` already docks
  internally (`_CHOME` -> `DOCK` -> `Z_TILT_ADJUST` -> `BED_MESH_CALIBRATE`), so it
  was redundant.

## Still redundant (leave for now; the macro will fix, tested):
- `_CAL_SELECT_T0` grabs+wipes PH1 for TAP, then `DOCK`, then `_PRIME_PHx` grabs the
  same head again to prime, then `SELECT_PHx` grabs it a THIRD context to print.
  Each used head is picked up twice. Eliminating that = prime while held + keep the
  first print tool held (no re-select) + geometry-safe pathing => the PREP macro.

---
## Single-color (PH1)
```gcode
M140 S{bed}
M104 S150 T0                 ; PH1 probe temp
G28
BED_MESH_SCAN                ; docks + meshes (redundant standalone DOCK removed)
_CAL_SELECT_T0               ; grab + WIPE PH1 (clean tip = reliable TAP)
TAP
DOCK                         ; PH1 heats on its pins (ooze == dock zone)
M104 S0 T0
M104 T0 S{ph1_print}
TEMPERATURE_WAIT SENSOR=extruder MINIMUM={ph1_print}
_PRIME_PH1
SET_FAN_SPEED FAN=filter_fan SPEED=1
SELECT_PH1
```

## Multi-color (e.g. PH1 + PH2, first tool PH2)
```gcode
M140 S{bed}
M104 S150 T0                 ; PH1 probe temp
M104 T1 S{ph2_print}         ; start PH2 warming early
G28
BED_MESH_SCAN                ; docks + meshes (redundant standalone DOCK removed)
_CAL_SELECT_T0               ; TAP always references PH1
TAP
DOCK
M104 S0 T0
M104 T0 S{ph1_print}
M104 T1 S{ph2_print}
TEMPERATURE_WAIT SENSOR=extruder  MINIMUM={ph1_print}
TEMPERATURE_WAIT SENSOR=extruder1 MINIMUM={ph2_print}
_PRIME_PH1
_PRIME_PH2
SET_FAN_SPEED FAN=filter_fan SPEED=1
SELECT_PH2                   ; first print tool
```

## Target (after PRINT_START_PREP is built + tested)
```gcode
M140 S{bed}
G28
PRINT_START_PREP TOOLS={used_tools} TEMPS={print_temps} INITIAL={initial_tool}
```
One call, single or multi. Handles heat + TAP(PH1) + prime + place with each head
grabbed at most once, pure-X off the forks.
