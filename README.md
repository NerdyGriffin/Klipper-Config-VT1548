# Klipper config for VT.1548

This repository contains the Klipper configuration for a Voron Trident 300mm (VT.1548) with multi-material support via AFC, plus shared macros via `nerdygriffin-macros` and KAMP.

## Architecture Overview
- Main entry: `printer.cfg` includes subsystems and plugins.
- Symlinked plugins: `nerdygriffin-macros/` (shared macros), `KAMP/` (Adaptive Purge/Smart Park), `AFC_menu.conf` (KlipperScreen menu). Do not modify symlinked content; override locally.
- Hardware-specific: `beacon.cfg` (probe), `nitehawk-36.cfg` (extruder, sensors), `status-macros.cfg`.
- Local multi-material: `AFC/` contains the AFC hub configuration and macros.

## Workflows
- Restart + tail logs after config edits:
	```bash
	sudo systemctl restart klipper
	tail -f ~/printer_data/logs/klippy.log
	```
- Test macros quickly:
	```bash
	curl -s -X POST "http://localhost:7125/printer/gcode/script?script=AFC_PARK"
	curl -s -X POST "http://localhost:7125/printer/gcode/script?script=HEAT_SOAK%20CHAMBER=50%20DURATION=10"
	```

## Notes
- Symlinked directories (`AFC_menu.conf`, `KAMP`, `nerdygriffin-macros`) contain files not tracked in this repo. Override behavior in local config; do not edit symlinked content.
- AFC configuration lives under `AFC/` and is printer-specific.

This is the contents of the /home/pi/printer_data/config/ directory

```
$ tree
.
├── .github
│   └── [copilot instructions & prompts]
├── AFC
│   └── [multi-material system configs]
├── AFC_menu.conf -> [symlink to AFC KlipperScreen menu]
├── autotune.cfg
├── beacon.cfg
├── crowsnest.conf
├── deprecated
│   └── [archived configs]
├── homing.cfg
├── KAMP -> [symlink to KAMP installation]
├── KAMP_Settings.cfg
├── KlipperScreen.conf
├── led-effects.cfg
├── nerdygriffin-macros -> [symlink to shared macros]
├── nitehawk-36.cfg
├── printer.cfg
├── print_macros.cfg
├── status-macros.cfg
└── TEST_SPEED.cfg

Note: Symlinked directories (AFC_menu.conf, KAMP, nerdygriffin-macros) contain files not tracked in this repo.
AFC directory contains local multi-material configuration.
See git history for complete file listing.
```
