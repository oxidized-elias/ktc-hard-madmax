# KTC-Hard MadMax — `ktc-hard-madmax` branch

This fork adapts [Contomo/klipper-toolchanger-hard](https://github.com/Contomo/klipper-toolchanger-hard)
for MadMax, with current Kalico probing compatibility fixes. MadMax development
lives on the [`ktc-hard-madmax` branch](https://github.com/oxidized-elias/klipper-toolchanger-hard/tree/ktc-hard-madmax)
of [oxidized-elias/klipper-toolchanger-hard](https://github.com/oxidized-elias/klipper-toolchanger-hard).

Use `ktc-hard-madmax` when cloning or configuring updates for this version.
The separate Easy Install MadMax implementation lives in another repository;
its configuration and installer are different. Do not switch an installed printer
between implementations without a configuration migration.

## MadMax (experimental)

An opt-in XY-only profile for front/side-keyslot MadMax docks is available.
Run `./install.sh --madmax` from this checkout, or choose MadMax in the interactive
installer. Existing noninteractive installs remain extensions-only by default.
See [installation, native detection and calibration](examples/madmax/README.md)
and [troubleshooting](examples/madmax/TROUBLESHOOTING.md).
The profile blocks docking until measured coordinates are configured and has not
been validated on a physical printer.

## Kalico status

Current Kalico named-probe, context-aware session, retry/scrubbing and mesh-based
compensation support is handled through a native adapter. See
[verified versions, configuration and limits](docs/kalico-compatibility.md).

* Toolchanger, tool probes, calibration, rounded path and heater power distributor include Klipper/Kalico compatibility paths; validation coverage and limits are documented above.
* manual rail is untested and will likely break on kalico, neither did i test the other extras on kalico yet.

## Installation

Clone the named branch into a separate directory over SSH on the printer:

```sh
cd ~
git clone --branch ktc-hard-madmax --single-branch https://github.com/oxidized-elias/klipper-toolchanger-hard.git ktc-hard-madmax
cd ~/ktc-hard-madmax
./install.sh --madmax
```

Follow the [MadMax setup guide](examples/madmax/README.md) to configure real
hardware, native detection, homing and dock measurements before restarting
Klipper. The installer preserves `madmax-user.cfg` and does not restart Klipper
for MadMax. It does not edit `printer.cfg`.

Run the script from this clone. The upstream `wget ... | bash` command installs
upstream KTC-Hard, not this branch. The streamed fallback and secondary-only
installer retain upstream download behavior and are not the installation path
for this MadMax branch. `./install.sh --extensions-only` installs this checkout's
full set of extensions without adding a motion profile; it can replace another
toolchanger's Python links, so use it only when that is intended.

Optional [example temperature/slicer macros](examples/macros.cfg) are available;
review them for conflicts with existing macros before including them.

### Moonraker updates

Add to `moonraker.conf`, using your actual clone path if it differs:

```ini
[update_manager ktc-hard-madmax]
type: git_repo
channel: dev
path: ~/ktc-hard-madmax
origin: https://github.com/oxidized-elias/klipper-toolchanger-hard.git
primary_branch: ktc-hard-madmax
managed_services: klipper
```

Use one update-manager entry per checkout. Keep `primary_branch: ktc-hard-madmax`
for this installation; `main` does not contain these MadMax development changes. Back up editable
printer configuration before updating. Rerun `./install.sh --madmax` when new
Python extensions or symlinks are added, then review and restart manually.

## Changelog

* 2026.04.06 - Fix `abort_on_tool_missing` mis-firing if there are waits during tool change.
* 2026.02.05 - planned obsolescence of tool_probe_endstop
* 2025.12.26 - **Breaking change** Stop using G-code offsets for tool offsets. Uses a dedicated gcode transform instead. Hopefully making the code more robust.
* 2025.12.25 - Use Bezier curves for rounded paths.
* 2025.12.25 - Manual rail update to latest Klipper.

## Updates that add new files

Note that if an update has new `klipper/extra` files, they **will not** be automatically installed into Klipper.
You will need to run the intall script manualy to add them:

```commandline
bash ~/ktc-hard-madmax/install.sh --madmax
```

## Main-Components

* [Toolchanger](docs/toolchanger.md) - tool management support.
* [Tool probe](docs/tool_probe.md) - per tool Z probe.
* [Probe multiplexer](docs/probe_multiplexer.md) - route multiple probes through one logical probe.
* [Rounded path](docs/rounded_path.md) - rounds the travel path corners for fast non-print moves.
* [Tools calibrate](docs/tools_calibrate.md) - support for contact based XYZ offset calibration probes.

## Extra-Components

* [Heater power distributor](docs/heater_power_distributor.md) - dynamic heater group power limiting (toolheads for example)
* [Sensorless auto tune](docs/sensorless_auto_tune.md) - automatically determines SGT/SGTHR values for your G28
* [Tool drop detection](docs/tool_drop_detection.md) - continous polling of accelerometers, with peak/angle detection (tool dropped)
* [Heater chamber fan](docs/heater_chamber_fan.md) - like a temperature_fan but it heats. automated chamber filter fan using bed as heater.

## Migrating from KTC-Easy or upstream KTC-Hard

Back up your printer config, tool definitions and calibration files before changing
implementations. Clone this branch into its own directory using the command above;
keep the old checkout available until migration has been tested.

Keep real hardware sections, but replace the old dock-motion include with
`[include toolchanger/madmax-include.cfg]`. Remove Easy Install's renamed
initialization/verification wrappers and any per-tool lift-out motion overrides.
Configure native `detection_pin` on every tool and merge measured MadMax geometry
into `toolchanger/madmax-user.cfg` using the new parameter names. Do not copy an
entire Easy Install configuration over this branch's native detection profile.

Read the [migration and troubleshooting guide](examples/madmax/TROUBLESHOOTING.md)
and [Kalico compatibility notes](docs/kalico-compatibility.md), then commission
homing, detection and docking before printing. Do not delete the old installation
or calibration backups as part of the initial migration.
