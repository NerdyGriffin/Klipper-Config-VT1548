# Klipper config for VT.1548

This repository contains the Klipper configuration for a Voron Trident 300mm (VT.1548) with multi-material support via AFC, plus shared macros via `nerdygriffin-macros` and KAMP.

## Architecture Overview
- Main entry: `printer.cfg` includes subsystems and plugins.
- Symlinked plugins: `nerdygriffin-macros/` (shared macros), `KAMP/` (Adaptive Purge/Smart Park), `AFC_menu.conf` (KlipperScreen menu). Do not modify symlinked content; override locally.
- Hardware-specific: `nitehawk-36.cfg` (toolhead), `beacon.cfg` (probe).
- Local multi-material: `AFC/` contains the AFC hub configuration and macros.

## Shared Macros Integration

This config uses the [klipper-nerdygriffin-macros](https://github.com/NerdyGriffin/klipper-nerdygriffin-macros) plugin for hardware-agnostic macro support:
- 13 consolidated macros included via `nerdygriffin-macros/` symlink
- Automatic AFC detection and integration
- Per-printer customization via variable overrides in `printer.cfg`

Macros included:
- `auto_pid.cfg` - Automated PID tuning
- `beeper.cfg` - M300 beeper support
- `client_macros.cfg` - Pause/Resume/Cancel hooks (AFC-aware)
- `filament_management.cfg` - Load/Unload/Purge with AFC detection
- `heat_soak.cfg` - Chamber preheating with LED animations
- `maintenance_macros.cfg` - Maintenance helpers (SETTLE_BELT_TENSION, etc.)
- `rename_existing.cfg` - Enhanced G-code overrides
- `save_config.cfg` - Safe config saving
- `shaketune.cfg` - Shake&Tune wrappers
- `shutdown.cfg` - Safe shutdown operations
- `status_macros.cfg` - Status LED patterns
- `tacho_macros.cfg` - Fan preflight checks
- `utility_macros.cfg` - Utility helpers

## Directory Structure

```
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
├── mainsail.cfg
├── moonraker_obico_macros.cfg
├── moonraker-obico-update.cfg
├── nerdygriffin-macros -> [symlink to shared macros]
├── nitehawk-36.cfg
├── printer.cfg
├── print_macros.cfg
├── TEST_SPEED.cfg
└── timelapse.cfg
```

**Notes:**
- Symlinked directories (`AFC_menu.conf`, `KAMP`, `nerdygriffin-macros`) contain files not tracked in this repo
- `AFC/` directory is local and printer-specific (multi-material configuration)
- Override behavior in local config; do not edit symlinked content
