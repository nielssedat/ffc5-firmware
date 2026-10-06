# Creator 5 N4S4: statistics, per-pickup retract, prime-tower-aware tool changes, plate and material Z, purge work, and the shared extruder stepper

## Summary

This builds on `master` at `df57d27`. It is the part of the work in #29 that
`master` does not have, rebuilt on top of what happened to it since: two
commits that took pieces of #29 (`bb84772`, `308d97e`), two that removed some
of those again (`7e8ea9d`, `df57d27`), and two fixes by another contributor
(`01892ff`, `3d9f193`). The original's bed-mesh probing travel, adaptive-mesh
macros, deferred start-up, prime-tower geometry and Z-hop / in-dock retract
options are used as they are and are not touched here. `UPSTREAM_STATUS_N4S4.md`
lists, item by item, what is in `master`, what is added to it, and what was
removed or declined there and is kept.

The additions to the tool change are **opt-in**: without the N4S4
configuration a printer behaves as `master` does (`ff_print` only reads one
more thing out of the file, the tools it uses, and reports it in its status). Within
`ff_toolchange` every addition defaults to the original's behaviour, and every
test function of the original passes **unmodified**. Of the six macros the
configuration redefines, two (`ADAPTIVE_MESH`, `DEFINE_PRIME_TOWER_OBJECT`) are
the original's plus exactly the documented additions, and a test fails when
the original changes either of them; the other four replace the original's
text, and a test pins what that text looks like.

## Problems addressed

- Klipper's `print_stats` reports only about a quarter of the filament a
  multi-colour job uses, and nothing records tool changes, their duration and
  failures, or where the time of a job goes.
- After a tool change the new tool could return to the position at which the
  previous object was last printed, lower the nozzle over an existing part and
  leave a blob before travelling to the prime tower. (The original's `restore_*`
  options protect the trip, but not this.)
- Every pickup in a prime-tower job received an additional firmware retract
  even when Orca had already parked that tool with a 2 mm unload retract. With
  a small Prime Volume this stacked pressure deficit left holes at the aligned
  wall seam after a tool change.
- A pickup that is followed by a purge line needs a stronger retract than one
  that is not.
- A job without a prime tower had no safe place for a hot tool to wait and to
  build pressure.
- Build plates and filament materials could not have independent additive
  first-layer Z corrections.
- Residual filament could remain attached to a nozzle after a rear-chute purge
  and be dragged from the purge area toward the print.
- Pre-print purge behaviour, timelapse capture and the statistics were not
  conveniently available from Mainsail.
- The Creator 5 has four hotends and **one** filament stepper, but Klipper
  creates four stepper objects for it, and the MCU toggles the direction pin
  per object. The pin level then depends on the history of all four, and
  retract and extrude can swap when an idle tool was left extruded. The
  original removed the single-stepper adapter for lack of a demonstrated
  failure; the cause is in the MCU code (read in Klipper's and in the stock
  eBoard image) and unchanged, and the failure has now been shown on a
  Creator 5 (see below).

## Main changes

### Statistics (`ff_stats`)

- Lifetime and per-job statistics: tool changes (swaps and first pickups,
  duration, failed changes by stage, failed grab/release attempts), print
  time, filament measured per tool, and the time of each job divided into
  phases (prepare, homing, heating, mesh, purge, toolchange, print, paused,
  end).
- Filament is measured from each extruder's own running position.
  `correct_print_stats` (default on) replaces `print_stats.filament_used` with
  the measurement while a tracked job prints, which corrects Mainsail's
  dashboard and Moonraker's job history for new jobs.
- Phases come from Klipper's homing events, the tool-change code and a
  configurable list of commands (`phase_commands`) that `ff_stats` wraps. No
  macro is edited.
- Mainsail macros `FF_STATS`, `FF_STATS_JOB`, `FF_STATS_JOBS`; the numbers are
  also `printer.ff_stats`. `FF_STATS_RESET CONFIRM=1` starts over and keeps the
  old file.
- One JSON file under `/usr/data/anvil-data`, written atomically and rarely
  (the data partition is mounted `sync`); a job left open by a power loss is
  recovered as `interrupted`.
- The hooks in `ff_print` and `ff_toolchange` are optional and exception-safe:
  a fault in the statistics cannot fail a print or a tool change.
- Documented in `docs/statistics.md`. On a Creator 5, two four-colour jobs:
  the measured filament per tool, minus the start-up clean, is within 0.6 % of
  the slicer's per-tool lengths and 0.3 % above its total.

### Tool change: what to do over the model, and how hard to retract

- `TOOLCHANGE_SET_PRIME_TOWER` registers the active tower (called from
  `DEFINE_PRIME_TOWER_OBJECT`). A change issued **inside** it returns there as
  `restore_axis` does; one issued over the model skips the X/Y return and the
  descent, and the tool stays raised and retracted for the slicer's travel.
- Pickups choose their in-dock retract: `purge_retract` plus
  `purge_retract_dwell_ms` for a pickup armed with `TOOLCHANGE_PREPARE_PICKUP`
  or a tool's first pickup in a registered tower job; `tower_repeat_retract`
  (0 by default) for a repeat pickup, so it does not stack on Orca's unload
  retract; `restore_retract` otherwise.
- `TOOLCHANGE_BEGIN_JOB` resets the per-job record of the tools used.
- `no_tower_prime_macro` turns on the no-tower workflow: no return to the
  model, and one chute prime on a tool's first use (`_NS_TOOLCHANGE_PRIME`).
- `protect_every_change` extends the original's rule that only a change
  restoring X or Y is raised and retracted: the pickups of the cleaning macros
  restore nothing and need the same protection. Off unless set.
- `restore_via_corridor` makes the return after a change go along the safe
  column first when the target lies beyond `x_safe` (the purge chute, the wipe
  pad). One straight move from the dock to the chute cuts across the docked
  tools with the new tool on the carriage: `LOAD_FILAMENT` on T1 drove into T2
  that way, because the original's load macros pick up and park without
  `RESTORE_AXIS=` and `restore_axis` is set. Off unless set.
- `FF_TOOLCHANGE_STATUS`, and the plain `TOOLCHANGE_STATUS` unless a macro of
  that name is configured (a Mainsail button).
- `purge_retract`, `purge_retract_dwell_ms`, `tower_repeat_retract`,
  `no_tower_prime_macro`, `protect_every_change` and `restore_via_corridor`
  are the only new `[ff_toolchange]` options.

### Build-plate and filament Z

- The print-scoped Z correction is three independent components: the app's
  temperature / bed / layer term, a build-plate term and the active filament's.
  `TOOLCHANGE_SET_PRINT_OFFSET PLATE=` and `TOOLCHANGE_SET_MATERIAL_OFFSET`
  replace their component absolutely, so repeated calls cannot accumulate, and
  neither touches babystepping or tool calibration. `CLEAR=1` clears all three.
- `_BUILD_PLATE_OFFSETS` and the Orca mapping are in `printer_n4s4.cfg` and
  `ORCA_MACHINE_AND_FILAMENT_SETTINGS.md`. The values are `0.000` until
  calibrated (the shipped configuration sets Textured PEI to `0.03`).

### The used tools

- `ff_print` reads the tools a file uses from Orca's `; filament:` header (a
  scan of `Tn` that gives up after 2 s otherwise): `printer.ff_print.tools`.
  It is not passed on to the original's start macro, which would clean every
  one of them; `_NS_BEFORE_PRINT` reads it. It feeds the checks of every print
  (`_FF_PREFLIGHT`, `_NS_FILAMENT_PREFLIGHT`) and, in purge mode `ALL`, the
  clean of every colour.
- The preheat of the file's second tool after the mesh, which the original
  dropped on purpose, is not part of this change either.

### The shared extruder stepper (`ff_extruder`)

- `[extruder]` is the only owner of the physical stepper; `extruder1` to
  `extruder3` stay logical hotends with their own heaters and motion queues,
  and `SYNC_EXTRUDER_MOTION` connects the one stepper to the picked tool
  before the in-dock retract. One object means one direction history.
- `SET_PRESSURE_ADVANCE` works for an active logical extruder, and
  `post_m109_macro` runs a macro after the native `M109`.
- `gcode/n4s4-extruder-direction-test.gcode` shows the difference on the
  printer: it breaks the "retract before the change" rule on purpose. Run on a
  Creator 5 (2026-10-06): T0 is left extruded, then T1 purges. With
  `ff_extruder` T1 purged normally; on the stock design (without it) T1's
  purge ran the wrong way. A retract on T1 before the purge does not help (its
  pickup already retracts 0.4 mm in the dock); the old tool's last move has to
  be a retract, or there has to be one stepper object.

### Pre-print purge, cleaning and ooze

- `START_PURGE_SET` / `START_PURGE_STATUS`: `OFF`, `FIRST`, `ALL`, persistent.
- The wipe across the front silicone lip, the cooldown pad spread over a
  30-point grid, the next tool preheated while the current one is cleaned, a
  safe corridor for pickup and release, a 150 C front wipe, no unrecovered
  5 mm retract, filament preflight for every used tool.
- `_PURGE_NEAR_OBJECT`: a purge line next to the objects, by nozzle diameter.

### Timelapse, installation, HelixScreen

- `TIMELAPSE_TOGGLE` (persistent), the modified `timelapse.cfg` shipped by the
  `anvil-timelapse` package.
- `10-enable-printer-n4s4.sh` adds `[include printer_n4s4.cfg]` to
  `printer.cfg` safely and idempotently; `anvil-link-prog.sh` installs it.
- Two printer pictures for HelixScreen.

## Not included

- Nozzle hours at temperature and heat cycles, and maintenance counters (axis
  travel, motor, bed and fan hours): possible follow-ups of the statistics.

## Files changed

See the table in `UPSTREAM_STATUS_N4S4.md` (what and why) and
`CHANGELOG_N4S4.md` (per file). In short:

- `pkgs/klipper/payload/klipper/klippy/extras/`: `ff_stats.py` and
  `ff_extruder.py` (new), `ff_toolchange.py` and `ff_print.py` (extended).
- `pkgs/klipper-config/payload/config/printer_n4s4.cfg` and
  `pkgs/klipper-config/payload/scripts/10-enable-printer-n4s4.sh` (new);
  `pkgs/klipper-config/build.sh`, `pkgs/anvil-core/payload/bin/anvil-link-prog.sh`
  (the installer).
- `pkgs/timelapse/` (`payload/config/timelapse.cfg`, `build.sh`, `pkg.conf`),
  the HelixScreen pictures.
- `qa/static/`: `test_toolchange_n4s4.py`, `test_n4s4_overrides.py`,
  `test_ff_print_tools.py`, `test_ff_extruder.py`, `test_ff_stats.py`,
  `test_ff_toolchange.py`, `test_n4s4_include_installer.py`,
  `test_timelapse_config.py`, `test_klipper_config.py` (additions);
  `qa/replica/`: `test_ff_stats.py`, `test_custom_scripts.py` (additions).
- Documents: `docs/statistics.md`, `docs/ssh-keys.md`, `UPSTREAM_STATUS_N4S4.md`,
  `FEATURES_N4S4.md`, `CHANGELOG_N4S4.md`, `ORCA_MACHINE_AND_FILAMENT_SETTINGS.md`,
  this file; `gcode/n4s4-extruder-direction-test.gcode`.

`printer_n4s4.cfg` is shipped by the `anvil-klipper-config` package and
installed as `/usr/data/anvil-data/config/printer_n4s4.cfg`. It contains the
`[ff_extruder]` activation together with the N4S4 integration macros and the
calibration values, and is included from `printer.cfg` immediately before the
`SAVE_CONFIG` block:

```cfg
[include printer_n4s4.cfg]
```

The boot hook `/usr/data/anvil-data/scripts/10-enable-printer-n4s4.sh` does
that on its own: it verifies that both files exist, inserts the include, and
creates a backup only when it changes the file. Once the exact include exists,
repeated boots exit immediately without adding script output to the log.

## Validation

- The static suite in the build image: 586 passed and 2 skipped, among them
  every test function of the original, unmodified; it also passes at each of
  the nine commits.
- The replica lane: the Creator 5 package built from this branch, installed on
  a Creator 5 replica, 167 passed.
- The tool-change module was compared with the previous branch's on about
  134,000 combinations of restore axis, hop, retract values, tower position,
  mounted and requested tool, nozzle temperature and shared stepper; with the
  N4S4 configuration the command streams are identical (the only difference
  is the intended one named in `FEATURES_N4S4.md`).
- `ADAPTIVE_MESH` and `DEFINE_PRIME_TOWER_OBJECT` are rendered next to the
  original's by `test_n4s4_overrides.py`; changing either original makes the
  test fail. The original's text of the four other redefined macros is pinned
  the same way.
- On a Creator 5 (code of the earlier branch): tool pickup, in-dock retract,
  Z-hop, prime-tower travel, material and plate Z composition, adaptive mesh
  selection, timelapse suppression, the manual and automatic purge paths, and
  the statistics on two four-colour jobs.

## Compatibility and fallback behaviour

- No registered prime tower and no `no_tower_prime_macro`: the change is the
  original's.
- A `restore_axis` without X or Y: no hop, no in-dock retract (the original's
  reading), unless `protect_every_change` is set (it is in
  `printer_n4s4.cfg`).
- In the code, build-plate and material offsets are zero until set; the
  original print Z compensation stays active independently of them.
- Adaptive mesh, timelapse and purge modes can be selected without changing
  Orca's generated layer-by-layer commands.
- Without `[ff_stats]` nothing is recorded; without `[ff_extruder]` every
  `[extruderN]` owns its stepper and nothing is synced.
