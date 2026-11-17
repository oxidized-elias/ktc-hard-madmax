# MadMax support for klipper-toolchanger-hard

This is an **experimental, opt-in front/side-keyslot XY docking profile**. It adapts
[MadMax's documented geometry](https://github.com/zruncho3d/madmax/blob/main/CONFIGURATION.md)
and [r2pdx's Micron sequence](https://github.com/zruncho3d/madmax/blob/main/Configs/r2pdx_config-micron-2025-12-21/klipper-toolchanger/toolchanger.cfg)
to this repository's native detection and toolchange APIs. The MadMax source used
for the adaptation was commit `277a63cbb696f937fbf9d883a8d8c46a658cd6d9`.

Software checks cover macro rendering and installation. This profile has **not
been validated on a physical printer**. Installing it does not establish correct
wiring, homing, clearance or dock alignment.

## Compatibility and scope

The supported arrangement has front docks, increasing Y away from the docks, and
sideways keyslot engagement in X. The carriage and dock must stay aligned in Z:
use a fixed gantry or docks that move with the gantry. Frame-mounted docks on a
moving gantry need a separate Z-alignment design and are outside this profile.
Rear or rotated docks need different motion paths.

MadMax's coupling contacts provide tool presence through this fork's native
`detection_pin` on every tool. Bed probing can use either:

- Per-tool coupling probes, configured through `tool_probe` and `tool_probe_endstop`.
- A separate bed probe, with the coupling contacts still used for native tool detection.

This profile does not require the older probe-based detection wrappers or their
`INITIALIZE_TOOLCHANGER` / `VERIFY_TOOL_DETECTED` macro replacements. It uses
native verification and `abort_on_tool_missing`. It does not automatically install
accelerometer drop detection, liftbars or third-party offset calibration macros.

## Installation

This KTC-Hard implementation lives on the `ktc-hard-madmax` branch of
[oxidized-elias/ktc-hard-madmax](https://github.com/oxidized-elias/ktc-hard-madmax/tree/ktc-hard-madmax).
Use the explicit branch name below even if GitHub opens another default branch.
Clone this branch explicitly into its own directory:

```sh
cd ~
git clone --branch ktc-hard-madmax --single-branch https://github.com/oxidized-elias/ktc-hard-madmax.git ktc-hard-madmax
cd ~/ktc-hard-madmax
./install.sh --madmax
```

Without arguments, a terminal installation offers **1. Extensions only** and
**2. MadMax**. Piped/noninteractive installation keeps the original extensions-only
behavior unless `--madmax` is explicitly supplied. `--extensions-only` explicitly
chooses the original behavior, and `--help` describes the options.

The script uses its own checkout for local execution, including renamed clones.
Always run this branch's local script: the streamed upstream installer and the
secondary-only installer retain upstream download behavior and do not select this
branch. For updates, use the [Moonraker entry in the root README](../../README.md#moonraker-updates)
with `primary_branch: ktc-hard-madmax` and your actual clone directory.

The MadMax option symlinks the normal Python extensions and creates these files
under `~/printer_data/config/toolchanger/`:

| File | Purpose |
| --- | --- |
| `madmax.cfg` | Symlink to the shared motion profile in this checkout |
| `madmax-include.cfg` | Symlink to the wrapper loading the profile then editable settings |
| `madmax-user.cfg` | Editable calibration copied once and preserved on reruns |

It does not edit `printer.cfg`, fabricate toolboard definitions or install a
homing override. It does not restart Klipper for MadMax: review your configuration
first, then restart manually.

Replace your existing dock-motion include with:

```ini
# Keep your real MCU, tool, fan, extruder and probe definitions.
[include toolchanger/madmax-include.cfg]
```

Keep this wrapper after other shared toolchanger configuration so its motion and
monitoring settings take precedence. Do not include the conventional fixed-dock,
liftbar or flying-gantry path alongside it. Keep only one copy of any macro or
hardware section. Do not directly edit the two symlinked files; put measurements
in `madmax-user.cfg`.

`--extensions-only` updates Python links and restarts Klipper as before. It does
not remove or disable an already included MadMax profile. To switch mechanisms,
change the motion include deliberately and review the replacement configuration.

## Native detection and optional per-tool probing

Configure a unique tool number, real extruder/fan references and coupling input
for each tool. The following is an illustration to merge with existing hardware
sections, not a board configuration:

```ini
[tool T0]
tool_number: 0
extruder: extruder
fan: T0_partfan
# Replace the MCU/pin and verify electrical polarity on your hardware.
detection_pin: ^!toolboard0:COUPLING_PIN
```

The logical native detection state must be PRESENT only when the head is seated.
Check the `detected_tool` / `detected_tool_number` status and use
`INITIALIZE_TOOLCHANGER` after confirming the physical state. Exactly one attached
head should be detected. A detached or lifted head must stop reporting present.
All configured tools must have native detection pins; the core rejects mixed
setups where only some tools have detection.

If the same coupling contact is also the bed probe, configure the matching
`[tool_probe T0]` with `tool: 0`, the actual probe pin and calibrated Z offset.
Its probe state must be untriggered while seated and triggered when lifted or
detached. This fork supports sharing the physical input between native detection
and probing; detection and probe polarity can differ. Its supplied
[per-tool example](../z%20probe/per%20tool%20probe/toolhead_n.cfg) illustrates that
pattern. Confirm the states on the actual hardware before moving.

The profile selects `SET_ACTIVE_TOOL_PROBE T=...` after initialization and pickup
when `tool_probe_endstop` exists. In that arrangement every selectable tool must
have its corresponding probe. If using a separate bed probe, omit
`tool_probe_endstop` and per-tool probe sections; native detection remains required.
Current Kalico can also keep named coupling probes alongside a separate default
bed probe using `register_as_probe: False` on the tool-probe router. See
[Kalico compatibility](../../docs/kalico-compatibility.md) for that arrangement
and the verified native probing features.

Do not import Easy Install's renamed detection command wrappers. Do not enable
legacy `START_TOOL_PROBE_CRASH_DETECTION` for this profile: normal docking releases
the coupling, and this fork manages missing-tool monitoring around the change.
Use the native `abort_on_tool_missing` behavior described below.

## Homing and missing-tool monitoring

The profile initializes manually and refuses toolchanges on unhomed axes. Keep a
known head attached, verify detection, then call `INITIALIZE_TOOLCHANGER` from
your reviewed homing sequence before relying on tool selection. Initialization
and homing are separate; the profile does not supply a universal G28 override.

For first commissioning, use T0 for bed homing and probing. If using the coupling
probe as the Z endstop, configure `probe:z_virtual_endstop` as described in
[the tool probe reference](../../docs/tool_probe.md). Set a real Z offset rather
than copying the example. Keep unrelated tools, docks and umbilicals clear of
homing moves, leveling points, mesh bounds, purge lines and print-end moves.

Native `abort_on_tool_missing: True` is enabled. The core deactivates its monitor
for intentional docking, verifies the picked-up tool, then reactivates monitoring.
`ignore_detect_probing_events: True` prevents ordinary probe contact from being
processed as a tool removal. Review the core's `tool_missing_delay` for your
printer; a filtered detector is not an instantaneous electrical emergency stop.
With virtual-SD printing the existing helper suppresses missing-tool aborts when
no print is active, so verify operating behavior during commissioning rather than
assuming an idle bench test exercises print-time monitoring.

The supplied `error_gcode` turns heaters off and calls `PAUSE`. Your printer must
have a working `[pause_resume]` setup. Review its pause motion for dock clearance;
customize `error_gcode` in your user config if your machine needs another response.
No automatic recovery motion is supplied. Support a loose head and inspect the
cause before reinitializing.

## Measure and enter docking coordinates

Follow [MadMax's physical calibration procedure](https://github.com/zruncho3d/madmax/blob/main/CONFIGURATION.md#calibrate-docking-positions).
Measure in docking coordinates using `ENTER_DOCKING_MODE` and
`EXIT_DOCKING_MODE` on an initialized toolchanger. Docking mode removes tool and
G-code offsets; normal print coordinates may differ once tools are calibrated.
These commands do not establish homing or physical clearance.

| Parameter | Measurement |
| --- | --- |
| `params_park_x` | Absolute X with the head retained in the narrow keyslots |
| `params_detach_x` | Absolute X with its screws aligned to the large dock openings |
| `params_attach_y` | Absolute Y at the opening where sideways engagement is possible |
| `params_close_y` | Absolute Y outside magnetic capture, before slow approach |
| `params_safe_shuttle_y` | Absolute Y allowing the empty carriage to cross all docks |
| `params_safe_toolhead_y` | Absolute Y allowing an attached head to cross all docks |
| `params_path_speed` | Engagement/separation speed in mm/min |
| `params_fast_speed` | Clearance travel speed in mm/min |
| `params_madmax_calibrated` | Explicit permission to use a tool's measured coordinates |

Speeds of 600 and 6000 mean 10 and 100 mm/s, respectively. Start conservatively.
These are initial software defaults, not hardware-tested recommendations.

The Y ordering must be:

```text
front / entrance                                    back
attach_y < close_y <= safe_shuttle_y <= safe_toolhead_y
```

For a left dock, `detach_x` is commonly greater than `park_x`; for a right dock it
is commonly smaller. Either direction is supported. Measure actual keyslot and
head geometry rather than applying a guessed fixed X distance.

The installed calibration file starts with `None` Y coordinates. Add real tool
sections and their measurements; the commented examples intentionally do not
create incomplete tools on single-tool printers:

```ini
[toolchanger]
# Replace None with measured numeric coordinates.
params_safe_toolhead_y: None
params_safe_shuttle_y: None
params_close_y: None
params_attach_y: None

[tool T0]
params_park_x: None
params_detach_x: None
params_madmax_calibrated: False

[tool T1]
params_park_x: None
params_detach_x: None
params_madmax_calibrated: False
```

Use only sections matching your actual tools. After measuring all coordinates for
one head and reviewing the swept envelope, set its calibration flag to `True`
and restart Klipper. Calibrate other heads separately. Per-tool Y and speed values
can override shared values; every clearance still needs to clear all parked heads.

The guard checks both participating tools before normal release. It rejects unset
or out-of-axis-range coordinates, equal park/detach X, nonpositive speeds and the
wrong Y order. It cannot determine physical clearance, dock height, magnetic
strength or seating quality. It also cannot protect unrelated manual or homing
moves.

Remove per-tool motion overrides inherited from another mechanism. In particular,
`before_change_gcode`, `pickup_gcode`, `dropoff_gcode` and `recover_gcode` must not
silently retain lift-out behavior. Keep `perform_restore_move: False` for this
profile; its pickup path explicitly restores requested axes after clearing docks.
A second core restore move could bypass that intended motion order.

## Sequence and restoration

Dropoff reaches attached-head clearance Y before crossing X. It moves to
`detach_x`, approaches `attach_y`, slides to `park_x` at path speed, separates to
`close_y` at path speed, and retreats to empty-carriage clearance Y.

Pickup reaches empty-carriage clearance Y, crosses X to `park_x`, and approaches
`close_y`. It waits for the tool's current heater target using native
`SET_TOOL_TEMPERATURE ... WAIT=1`, then moves to `attach_y` and slides to
`detach_x`. Synchronous `VERIFY_TOOL_DETECTED T=...` checks the tool before any
attached-head departure. It then exits to attached-head clearance Y and restores
requested X, Y, Z in that order. Zero-valued restore coordinates are supported.

Dock engagement is strictly XY. There is no park Z, automatic Z hop, rounded
corner cutting or liftbar command. Provide print clearance before requesting a
change, for example through a slicer Z hop. A restored tool-offset Z may differ
between tools, so check travel clearance after offset calibration too.

## Commissioning and calibration

1. Check coupling contacts and native detection while stationary, with motors and
   heaters idle. Confirm the expected head is detected and no parked head reports
   present.
2. Review and test homing with a seated T0. Verify leveling and mesh positions
   against both parked heads and all umbilicals.
3. Align dock height and head screws mechanically, then measure both X positions
   and every clearance Y slowly in docking mode.
4. Enable one calibrated tool and test `UNSELECT_TOOL`, then its `T0` pickup from
   a clear position. Use a stop control you can reach. Do not run these commands
   before the measurements and electrical checks are complete.
5. Calibrate the next head, test it alone, then test both directions of toolchange.
6. After individual tests succeed, run controlled repetitions with
   `TEST_TOOL_DOCKING RESTORE_AXIS=XYZ`. Increase speed only after consistent
   seating and clearance are established.
7. Calibrate probe Z and inter-tool XYZ offsets. Start with a single-tool print,
   then a two-tool alignment print. Recheck at operating temperatures.

Dock geometry, probe Z offset and tool XYZ offsets solve different problems.
Moving a dock changes docking coordinates; changing a nozzle can change probe or
inter-tool offsets without changing dock geometry. Use the current
[tools calibration reference](../../docs/tools_calibrate.md) for sensor-based
calibration. A Nudge-style sensor needs its own configured input and reachable
position. This profile does not copy older MadMax calibration macros over the
current API or supply printer-specific sensor coordinates.

## Validation and troubleshooting

See [troubleshooting and migration](TROUBLESHOOTING.md) for failure cases.

```sh
python -m pip install jinja2
python -m unittest discover -s tests -v
bash -n install.sh
```

Tests render the actual config with Klipper's Jinja delimiters and exercise both
dock sides, guard rejection, native detection requirements, optional probe routing,
clearance order and restoration. Installer tests use temporary config and Klipper
extras paths, including reruns and streamed/interactive entrypoints. They do not
exercise live MCU communication, physical collisions or a complete printer runtime.
