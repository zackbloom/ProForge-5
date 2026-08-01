# Power-Loss Recovery (ProForge 5, experimental)

Adds opt-in "resume after power loss" by **wrapping** the proven
[BigTreeTech KlipperPLR](https://github.com/bigtreetech/KlipperPLR) engine and
patching in the ProForge's toolchanger head re-pick. Nothing here is enabled by
default.

## Why wrap KlipperPLR instead of rolling our own
KlipperPLR's shell script rebuilds a **fresh resume g-code file** from the
interruption point: it re-emits temperatures, restores extrusion mode
(`M83`/`G92 E0`), sets the trusted Z, lifts, homes XY, drops, and continues. That
removes the two failure modes an in-house macro kept tripping on — relative-E
mismatch on resume, and writing/corrupting state in the offsets file. We only add
the toolchanger bits.

## Dependency
Requires `fix/toolchanger-safety` merged (guarded `SELECT_PHx`). Recovery mounts a
head via `SELECT`, and without the collision guard a bad state can crash heads.

## Install
1. **KlipperPLR engine**
   ```bash
   git clone https://github.com/bigtreetech/KlipperPLR
   cd KlipperPLR && ./install.sh
   ```
2. **De-dupe the generated `plr.cfg`** so it doesn't clash with the ProForge
   config. Delete these blocks from the KlipperPLR `plr.cfg`:
   - the second `[save_variables]`  (printer.cfg already declares one)
   - `[respond]`                    (already provided)
   - `[delayed_gcode KINEMATIC_POSITION]` that zeroes X/Y/Z at boot — it fights
     the ProForge homing / `safe_z_home` flow.
3. **Patch `plr.sh`** to call our re-pick (see `plr-repick.patch`):
   ```bash
   grep -q "_PLR_REPICK_TOOL" plr.sh || \
     sed -i "/echo 'G28 X Y'/a echo '_PLR_REPICK_TOOL' >> \${PLR_PATH}/\"\${plr}\"" plr.sh
   ```
4. **Include our integration** in `printer.cfg`, after the KlipperPLR include:
   ```
   [include plr-toolchanger.cfg]
   ```

## Slicer hooks (Orca)
- **Machine start G-code** (near the top, in addition to the existing routine):
  ```
  G31
  save_last_file
  SAVE_VARIABLE VARIABLE=was_interrupted VALUE=True
  ```
- **Layer change G-code** — use our snapshot (captures Z *and* tool):
  ```
  _PLR_LOG_STATE
  ```
- **Machine end G-code**:
  ```
  SAVE_VARIABLE VARIABLE=was_interrupted VALUE=False
  clear_last_file
  G31
  ```
  Also make sure `CANCEL_PRINT` clears it (`SAVE_VARIABLE VARIABLE=was_interrupted VALUE=False`)
  so a cancel doesn't leave a stale resume offer.

## Recovering after an outage
Do **not** move the gantry. Run `RESUME_INTERRUPTED` in the Mainsail console.
It rebuilds the resume file and runs it: set Z → reheat → lift → home XY →
`_PLR_REPICK_TOOL` (mount the saved head) → drop → continue.

## Logging (for future debugging)
`_PLR_LOG_STATE` writes a `PLR_SNAPSHOT z=… tool=… layer=…` line to klippy.log
every layer, and `_PLR_REPICK_TOOL` logs the carriage/dock sensor states before
and after the re-pick. After any failed resume, those breadcrumbs in klippy.log
show exactly where it was and what the toolchanger believed.

## Residual limitations — NO software fixes these (validate/accept before trusting)
- **B3 — trusted Z, four independent leadscrews.** Recovery assumes Z held with
  power off. Your `z_tilt` has four Z motors; uneven relaxation → wrong/tilted Z
  on resume. Single-leadscrew machines (KlipperPLR's target) don't have this.
- **B4 — nozzle welded to the part.** A cold nozzle fused to the top layer can
  peel the print on the first lift. Nothing software-side prevents it.
- **B5 — filenames.** Your files contain spaces and `+`
  (`Wikinger+Doppelaxt_0.2mm_PLA_ProForge 5_…`). KlipperPLR round-trips the name
  through `save_variables` and the shell; verify a resume works with a spaced/`+`
  filename before relying on it (test plan step 2).
- **The reliable fix is still hardware:** a UPS sized for the printer, or a small
  UPS + a mains-loss GPIO that fires a fast park+save. This is best-effort software.

## Test plan (in order, emergency stop in reach)
1. Small print. Mid-print run `FIRMWARE_RESTART` (never a real outage first).
2. Confirm `was_interrupted`/`power_resume_z`/`power_resume_tool` saved, and that
   the filename (with spaces/`+`) round-tripped.
3. Run `RESUME_INTERRUPTED`. Watch the lift → XY home → **tool re-pick** → drop
   for any collision; `M112` if anything looks wrong.
4. Repeat with a multi-tool print so `_PLR_REPICK_TOOL` mounts a non-PH1 head.
5. Only after several clean restart-resumes, try a real breaker flip on a scrap
   print.
