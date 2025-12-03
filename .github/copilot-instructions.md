# AI Agent Instructions for Klipper-Config-VT-1548

This is a **Klipper 3D printer configuration** for a Voron Trident 300mm (VT.1548) with advanced multi-material capabilities via AFC (Automated Filament Control from https://github.com/ArmoredTurtle/AFC-Klipper-Add-On).

## Remote Access Context

⚠️ **When accessed from V0-3048 Remote-SSH:**
- This config is NFS-mounted at `/mnt/vt-1548/printer_data/config` (writable)
- To edit macros: work in V0-3048's `/home/pi/klipper-nerdygriffin-macros`, then sync using `dev/sync_macros_repo.sh`

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
- **Status macros** (`status-macros.cfg`): `STATUS_HOMING`, `STATUS_HEATING`, `STATUS_PRINTING`, etc.
- **Pattern**: All motion/heating operations should call appropriate status macro first
- **LED hardware**: Nitehawk toolhead has 3 RGBW neopixels; bed LEDs commented out (hardware not installed)

### Probing & Homing Strategy
- **Beacon probe** (`beacon.cfg`): Eddy-current probe with contact/proximity dual-mode
  - Contact mode: Used for initial Z calibration (requires cooling to <180°C)
  - Proximity mode: Faster subsequent homing after initial contact calibration
- **Sensorless XY homing** (`homing.cfg`): TMC stallguard-based with reduced motor current during homing
- **Z-tilt leveling**: 3-point bed leveling (front-left, rear-center, front-right) before every mesh
- **Thermal compensation**: `thermal_z_offset: 0.070` added during print, removed after (`_BEACON_VARIABLE`)

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
- **Beacon contact probing limit**: Must cool extruder to ≤150°C (enforced in `_CONTACT_ACTIVATE`)
- **Standby temperature**: `PRINT_START` sets extruder to 150°C during bed heating to minimize beacon thermal drift
- **Heat soak pattern**: `HEAT_SOAK DURATION=15 CHAMBER=60` waits for chamber temp + stabilization time

### File Modification Rules
1. **Never edit symlinked directories**: `nerdygriffin-macros/`, `KAMP/` are git-managed externally
2. **Override pattern for symlinked macros**: Define replacement in `printer.cfg` or local config
   - Example: `[pwm_cycle_time beeper]` overrides pin after including `nerdygriffin-macros/beeper.cfg`
3. **AFC customization**: Edit `AFC/AFC.cfg` locally; it's not symlinked
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
- **BMG-style extruder**: 44:8, 25:17 gear ratio, `rotation_distance: 35.2` (calibrate per printer)
- **Pressure advance**: See the authoritative value in `printer.cfg` (`[extruder]` section). Avoid duplicating values in docs.
- **Toolhead sensors**: Start sensor (`gpio3`), end sensor (`gpio13`) for AFC
- **Hotend fan**: Has tachometer feedback (`tachometer_pin: nhk:gpio16`)

### Filament Sensors
- **Switch sensor** (`bypass`): Detects filament presence before toolhead
- **Motion sensor** (`encoder_sensor`): BTT SFS v2.0, detects flow issues/clogs
  - `detection_length: 20` (tuned for reliability; BTT default is 2.88)
  - Enabled during print (`PRINT_START`), disabled after (`PRINT_END`)

## Common Workflows

### Adding New Macros
1. For printer-specific: Add to `printer.cfg`, `print_macros.cfg`, or relevant local config
2. For hardware-agnostic (shareable): Consider contributing to `klipper-nerdygriffin-macros` repo
3. Always include status LED updates, sometimes include conditional homing (`_CG28`) only as needed

### Testing Macro Changes
```bash
# Restart Klipper after config edits
sudo systemctl restart klipper

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
- **AFC long moves**: 150 mm/s, 250 mm/s² (fast filament changes) (consider increasing if reliable)
- **AFC short moves**: 50 mm/s, 300 mm/s² (precise toolhead loading)

### Dimensions
- **Build volume**: 300x310x280mm (X: 0-300, Y: 0-310, Z: -2.5 to 280)
- **Parking positions**:
  - AFC_PARK: Near rear-left for tool changes
  - Standard park: Dynamic — `X=axis_max/2`, `Y=axis_max-2` (computed at runtime in `PRINT_END`)
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
- Ensure extruder is cool (<180°C for contact mode)
- Check `beacon.cfg` home_method: `contact` for first home, `proximity` after calibration
- Verify Z-tilt clears before mesh: `Z_TILT_ADJUST` must complete before `BED_MESH_CALIBRATE`

### AFC tool change failures
- Verify sensors: `QUERY_FILAMENT_SENSOR SENSOR=encoder_sensor`
- Check AFC calibration: `tool_stn` (27.23mm) and `tool_stn_unload` (96.8mm) in `AFC_Hardware.cfg`
- Review AFC LED states on hub to diagnose (defined in `AFC/AFC.cfg` led_* variables)

## Version Control Notes

- **Active branch**: `main` (Owner: NerdyGriffin, Repo: Klipper-Config-VT1548)
- **Backup strategy**: `backup/` directory stores historical config snapshots
- **Auto-generated sections**: Everything below `#*# <--- SAVE_CONFIG --->` in `printer.cfg` is auto-updated by Klipper (PID, input shaper, etc.)
- **Don't commit**: `*.bak`, `*.var`, `ShakeTune_results/`, `.moonraker.conf.bkp`
