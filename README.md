# Klipper config for VT.1548

This repository contains the Klipper configuration for a Voron Trident 300mm (VT.1548) with multi-material support via AFC, plus shared macros via `nerdygriffin-macros` and KAMP.

## Architecture Overview
- Main entry: `printer.cfg` includes subsystems and plugins.
- Symlinked plugins: `nerdygriffin-macros/` (shared macros), `KAMP/` (Adaptive Purge/Smart Park), `AFC_menu.conf` (KlipperScreen menu). Do not modify symlinked content; override locally.
- Hardware-specific: `nitehawk-36.cfg` (toolhead), `beacon.cfg` (probe).
- Local multi-material: `AFC/` contains the AFC hub configuration and macros.

## Installed Klipper Plugins

- [AFC KlipperScreen Add On](https://github.com/ArmoredTurtle/AFC-Klipper-Screen-Add-On)
- [AFC Klipper Add On](https://github.com/ArmoredTurtle/AFC-Klipper-Add-On)
- [Beacon Surface Scanner](https://github.com/beacon3d/beacon_klipper)
- [crowsnest](https://github.com/mainsail-crew/crowsnest)
- [katapult](https://github.com/Arksine/katapult)
- [Klippain-ShakeTune](https://github.com/Frix-x/klippain-shaketune)
- [Klipper-Adaptive-Meshing-Purging](https://github.com/kyleisah/Klipper-Adaptive-Meshing-Purging) (KAMP)
- [klipper-nerdygriffin-macros](https://github.com/NerdyGriffin/klipper-nerdygriffin-macros)
- [klipper_tmc_autotune](https://github.com/andrewmcgr/klipper_tmc_autotune)
- [KlipperScreen](https://github.com/KlipperScreen/KlipperScreen)
- [moonraker-obico](https://github.com/TheSpaghettiDetective/moonraker-obico)
- [timelapse](https://github.com/mainsail-crew/moonraker-timelapse)

## Notes
- Symlinked directories (`AFC_menu.conf`, `KAMP`, `nerdygriffin-macros`) contain files not tracked in this repo
- `AFC/` directory is local and printer-specific (multi-material configuration)
- Override behavior in local config; do not edit symlinked content
