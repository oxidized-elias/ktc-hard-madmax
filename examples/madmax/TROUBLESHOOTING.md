# MadMax troubleshooting and migration

Start with [the setup guide](README.md) and
[MadMax's mechanical troubleshooting](https://github.com/zruncho3d/madmax/blob/main/TROUBLESHOOTING.md).

| Symptom | Checks |
| --- | --- |
| No MadMax installer option | Run this changed checkout's `install.sh --madmax`; verify `git branch --show-current` reports `ktc-hard-madmax` and the remote points to `oxidized-elias/klipper-toolchanger-hard` |
| `calibrate all dock coordinates` | Measure the named tool and explicitly enable its calibration flag |
| Missing/out-of-range coordinate | Replace `None`; verify docking coordinates and configured X/Y travel; do not expand travel merely to silence the guard |
| Wrong Y ordering | Confirm front-dock orientation and every tool's Y overrides |
| Native detection required | Configure and electrically verify `detection_pin` on every tool |
| Wrong head detected | Check unique tool numbers, input polarity, parked-head states and wiring |
| Pickup pauses at close Y | Check the incoming tool's heater target; the native temperature wait occurs there |
| Screws strike keyslot edges | Recheck dock height, head angle, attach Y and detach X in docking mode |
| Empty carriage drags another head | Recheck empty-carriage clearance against all heads and magnets |
| Attached head clips another head | Recheck attached-head clearance and unrelated homing/purge/parking paths |
| Docking works at one Z only | Confirm dock and carriage remain vertically aligned; this profile has no dock-Z move |
| Printed colors misalign | Calibrate inter-tool XYZ offsets and verify repeatable seating, separately from dock geometry |
| Extra move after pickup | Remove per-tool `perform_restore_move: True` overrides; the MadMax profile performs the restoration |

## Detection and probing

Native tool detection and the probe state are different interpretations of the
coupling contact. The native detector must report PRESENT when seated, while a
coupling probe must be untriggered when seated and triggered when lifted. This
fork supports sharing the pin; check each logical interpretation rather than
assuming identical `!` prefixes are correct.

A separate bed probe does not replace coupling-based presence detection. If using
per-tool probing, configure a probe for every selectable tool because the profile
routes the active probe after selection. If using a separate bed probe, remove
unused per-tool probe/endstop definitions so that routing is not attempted.

Do not load Easy Install's renamed command wrappers for initialization or
verification. This fork detects tools directly. Avoid calling legacy probe-crash
start/stop commands around this profile: its native missing-tool monitor already
handles intentional release and pickup. Check native monitoring during realistic
printing conditions, including the configured delay and virtual-SD behavior.

After a failure, treat the physical tool state as unknown. Turn off heaters,
support loose parts, and inspect the coupling, dock and wiring before another
home or initialization. A pause response is not a mechanical recovery routine.

## Migration from another motion profile

Back up your printer, tool and calibration configuration first. Keep your real
hardware sections, but replace the lift-out/fixed-dock motion include with the
MadMax wrapper. Remove per-tool pickup/dropoff/recovery overrides that still use
park Z, liftbar motion or another mechanism's paths. Do not include both motion
profiles and rely on incidental section order to choose one.

The editable template does not define T0/T1 automatically. Merge dock values into
matching existing tools and uncomment only the sections you actually use. Native
T commands are supplied by the core; custom T macros should forward `rawparams`
if you need restore-axis options to reach `SELECT_TOOL`.

Rerunning the MadMax installer preserves `madmax-user.cfg`. New template options
may need to be merged manually into an existing file. Switching the installer to
extensions-only does not rewrite printer includes or delete measurements; change
the active dock include explicitly when changing mechanisms.

## Reporting hardware results

Include the repository commit, Klipper or Kalico commit, printer/gantry type,
dock mounting, toolhead design and the exact failing stage. Provide sanitized
console output and the relevant native detection, probe, XY coordinate and speed
settings. State whether the failure occurs on the left, right, or both docks and
whether chamber temperature or umbilical movement affects it.

The software tests are useful for macro and installer behavior. Physical seating,
contact repeatability, print clearance and dock reliability still need testing on
the actual printer.

## Branch and update checks

Inside the installed checkout, inspect:

```sh
git remote -v
git branch --show-current
git log -1 --oneline
```

For this implementation use `ktc-hard-madmax`, which contains the MadMax work in this fork. Set Moonraker's `primary_branch` accordingly. If you
installed the wrong branch, keep a configuration backup and clone the intended
branch into a separate directory before changing extension symlinks. Do not
blindly switch an active installation between implementations.

If your checkout tracks another branch, switch to `ktc-hard-madmax` with
an unchanged working tree:

```sh
git fetch origin
git switch ktc-hard-madmax
git merge --ff-only origin/ktc-hard-madmax
```

Verify that `origin` points to `oxidized-elias/klipper-toolchanger-hard` before
running those commands. Set Moonraker's `primary_branch: ktc-hard-madmax`; keep its `path`
pointing to your actual existing clone directory.
