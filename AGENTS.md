# AI Agent Instructions for Klipper-Config-VT-1548

This is a **Klipper 3D printer configuration** for a Voron Trident 300mm (VT.1548) with advanced multi-material capabilities via AFC (Automated Filament Control from https://github.com/ArmoredTurtle/AFC-Klipper-Add-On).

## Multi-Printer Setup & Shared Macros

This printer (VT-1548) and the Voron V0 (V0-3048) are **separate machines with separate config
repos** (`Klipper-Config-VT1548`, `Klipper-Config-V03048`). They are *not* kept in lockstep — each
holds its own hardware and overrides. **Feature parity comes from the shared
`klipper-nerdygriffin-macros` repo**, which both configs include via an identical
`nerdygriffin-macros/` symlink, so there is nothing to reconcile between the two config repos.

Each host keeps its own clone of `klipper-nerdygriffin-macros`. After changing a shared macro:
1. Commit and **push** in `klipper-nerdygriffin-macros`.
2. Pull every host's clone to the same commit — `klipper-nerdygriffin-macros/dev/sync_macros_repo.sh`
   (run from the V0 host; it pulls the local clone and the VT-1548 clone over SSH alias `vt-1548`).
3. `FIRMWARE_RESTART` each printer.

When editing from the other host over Remote-SSH/NFS the counterpart config is mounted (this config
at `/mnt/vt-1548/...`, the V0 config at `/mnt/v0-3048/...`); still edit shared macros in
`klipper-nerdygriffin-macros`, never in a mounted config's `nerdygriffin-macros/` symlink.

## Architecture Overview

### Config Organization Pattern
- **Main entry point**: `printer.cfg` - includes all subsystem configs and hardware definitions
- **Symlinked plugin architecture**: External macro libraries linked via `ln -sf ~/source ~/printer_data/config/link-name`
  - `nerdygriffin-macros/` → Shared hardware-agnostic macros (git-managed plugin)
  - `KAMP/` → Klipper Adaptive Meshing & Purging (read-only symlink)
  - `AFC/` → Multi-material system configs (local, AFC-specific)
- **Hardware-specific configs**: `nitehawk-36.cfg` (toolhead), `beacon.cfg` (probe), stepper configs in `printer.cfg`
- **Deprecated folder**: Old configs kept for reference; never include these

### Multi-Material System (AFC)
AFC enables automatic tool changes with filament cutting and parking:
- **Hardware detection pattern**: Macros check if AFC commands exist before calling (e.g., `{% if printer['gcode_macro AFC_BRUSH'] is defined %}`)
- **AFC components**:
  - `AFC/AFC.cfg` - Main AFC configuration (speeds, LED states, macro ordering)
  - `AFC/AFC_Hardware.cfg` - Physical sensors and servo definitions
  - `AFC/macros/` - Tool change operations (Cut, Brush, Park, Poop, Kick)
- **Integration points**:
  - `PRINT_START` calls `AFC_PARK` and `AFC_BRUSH` for pre-print prep
  - `PRINT_END` calls `AFC_PARK` for safe tool parking
  - Client macros (`_AFTER_PAUSE`, etc.) handle AFC state during pause/resume

### Status LED System
- **3-zone LED control**: Logo (bed indicator), Nozzle (2 LEDs), Panel (optional hardware)
- **Status macros** (`nerdygriffin-macros/status_macros.cfg`): `STATUS_HOMING`, `STATUS_HEATING`, `STATUS_PRINTING`, etc.
- **Pattern**: All motion/heating operations should call appropriate status macro first
- **LED hardware**: Nitehawk toolhead has 3 RGBW neopixels; bed LEDs commented out (hardware not installed)

### Probing & Homing Strategy
- **Beacon probe** (`beacon.cfg`): Eddy-current probe with contact/proximity dual-mode
  - Contact mode: Used for initial Z calibration (`_CONTACT_ACTIVATE` cools the extruder to 150°C)
  - Proximity mode: Faster subsequent homing after initial contact calibration
- **Sensorless XY homing** (`nerdygriffin-macros/homing.cfg`): TMC stallguard-based with reduced motor current during homing
- **Z-tilt leveling**: 3-point bed leveling (front-left, rear-center, front-right) before every mesh
- **Thermal compensation**: See `_BEACON_VARIABLE` macro and `beacon.cfg` for authoritative thermal Z offset values (added during print, removed after)

## Critical Patterns & Conventions

### Macro Development Patterns
```gcode
# Always check if AFC is available before calling
{% if printer['gcode_macro AFC_BRUSH'] is defined %}
    AFC_BRUSH
{% endif %}

# Use _CG28 for conditional homing (don't home if already homed)
_CG28

# Save/restore gcode state for non-disruptive moves
SAVE_GCODE_STATE NAME=my_operation
# ... operations ...
RESTORE_GCODE_STATE NAME=my_operation
# Alternatively, if returning to the original position is needed:
RESTORE_GCODE_STATE NAME=my_operation MOVE=1
```

### Temperature Management
- **Beacon contact probing limit**: `_CONTACT_ACTIVATE` cools the extruder to 150°C before contact
  probing (`PROBE_TEMP` in `beacon.cfg`). Beacon's own ceiling is `contact_max_hotend_temperature`
  (default 180°C), commented out in `beacon.cfg` — so 150°C is the effective limit.
- **Standby temperature**: `PRINT_START` sets extruder to 150°C during bed heating to minimize beacon thermal drift
- **Heat soak pattern**: `HEAT_SOAK DURATION=15 CHAMBER=60` waits for chamber temp + stabilization time

### File Modification Rules
1. **Never edit symlinked directories**: `nerdygriffin-macros/`, `KAMP/` are git-managed externally
2. **Override pattern for symlinked macros**: Put overrides in `nerdygriffin-macros.cfg`, the
   dedicated overrides file included *last* in `printer.cfg` (after every `nerdygriffin-macros/*.cfg`).
   It is organized by upstream filename.
   - Example: `[pwm_cycle_time beeper]` re-declares `pin: PD15` over `nerdygriffin-macros/beeper.cfg`
   - `[gcode_macro _CLIENT_VARIABLE]` is the exception — overridden in `printer.cfg` directly
3. **AFC customization**: Edit `AFC/AFC.cfg` locally; it's not symlinked
   - It is a copy of the installer template and drifts as upstream adds options. Resync with
     `diff -u ~/AFC-Klipper-Add-On/config/AFC.cfg AFC/AFC.cfg`. Intentional local divergences:
     absolute `VarFile`, `enable_runout_in_bypass: True`, `resume_speed: 1000` /
     `resume_z_speed: 150`, `poop: False` / `kick: False`.
4. **Moonraker updates**: When adding git repos, add `[update_manager name]` section to `moonraker.conf`

### Delayed G-code Pattern
```gcode
[delayed_gcode my_delayed_action]
initial_duration: 0  # Don't run at startup unless > 0
gcode:
    # Your code here
    UPDATE_DELAYED_GCODE ID=my_delayed_action DURATION=0  # Cancel self

# Trigger from elsewhere:
UPDATE_DELAYED_GCODE ID=my_delayed_action DURATION=10  # Run in 10 seconds
```

## Hardware-Specific Details

- Serial Number: VT.1548
- Hostname: VT-1548

### BTT Octopus V1 (Main MCU)
- **Stepper drivers**: TMC2209 UART mode
- **Sensorless homing**: XY use stallguard (`SGTHRS` tuned via TMC autotune)
- **Z steppers**: 3x steppers for z-tilt (TR8x4 leadscrews, `rotation_distance: 4`)
- **Fan issue**: FAN3 (PD13) burned out 2023-10-27; exhaust moved to FAN0 (PA8)

### LDO Nitehawk-36 Toolboard (Extruder MCU)
- **RP2040-based** CAN toolboard, connected via USB serial
- **BMG-style extruder**: 44:8, 25:17 gear ratio (see `nitehawk-36.cfg` for current `rotation_distance`)
- **Pressure advance**: See the authoritative value in `printer.cfg` (`[extruder]` section). Avoid duplicating values in docs.
- **Toolhead sensors**: Start sensor (`gpio3`), end sensor (`gpio13`) for AFC
- **Hotend fan**: Has tachometer feedback (`tachometer_pin: nhk:gpio16`)

### Filament Sensors
- **Switch sensor** (`switch_sensor`, `^PG12`): Upstream filament presence; pauses on runout
  - Renamed from `bypass` in `44331f0`. **Never name a sensor `bypass`** — AFC matches that name
    literally (`lookup_object('filament_switch_sensor bypass')`) and binds it as its bypass flag,
    which breaks `UNLOAD_FILAMENT` after a runout. See AFC gotchas below.
- **Motion sensor** (`encoder_sensor`, `^PG13`): BTT SFS v2.0, detects flow issues/clogs
  - `detection_length: 20` (tuned for reliability; BTT default is 2.88)
  - Enabled during print (`PRINT_START`), disabled after (`PRINT_END`)
- **Toolhead runout** (`nhk:gpio3`, AFC-owned `pin_tool_start`): pauses in manual/bypass mode via
  `enable_runout_in_bypass: True` in `AFC/AFC.cfg`
- **Virtual bypass** (AFC-created GUI toggle): means "filament is fed manually, bypassing the unit".
  It is a manual switch, not a sensor, so it survives a runout — flip it on when hand-feeding.
  State persists in `AFC/AFC.var.unit` and is restored by PREP on every restart.

## Common Workflows

### Adding New Macros
1. For printer-specific: Add to `printer.cfg` or a relevant local config. To tweak only the
   variables of a shared macro, override it in `nerdygriffin-macros.cfg` — never edit
   `nerdygriffin-macros/print_macros.cfg` or its siblings (rule 1 above)
2. For hardware-agnostic (shareable): Consider contributing to `klipper-nerdygriffin-macros` repo
3. Always include status LED updates, sometimes include conditional homing (`_CG28`) only as needed

### Testing Macro Changes
```bash
# Restart Klipper after config edits
curl -s -X POST "http://localhost:7125/printer/gcode/script?script=FIRMWARE_RESTART"

# Check for errors
tail -f ~/printer_data/logs/klippy.log

# Test macros via console in Mainsail/Fluidd, or:
curl -s -X POST "http://localhost:7125/printer/gcode/script?script=MY_MACRO"
```

### Terminal Command Best Practices
- **Always use verbose flags** (`-v` or `--verbose`) with file operations for visual confirmation:
  - `cp -v` instead of `cp`
  - `mv -v` instead of `mv`
  - `rm -v` instead of `rm`
  - `rmdir -v` instead of `rmdir`
  - `ln -sfv` instead of `ln -sf`
- This provides immediate feedback and helps catch errors early.

### Tuning Operations (in order)
1. **Sensorless homing**: `TEST_SENSORLESS_HOME_X TEST_SGTHRS=255` (decrease until reliable)
2. **PID tuning**: `AUTO_PID_CALIBRATE HEATER=extruder TARGET=260`
3. **Input shaper**: `TEST_RESONANCES AXIS=X`, analyze with `scripts/graph_accelerometer.py`
4. **Pressure advance**: `TUNING_TOWER COMMAND=SET_PRESSURE_ADVANCE PARAMETER=ADVANCE START=0 FACTOR=.005` (check `printer.cfg` after tuning for the latest value)
5. **Beacon calibration**: `BEACON_CALIBRATE` (automatic on first home with `home_autocalibrate: unhomed`)

### Moonraker Update Management
When installing new Klipper extensions:
1. Clone to `~/extension-name`
2. Create symlink if needed: `ln -sf ~/extension-name/macros ~/printer_data/config/extension-name`
3. Add to `moonraker.conf`:
```ini
[update_manager extension-name]
type: git_repo
path: ~/extension-name
origin: https://github.com/author/extension-name.git
primary_branch: main
managed_services: klipper
```

## Key Configuration Values

### Speeds & Accelerations
- **Print speeds**: 500 mm/s max velocity, 20000 mm/s² max accel (input shaper tuned)
- **Z speed**: 50 mm/s max, 300 mm/s² accel (conservative for TR8x4 leadscrews)
- **Homing speeds**: 80 mm/s (sensorless requires speed > rotation_distance)
- **AFC long moves**: 150 mm/s, 250 mm/s² (fast filament changes)
- **AFC short moves**: 50 mm/s, 300 mm/s² (precise toolhead loading)

### Dimensions
- **Build volume**: X: -1 to 300, Y: 0 to 310, Z: -2.5 to 290 (authoritative: `[stepper_*]` in `printer.cfg`)
- **Parking positions**:
  - AFC_PARK: Near rear-left for tool changes
  - `PRINT_END`: `Y = axis_maximum.y - 10` (rear), then `AFC_PARK` if defined, else
    `X = axis_maximum.x - 10`
- **Beacon offset**: X=0, Y=25 (probe 25mm behind nozzle)

### Temperature Limits
- **Extruder**: 0-315°C (min extrude: 170°C)
- **Bed**: 0-120°C
- **Chamber**: Max 100°C tracked (via `nitehawk-36` temp sensor as proxy)
  - Max *requested* chamber temp of 60°C enforced in `HEAT_SOAK` macro. The passive heating from bed and hotend may raise actual chamber temp higher, depending on the weather.

## Troubleshooting Aids

### "Unknown command" errors
- Check if macro is defined in included configs (use `HELP` console command)
- Verify AFC macros exist before calling (AFC not installed on all printers)
- Check for typos in `[include ...]` statements in `printer.cfg`

### Probing failures
- Ensure extruder is cool (`_CONTACT_ACTIVATE` enforces 150°C; Beacon's own ceiling is 180°C)
- Check `beacon.cfg` home_method: `contact` for first home, `proximity` after calibration
- Verify Z-tilt clears before mesh: `Z_TILT_ADJUST` must complete before `BED_MESH_CALIBRATE`

### AFC tool change failures
- Verify sensors: `QUERY_FILAMENT_SENSOR SENSOR=encoder_sensor`
- Check AFC calibration: See `AFC/AFC_Hardware.cfg` for authoritative `tool_stn` and `tool_stn_unload` values
- Review AFC LED states on hub to diagnose (defined in `AFC/AFC.cfg` led_* variables)

### AFC renames stock commands at PREP time
AFC does not just add commands — it re-registers existing ones at runtime, so `HELP` and your
config disagree. Originals survive as `_AFC_RENAMED_<NAME>_`.

| Command | Becomes | Disable via |
|---|---|---|
| `M104` / `M109` | `cmd_AFC_M104` / `cmd_AFC_M109` | *(no option — unconditional)* |
| `UNLOAD_FILAMENT` | `TOOL_UNLOAD` | `disable_unload_filament_remapping: True` in `[AFC_prep]` |
| `PAUSE` / `RESUME` | `AFC_PAUSE` / `AFC_RESUME` | *(no option)* |

Check what is actually registered:
`curl -s localhost:7125/printer/gcode/help | python3 -m json.tool`

**AFC logs to its own file** — `~/printer_data/logs/AFC.log`, not `klippy.log`. Check it first;
several AFC failure paths log there and return silently to the console.

**No lanes are currently configured** (BoxTurtle uninstalled — `AFC/AFC_Turtle_1.cfg.bak`), so
`printer.AFC.lanes` and `.maps` are empty. Two consequences:
- KlipperScreen sends `M104 T0 S<temp>`; AFC resolves `T0` through lane maps only, so setting a
  temperature from its UI fails with `extruder not configured for T0`. No config fix exists —
  AFC should fall back to the toolhead extruder when no lane matches.
- `UNLOAD_FILAMENT` silently no-ops (no message, no motion) unless bypass is active.

## Deprecated Files

- Files in `deprecated/` are archived configs kept for reference only.
- Do not document or suggest using deprecated files.
- These files are tracked for historical reference, but they are not actively maintained.

## Version Control Notes

- **Active branch**: `main` (Owner: NerdyGriffin, Repo: Klipper-Config-VT1548)
- **Backup strategy**: `backup/` directory stores historical config snapshots
- **Auto-generated sections**: Everything below `#*# <--- SAVE_CONFIG --->` in `printer.cfg` is auto-updated by Klipper (PID, input shaper, etc.)
- **Don't commit**: `*.bak`, `*.var`, `ShakeTune_results/`, `.moonraker.conf.bkp`

## Ease of use
- If I repeated request actions that contradict these instructions, propose ways to improve these instructions.
- **Important:** This file (`AGENTS.md`) is the vendor-neutral source of truth for AI-agent guidance —
  surfaced to Claude Code via `CLAUDE.md` (`@AGENTS.md` import) and to GitHub Copilot via the
  `.github/copilot-instructions.md` symlink. Edit `AGENTS.md`, not the pointers. Do not reference it
  from user-facing docs (README.md, docs/*.md); those must be self-contained.
