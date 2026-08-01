# Power-Loss Recovery (experimental)

Adds an opt-in "resume after power loss" flow for the ProForge 5, integrated
with the toolchanger so recovery re-picks the correct head.

**This is untested scaffolding.** It is not wired into `printer.cfg`. Treat it as
a starting point to bench-test, not a finished feature.

## Why this exists
Klipper has no native power-loss recovery: on an outage klippy restarts with no
position, homing, or tool state, so the "resume" option only appears
inconsistently (basically never after a real outage). This makes it consistent by
continuously saving print state to disk during the print and providing a
`RESUME_INTERRUPTED` macro.

## Dependency — land the safety fix first
Requires `fix/toolchanger-safety`. Recovery re-picks a head via `SELECT_PHx`, and
if the collision guard is missing a bad state can drive one head into another.
Do not enable PLR without that branch merged.

## Enable (once tested)
1. Merge `fix/toolchanger-safety`.
2. `[include power-loss-recovery.cfg]` in `printer.cfg`.
3. Add `_PLR_ON_LAYER` to Orca's **Layer change G-code** (snapshots each layer).
4. Add `_PLR_CLEAR` to the end-print and cancel paths (so a clean finish doesn't
   leave a stale resume offer).

## How it works
- `_PLR_ON_LAYER` (per layer) saves file, `virtual_sdcard.file_position`, Z,
  layer, active tool, and bed/tool target temps into the existing
  `[save_variables]` file.
- On startup, `_PLR_ON_READY` checks `plr_state` and, if a print was live, tells
  the user a resume is available.
- `RESUME_INTERRUPTED`: reheats, trusts saved Z (`SET_KINEMATIC_POSITION`), blind
  Z-lift to clear the layer, homes XY, re-picks the saved head (guarded),
  restores Z, primes, then continues the SD file from the saved byte offset
  (`M23`/`M26`/`M24`).

## Known-fragile points to validate on the bench
1. **Z is trusted, never re-probed.** Depends on the four Z leadscrews holding
   position unpowered. Do not touch the gantry after an outage. Re-probing is not
   an option — the Eddy would have to descend into the printed part.
2. **Nozzle-weld.** A cold nozzle fused to the top layer can peel the part on the
   first lift. No software fixes this.
3. **Byte-seek resume (`M26`).** Assumes the saved position is a clean layer
   boundary (it is, saved at layer change) and that modal state (G90/M83/fan/
   accel) matches what `RESUME_INTERRUPTED` sets. The community plugins instead
   *rebuild a resume g-code file* with a fresh header — more robust; worth
   evaluating (see below).
4. **Frequent `SAVE_VARIABLE` writes** rewrite the whole variables file each
   layer. Fine for occasional/slow prints; consider a dedicated save file or a
   time-throttle for very fast/thin-layer prints.

## Test plan (do this, in order, before trusting it)
1. Small test print. Mid-print, run `FIRMWARE_RESTART` (NOT a real power cut).
2. Confirm `_PLR_ON_READY` reports the correct tool/Z/layer.
3. With the **emergency stop in reach**, run `RESUME_INTERRUPTED`. Watch the
   Z-lift, XY home, and tool re-pick for any collision; `M112` if anything looks
   wrong.
4. Only once several restart-resumes succeed cleanly, try a real breaker-flip
   test on a throwaway print.

## Alternatives worth considering instead of this hand-rolled version
- **[BTT KlipperPLR](https://github.com/bigtreetech/KlipperPLR)** or
  **[ankurv2k6/klipper-plr](https://github.com/ankurv2k6/klipper-plr)** as the
  state-saving/resume engine, with the ProForge tool re-pick grafted into their
  `RESUME_INTERRUPTED`. These are more battle-tested than this file; this cfg can
  serve as the toolchanger-integration reference.
- **The reliable fix is hardware:** a UPS sized for the whole printer, or a small
  UPS + a mains-loss GPIO that fires a fast "lift, park, heaters off, save state"
  before the UPS dies. Software PLR is best-effort; hardware prevents the loss.
