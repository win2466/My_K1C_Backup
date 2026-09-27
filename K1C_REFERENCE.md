# K1C Klipper Configuration & Troubleshooting Reference

**Version:** 2.0.9-corrections
**Last updated:** 2026-09-27
**Printer:** Creality K1C (220x220x250 mm, CoreXY)
**Firmware:** Stock Klipper fork with Creality PRTouch v2 extensions
**Host:** Ingenic CPU, /usr/share/klipper/ install path
**UI:** GuppyScreen + Mainsail + OrcaSlicer

This document captures every change, fix, and lesson from the configuration
work done on this printer. Keep it alongside your config backups — it
explains WHY each change exists, so future-you doesn't undo something
important by accident.

### Changelog

- **v2.0.9-corrections** (this version)
  - Corrected TRSYNC_TIMEOUT explanation (it is NOT the identify-handshake
    timeout window).
  - Documented the overlayfs layout of /usr/share/klipper/.
  - Documented that the startup "Timeout on connect" is a self-healing
    race condition (empirically confirmed).
  - Added healthy-stat markers so future-you can tell at a glance whether
    the printer is actually sick.
  - Added Correction 6 to the Honest Corrections Log.

- **v2.0.9-quality-fix-11-12-13**
  - Initial consolidated reference.

---

## Table of Contents

1. System Overview
2. Config File Inventory & Roles
3. The Errors Encountered — Root Causes
4. Critical Fixes (Applied)
5. Quality / Robustness Fixes (Applied)
6. External Modifications (Non-Config)
7. Communication Issues & Workarounds
8. Deployment Reference
9. Verification Checklist
10. Rollback Procedures
11. Workflow Guides
12. Honest Corrections Log
13. What NOT to Change
14. Quick Command Reference
15. Appendices A-D

---

## 1. System Overview

| Component      | Detail                                        |
|----------------|-----------------------------------------------|
| Kinematics     | CoreXY                                        |
| Bed size       | 220 x 220 x 250 mm                            |
| Main MCU       | GD32F303RET6 @ /dev/ttyS7 @ 230400            |
| Nozzle MCU     | GD32F303CBT6 @ /dev/ttyS1 @ 230400            |
| Leveling MCU   | GD32E230F8P6 @ /dev/ttyS9 @ 230400            |
| Host MCU       | Linux @ /tmp/klipper_host_mcu                 |
| ADXL345        | On nozzle_mcu, software SPI, axes_map: x,-z,y |
| Probe          | PRTouch v2 (strain-gauge based, not BLTouch)  |
| Host install   | /usr/share/klipper/ (NOT ~/klipper/)          |
| Config path    | /usr/data/printer_data/config/                |
| Variables file | Helper-Script/variables.cfg                   |

### Key architectural note

This printer uses Creality's modified Klipper, not upstream. It installs to
/usr/share/klipper/ as a system package. Any online guide that says
~/klipper/ must be translated to /usr/share/klipper/.

### Filesystem layout — overlayfs

/usr/share/klipper/ is served through an overlayfs mount. Three paths point
at the same logical file:

    /rom/usr/share/klipper/...            read-only lower layer (factory)
    /overlay/upper/usr/share/klipper/...  writable upper layer (edits go here)
    /usr/share/klipper/...                merged view (what Klipper reads)

You edit the merged view. The overlay automatically routes writes to
/overlay/upper/. Do NOT edit /rom/ directly — it is read-only and any
attempt will either fail or be shadowed by the upper layer.

To locate a Klipper source file on this printer:

    find / -name "mcu.py" -path "*klippy*" 2>/dev/null

### Key PRTouch constraint

The PRTouch post-processor hardcodes mesh dimensions based on
[bed_mesh] probe_count. Passing a runtime PROBE_COUNT to
BED_MESH_CALIBRATE causes:

    IndexError: list index out of range
    -> {"code":"key60","msg":"Internal error on command:BED_MESH_CALIBRATE"}

This is why BED_MESH_CALIBRATE is wrapped in gcode_macro.cfg to strip
PROBE_COUNT from every caller.

---

## 2. Config File Inventory & Roles

### Active (included by printer.cfg)

| File                        | Role                                                             |
|-----------------------------|------------------------------------------------------------------|
| printer.cfg                 | Main config — MCUs, steppers, extruder, bed, fans, ADXL, PRTouch |
| sensorless.cfg              | Homing override, force-move helpers, XYZ ready state machine     |
| gcode_macro.cfg             | All macros except M600/pause — Z-offset, chamber, START_PRINT    |
| printer_params.cfg          | [fan_feedback], [custom_macro], [gcode_macro product_param]      |
| zoffset-guppy.cfg           | [save_variables], guarded SET_GCODE_OFFSET, ZOFFSET_CLEAR/RESTORE|
| Helper-Script/git-backup.cfg| Git backup macros                                                |
| GuppyScreen/*.cfg           | GuppyScreen UI macros and shell command wrappers                 |
| Dummy_M191.cfg              | Ignores M191 (chamber temp) — no chamber heater on this printer  |
| custom_purge.cfg            | _CUSTOM_PURGE_LINE used at start of every print                  |
| M600-support-custom.cfg     | Defines PAUSE, RESUME, M600, _UNLOAD_FILAMENT, _LOAD_FILAMENT    |

### Disabled (renamed .disabled, not loaded)

| File                                    | Why disabled                                        |
|-----------------------------------------|-----------------------------------------------------|
| M600-support.cfg.disabled               | Superseded by M600-support-custom.cfg               |
| Helper-Script/save-zoffset.cfg.disabled | Superseded by zoffset-guppy.cfg                     |
| useful-macros.cfg                       | Not included; utility library not referenced        |

### Global sections (defined once, in printer.cfg)

| Section                 | Where                    | Why                                  |
|-------------------------|--------------------------|--------------------------------------|
| [respond]               | printer.cfg only         | Duplicates cause key1 shutdown       |
| [save_variables]        | zoffset-guppy.cfg only   | Filename: Helper-Script/variables.cfg|
| [idle_timeout]          | M600-support-custom.cfg  | M600-aware gcode callback            |

---

## 3. The Errors Encountered — Root Causes

### 3.1 gcode_macro resume is not defined

Error: {"code":"key60", "msg":"...gcode_macro resume is not defined..."}

Root cause: M600-support.cfg (old) and M600-support-custom.cfg (new) both
defined [gcode_macro RESUME]. Creality's CX_* macros reference
printer["gcode_macro RESUME"] internally; when the name collision resolved
incorrectly, RESUME ended up undefined.

Fix: [include M600-support-custom.cfg] in printer.cfg;
M600-support.cfg renamed to .disabled (fix #14).

### 3.2 Unhandled exception during run (key1)

Error: {"code":"key1", "msg":"Unhandled exception during run..."}

Two distinct root causes:

a) Duplicate sections
   Two [respond] or two [save_variables] sections cause an internal crash
   that surfaces as key1.

b) Serial handshake timeout
   Actual traceback:
     File "/usr/share/klipper/klippy/serialhdl.py", line 74, in _get_identify_data
     TypeError: initializer for ctype 'struct serialqueue *' must be a
     cdata pointer, not NoneType

   Root cause: "mcu 'mcu': Timeout on connect" — the MCU did not respond
   to Klipper's identify packet within the handshake window. The
   serialqueue was still None when Klipper tried to send the identify
   request.

   IMPORTANT — this is NOT a TRSYNC_TIMEOUT problem. The identify
   handshake window is a separate, hardcoded ~5-second timeout in
   serialhdl.py. TRSYNC_TIMEOUT governs the clock-sync phase after
   identify succeeds. See Correction 6 in Section 12.

   Fix: none required. Klipper retries automatically and succeeds. See
   Section 7.1.

### 3.3 Move queue overflow

Error: {"code":"", "msg":"MCU 'mcu' shutdown: Move queue overflow"}

Root cause: Host CPU sent G-code commands faster than the main MCU could
process them. Common triggers on K1C:

- Power Loss Recovery (PLR) enabled — constant SD writes starve the queue
- Arc fitting enabled — G2/G3 expanded into hundreds of micro-moves
- Extrusion Rate Smoothing enabled — same effect
- Arachne wall generator — generates many short segments
- Verbose G-code output — bloats file with comments

Fix: Slicer-side changes only (see Section 7.2).

### 3.4 Bed mesh profile stuck at x_count = 4

Root cause: SAVE_CONFIG block at bottom of printer.cfg holds a persistent
profile separate from the [bed_mesh] section. Changing probe_count in the
main config does not update the saved profile. BED_MESH_PROFILE REMOVE
alone does not commit the removal — you must run SAVE_CONFIG immediately
after, THEN calibrate and save again.

Fix: Two-stage SAVE_CONFIG procedure (see Section 11.2).

---

## 4. Critical Fixes (Applied)

### Fix #1 — Stale 4x4 Bed Mesh Removed

What: Replace the 4x4 profile in the SAVE_CONFIG block with a fresh 5x5.

Why: The [bed_mesh] section specifies probe_count: 5,5, but the saved
profile in persistent memory was still 4x4. homing_override loaded this
stale profile on every G28, masking calibration issues.

Procedure (one-time, from console):

    BED_MESH_CLEAR
    BED_MESH_PROFILE REMOVE=default
    SAVE_CONFIG
    # wait for Klipper restart
    G28
    BED_MESH_CALIBRATE
    BED_MESH_PROFILE SAVE=default
    SAVE_CONFIG

Verification: SAVE_CONFIG block shows x_count = 5 and y_count = 5.

### Fix #2 — homing_override No Longer Loads Mesh During START_PRINT

File: sensorless.cfg

Changed: The BED_MESH_PROFILE LOAD="default" line at the end of
[homing_override] is now guarded:

    {% if printer['gcode_macro START_PRINT'].prepare|int == 0 %}
      BED_MESH_PROFILE LOAD="default"
    {% endif %}

Why: START_PRINT calls G28, which runs homing_override, which previously
loaded the default mesh. Wasted work, and could mask mesh issues since
START_PRINT replaces the mesh immediately after.

### Fix #3 — Dead PROBE_COUNT Handling Removed from BED_LEVELING

File: useful-macros.cfg

Changed: Removed the block that reads PROBE_COUNT and passes it to
BED_MESH_CALIBRATE. The wrapper in gcode_macro.cfg already strips it.

Why: Users could pass BED_LEVELING PROBE_COUNT=7,7 and believe it took
effect. It didn't.

### Fix #4 — [verify_heater extruder] Parameters Added

File: printer.cfg

Changed:

    [verify_heater extruder]
    check_gain_time: 120
    heating_gain: 1.0
    hysteresis: 10

Why: Previously empty, so Klipper used aggressive defaults
(check_gain_time: 20, heating_gain: 2.0). On K1C with a fast nozzle heater,
this occasionally triggered false "Heater extruder not heating at expected
rate" errors during heat-up from cold.

### Fix #5 — INPUTSHAPER Now Calibrates Both Axes

File: gcode_macro.cfg

Changed: Now runs SHAPER_CALIBRATE AXIS=x, then AXIS=y, each followed by
CXSAVE_CONFIG.

Why: Previous version only calibrated Y. Running INPUTSHAPER left X using
whatever shaper was last saved — potentially months old or from different
belt tension. Genuine half-calibration trap.

### Fix #6 — TUNOFFINPUTSHAPER Now Warns

File: gcode_macro.cfg

Changed: Added description: and a console warning before zeroing shapers.

Why: Macro name looked like a harmless toggle. Accidental click during
print silently removes ringing compensation. Warning makes effect explicit.

---

## 5. Quality / Robustness Fixes (Applied)

### Fix #11 — LOAD_MATERIAL / QUIT_MATERIAL Temperature Override

File: gcode_macro.cfg

Changed: Both macros now accept optional EXTRUDER_TEMP= parameter:

    {% set t = params.EXTRUDER_TEMP|default(printer.custom_macro.default_extruder_temp)|int %}
    M109 S{t}

Why: Previously always heated to default_extruder_temp (240 C). Running
Load or Unload with PLA (200 C) or PETG (230 C) cooked the filament in the
melt zone, causing silent flow degradation and eventual clogs.

Usage:

    LOAD_MATERIAL                          # 240 C (default)
    LOAD_MATERIAL EXTRUDER_TEMP=210        # PLA
    LOAD_MATERIAL EXTRUDER_TEMP=230        # PETG
    QUIT_MATERIAL EXTRUDER_TEMP=210        # PLA unload

Backwards compatible: Existing GuppyScreen/Mainsail buttons need no change.

### Fix #12 — SKEW_PROFILE LOAD Guard

File: gcode_macro.cfg

Changed: In START_PRINT, unconditional skew load became:

    {% if 'skew_correction default' in printer.configfile.settings %}
      SKEW_PROFILE LOAD=default
    {% endif %}

Why: Previously every print printed "!! Unknown skew profile: default" if
no profile had been saved. Harmless but noisy, and trains users to ignore
"!!" messages.

CRITICAL: The check is against printer.configfile.settings (which contains
the SAVE_CONFIG block), NOT printer.save_variables.variables.
SKEW_PROFILE SAVE=default writes to the config block via SAVE_CONFIG, not
to variables.cfg.

### Fix #13 — Qmode Dead Writes Removed

File: gcode_macro.cfg

Changed: Removed three SET_GCODE_VARIABLE lines from [gcode_macro Qmode]
that were immediately overwritten by the fan-scaling block below them:

    # Deleted — dead stores
    SET_GCODE_VARIABLE MACRO=Qmode VARIABLE=fan0_value VALUE={printer['output_pin fan0'].value}
    SET_GCODE_VARIABLE MACRO=Qmode VARIABLE=fan1_value VALUE={printer['output_pin fan1'].value}
    SET_GCODE_VARIABLE MACRO=Qmode VARIABLE=fan2_value VALUE={printer['output_pin fan2'].value}

Why: Values written at 0-1 scale were overwritten at 0-255 scale before
anything read them. Removal is provably behavior-neutral.

IMPORTANT CORRECTION: The original claim that these writes caused an eMMC
"Saving config..." stutter was WRONG. SET_GCODE_VARIABLE updates an
in-memory dict and fires a status broadcast — it does NOT write to disk.
Cosmetic cleanup, not a performance fix.

### Fix #14 — Dead .cfg Files Renamed to .disabled

Files affected: M600-support.cfg, Helper-Script/save-zoffset.cfg

Commands:

    cd /usr/data/printer_data/config
    mv M600-support.cfg M600-support.cfg.disabled
    mv Helper-Script/save-zoffset.cfg Helper-Script/save-zoffset.cfg.disabled

Why: Both files are previous versions of currently-active files. They are
un-included but contain colliding section definitions. Renaming eliminates
the possibility of accidentally re-including them and causing a key1
shutdown.

No Klipper restart needed. Filesystem-only change.

### Fix #16 — Adaptive Pressure Advance (Informational)

No file to upload. Adaptive PA is an OrcaSlicer feature, not a Klipper
config change. Your pressure_advance: 0.04 in [extruder] remains as the
fallback.

When adaptive PA helps: High-flow PETG/ABS prints, corner bulge at high
speed, very different layer heights in the same model.

When it doesn't: Standard PLA prints at typical K1C speeds. Static 0.04 is
fine.

To enable: OrcaSlicer -> Filament Settings -> Advanced -> Pressure Advance
-> add (flow, PA) calibration pairs. OrcaSlicer will emit
SET_PRESSURE_ADVANCE commands during the print.

### Fix #17 — [fan_feedback] Comments Added

File: printer_params.cfg

Changed: Added header comment block explaining that fan0_pin and fan1_pin
are TACHOMETER INPUTS, not PWM outputs:

    # Fan   | PWM output pin     | Feedback (tach) pin  | Role
    # ------|--------------------|----------------------|--------------------
    # fan0  | nozzle_mcu:PB8     | nozzle_mcu:PB4       | throat / model fan
    # fan1  | PC0                | PC6                  | backplane / exhaust
    # fan2  | PB1                | (none declared)      | hotend / part fan

Why: Section is confusing without context. fan0 uses PB8 for PWM and PB4
for tach — two different physical MCU pins.

---

## 6. External Modifications (Non-Config)

### TRSYNC_TIMEOUT Increase

File: /usr/share/klipper/klippy/mcu.py  (NOT ~/klipper/)

Default value: TRSYNC_TIMEOUT = 0.025
New value:     TRSYNC_TIMEOUT = 0.05

### What this parameter actually does

TRSYNC_TIMEOUT governs the sync/clock-check phase that runs AFTER the
initial identify handshake succeeds. It is the window Klipper allows for
the MCU to confirm it has received the timing reference before deciding
the clock is unstable.

### What this parameter does NOT do

It does NOT control the identify-handshake timeout. That is a separate,
hardcoded ~5-second window in serialhdl.py. Increasing TRSYNC_TIMEOUT will
NOT prevent "Timeout on connect" messages at cold boot.

It also does NOT affect the `rto=` value reported in the Stats lines.
`rto` (retransmit timeout) is computed at runtime from baud rate and
message size. At 230400 baud it will read `rto=0.025` regardless of
TRSYNC_TIMEOUT.

### Why the edit was still applied

Increased margin on the sync phase eliminates a class of subtle "clock not
converged" warnings that occasionally appear on K1C under CPU load. It is
a low-risk, defensible tweak even though it does not address the startup
identify race.

### Locate the file

Because of the overlayfs layout (see Section 1), the file can be found at
three paths. To confirm the actual file:

    find / -name "mcu.py" -path "*klippy*" 2>/dev/null

Expected:

    /overlay/upper/usr/share/klipper/klippy/mcu.py
    /rom/usr/share/klipper/klippy/mcu.py
    /usr/share/klipper/klippy/mcu.py

### Verify current value

    grep -n "TRSYNC_TIMEOUT" /usr/share/klipper/klippy/mcu.py

Expected output:

    128:TRSYNC_TIMEOUT = 0.05
    191:        expire_timeout = TRSYNC_TIMEOUT

### Editing method on K1C

BusyBox sed does not fully support bracket expressions like [0-9.], and
reports cryptic "unmatched '/'" errors. Use nano instead:

    sudo cp /usr/share/klipper/klippy/mcu.py /usr/share/klipper/klippy/mcu.py.bak
    sudo nano /usr/share/klipper/klippy/mcu.py
    # Ctrl+W -> search for TRSYNC_TIMEOUT
    # Change 0.025 to 0.05
    # Ctrl+X -> Y -> Enter to save

Or use a literal-value sed (works on BusyBox):

    sudo sed -i 's/TRSYNC_TIMEOUT = 0.025/TRSYNC_TIMEOUT = 0.05/' /usr/share/klipper/klippy/mcu.py

### Restart Klipper

    sudo systemctl restart klipper

### PERSISTENCE WARNING

Because the file lives in the overlayfs upper layer, the change survives
normal reboots. A firmware update — from Creality's official updater or
via the Helper Script's firmware tools — will typically reset the overlay.
Re-apply after every firmware update. Check with:

    grep "TRSYNC_TIMEOUT =" /usr/share/klipper/klippy/mcu.py

### Rollback

    sudo cp /usr/share/klipper/klippy/mcu.py.bak /usr/share/klipper/klippy/mcu.py
    sudo systemctl restart klipper

---

## 7. Communication Issues & Workarounds

### 7.1 "Timeout on connect" at startup — self-healing race condition

### Empirically confirmed behaviour

This printer has been observed to sometimes fail the initial identify
handshake on cold boot, then succeed on Klipper's automatic retry. From a
real log:

    02:01:44,726  mcu 'mcu': Starting serial connect
    02:01:49,888  mcu 'mcu': Timeout on connect        <- first attempt failed
    02:01:53,290  Loaded MCU 'mcu' 116 commands        <- retry succeeded
    02:01:53,292  mcu 'nozzle_mcu': Starting serial connect
    02:01:54,336  Loaded MCU 'nozzle_mcu' 116 commands
    02:01:54,338  mcu 'leveling_mcu': Starting serial connect
    02:01:55,371  Loaded MCU 'leveling_mcu' 116 commands
    02:01:56,100  Loaded MCU 'rpi' 104 commands

The same log shows other cold boots where no timeout occurred at all:

    06:00:21,796  mcu 'mcu': Starting serial connect
    06:00:22,869  Loaded MCU 'mcu' 116 commands        <- clean, 1 second

Same printer, same firmware, same config — the result varies. This is a
power-on race condition between the host's first identify packet and the
MCU's own boot sequence. It is inherent to the K1C and cannot be fully
eliminated by any config change.

### Symptoms in the console

On rare occasions the host Klipper may not retry successfully, and the
console will show:

    {"code":"key1", "msg":"Unhandled exception during run..."}

with a traceback ending in:

    File ".../serialhdl.py", line 74, in _get_identify_data
    TypeError: initializer for ctype 'struct serialqueue *' must be a
    cdata pointer, not NoneType

### Response — in order of effort

1. Wait 10 seconds. Klipper often recovers on its own via retry.
2. FIRMWARE_RESTART from the console. Recovers in most remaining cases.
3. Full power cycle — turn off with the physical switch, wait 30
   seconds, turn on. Always recovers.
4. If it happens on almost every boot, reseat mainboard cables,
   especially J11 and J51.

### Do NOT

- Do NOT re-flash the MCU firmware to "fix" this. The firmware is
  correct; the retry succeeds instantly. Re-flashing risks introducing
  real problems.
- Do NOT increase TRSYNC_TIMEOUT expecting it to prevent this. It will
  not. TRSYNC_TIMEOUT is unrelated (see Section 6).
- Do NOT panic. The printer self-recovers in the overwhelming majority
  of occurrences.

### 7.2 Move queue overflow

Symptoms: "MCU 'mcu' shutdown: Move queue overflow" during a print,
typically 5-30 minutes in.

Root cause: Host CPU sends commands faster than MCU can consume them.

Slicer fixes (required):

- Disable Power Loss Recovery: Add M413 S0 to start G-code
- Disable Arc Fitting: OrcaSlicer -> Printer Settings -> Advanced ->
  uncheck "Arc fitting"
- Disable Extrusion Rate Smoothing
- Switch to Classic wall generator (from Arachne)
- Disable Verbose G-code in output settings

If still occurring: Consider reducing print speed, or investigate host CPU
load with "top" over SSH during a print.

### 7.3 Diagnosing which MCU failed

Startup log shows each MCU loading in sequence:

    mcu 'mcu': Starting serial connect
    mcu 'mcu': Loaded MCU 'mcu' 116 commands (...
    mcu 'nozzle_mcu': Starting serial connect
    mcu 'nozzle_mcu': Loaded MCU 'nozzle_mcu' 116 commands (...
    mcu 'leveling_mcu': Starting serial connect
    mcu 'leveling_mcu': Loaded MCU 'leveling_mcu' 116 commands (...
    mcu 'rpi': Starting connect
    mcu 'rpi': Loaded MCU 'rpi' 104 commands (...

Whichever MCU is last in the log before the crash is the one that timed out.

### 7.4 Reading the Stats lines — healthy vs. sick

Klipper emits a Stats line every 3 seconds. Below is what to look for.

### Healthy

    mcu:   mcu_awake=0.004 ... bytes_invalid=0 ... stalled_bytes=0
           ready_bytes=0 freq=119997115
    nozzle_mcu:   ... bytes_invalid=0 stalled_bytes=0 ready_bytes=0
    leveling_mcu: ... bytes_invalid=0 stalled_bytes=0 ready_bytes=0
    rpi:          ... bytes_invalid=0 stalled_bytes=0 ready_bytes=0

### Sick — warning signs to watch for

| Indicator                     | What it means                          |
|-------------------------------|----------------------------------------|
| stalled_bytes > 0             | MCU cannot consume commands fast enough|
| ready_bytes climbing          | Outgoing buffer backing up             |
| bytes_invalid > 0             | Wire corruption (cable, EMI, ground)   |
| bytes_retransmit climbing     | Ongoing packet loss                    |
| mcu_awake > 0.05              | MCU spending >5% time in interrupts    |
| mcu_task_avg > 0.00005        | Tasks taking unusually long            |
| freq drifting from nominal    | Unstable clock (power supply issue)    |
| send_seq diverging from recv  | Lost packets in flight                 |

### Notes on baseline values

- `bytes_retransmit=9` with `retransmit_seq=2` on the three hardware
  MCUs is normal. It reflects two handshake retransmits during startup.
  If it stays frozen across samples, it is not a problem.
- `rpi` shows `bytes_retransmit=0` because it uses a Linux pipe
  (/tmp/klipper_host_mcu), not a serial UART.
- `rto=0.025` at 230400 baud is normal and expected. It is derived
  from the baud rate and message size, not from TRSYNC_TIMEOUT.

---

## 8. Deployment Reference

### Files to upload (in order)

| Priority | File                    | Purpose                              |
|----------|-------------------------|--------------------------------------|
| 1        | printer.cfg             | Main config with all critical fixes  |
| 2        | gcode_macro.cfg         | Fixes #5, #6, #11, #12, #13          |
| 3        | sensorless.cfg          | Fix #2                               |
| 4        | printer_params.cfg      | Fix #17 (comments)                   |
| 5        | M600-support-custom.cfg | Global [respond] moved out           |
| 6        | guppy_cmd.cfg           | Global [respond] removed             |
| 7        | useful-macros.cfg       | Fix #3 (only if included)            |

### Files to rename (SSH)

Note: These files live inside Helper-Script/ on this printer's actual
filesystem. Earlier instructions in v2.0.9-quality-fix-11-12-13 pointed at
the wrong path for M600-support.cfg.

    cd /usr/data/printer_data/config
    mv Helper-Script/M600-support.cfg Helper-Script/M600-support.cfg.disabled
    mv Helper-Script/save-zoffset.cfg Helper-Script/save-zoffset.cfg.disabled

Verify:

    ls /usr/data/printer_data/config/Helper-Script/*.disabled

### Console commands to run once after upload

    # Two-stage bed mesh reset (fix #1)
    BED_MESH_CLEAR
    BED_MESH_PROFILE REMOVE=default
    SAVE_CONFIG
    # [wait for restart]
    G28
    BED_MESH_CALIBRATE
    BED_MESH_PROFILE SAVE=default
    SAVE_CONFIG

---

## 9. Verification Checklist

Run these from the console after every deployment:

| #  | Command / Action                              | Expected Result                        |
|----|-----------------------------------------------|----------------------------------------|
| 1  | STATUS                                        | "state": "Ready"                       |
| 2  | GET_GCODE_OFFSET                              | z = -0.0275 (your saved offset)        |
| 3  | BED_MESH_OUTPUT PGP=1                         | 25 probe points (indices 0-24)         |
| 4  | Check SAVE_CONFIG block in printer.cfg        | x_count = 5, y_count = 5               |
| 5  | LOAD_MATERIAL EXTRUDER_TEMP=210               | Heats to 210 C, not 240 C              |
| 6  | LOAD_MATERIAL (no param)                      | Heats to 240 C                         |
| 7  | QUIT_MATERIAL EXTRUDER_TEMP=200               | Unloads at 200 C                       |
| 8  | Start print with no saved skew                | No "!! Unknown skew profile" message   |
| 9  | INPUTSHAPER                                   | Runs X and Y calibration in sequence   |
| 10 | TUNOFFINPUTSHAPER                             | Prints WARNING line                    |
| 11 | Heat nozzle from cold to 240 C                | No "Heater not heating at expected"    |
| 12 | ls Helper-Script/*.cfg.disabled               | Files exist                            |

---

## 10. Rollback Procedures

### Per-file rollback

Each file you uploaded was preceded by a backup command. To restore:

    cp /usr/data/printer_data/config/printer.cfg.bak /usr/data/printer_data/config/printer.cfg
    cp /usr/data/printer_data/config/gcode_macro.cfg.bak3 /usr/data/printer_data/config/gcode_macro.cfg
    cp /usr/data/printer_data/config/sensorless.cfg.bak /usr/data/printer_data/config/sensorless.cfg
    cp /usr/data/printer_data/config/printer_params.cfg.bak /usr/data/printer_data/config/printer_params.cfg
    sudo systemctl restart klipper

### Full config rollback (via git-backup)

    cd /usr/data/printer_data/config
    git log --oneline -10
    git checkout <commit-hash> -- printer.cfg

### TRSYNC_TIMEOUT rollback

    sudo cp /usr/share/klipper/klippy/mcu.py.bak /usr/share/klipper/klippy/mcu.py
    sudo systemctl restart klipper

### Bed mesh rollback

You cannot "roll back" a bed mesh profile. To revert, re-run calibration:

    BED_MESH_CLEAR
    G28
    BED_MESH_CALIBRATE
    BED_MESH_PROFILE SAVE=default
    SAVE_CONFIG

---

## 11. Workflow Guides

### 11.1 Z-Offset Calibration (Full Workflow)

Two separate offsets exist — do not confuse them:

| Offset          | Location                          | Purpose                          |
|-----------------|-----------------------------------|----------------------------------|
| PROBE Z-offset  | [prtouch_v2] z_offset in SAVE_CFG | Physical probe-to-nozzle. Once.  |
| GCODE Z-offset  | Helper-Script/variables.cfg       | Fine-tuning. Per filament/plate. |

Step 1 — Calibrate PROBE Z-offset (rarely):

    G28
    # Heat bed to printing temp, nozzle to 150 C
    PROBE_CALIBRATE
    # Use TESTZ Z=-0.05 / TESTZ Z=+0.05 until paper drags lightly
    ACCEPT
    SAVE_CONFIG

Step 2 — Tune GCODE Z-offset (per filament):

Option A — GuppyScreen: Open Z-offset menu during print, adjust in 0.01 mm
steps. Auto-persisted.

Option B — Console:

    SET_GCODE_OFFSET Z=-0.05 MOVE=0

Value is auto-saved (suppression flag is clear outside START_PRINT).

Step 3 — Verify: Restart Klipper. Watch for:

    Loaded Z-Offset from variables.cfg: <value>mm

Or run: GET_GCODE_OFFSET

Managing offsets:

| Command          | Effect                                           |
|------------------|--------------------------------------------------|
| ZOFFSET_CLEAR    | Zero live offset, block saves (used by START)    |
| ZOFFSET_RESTORE  | Re-apply saved offset, unblock saves             |
| ZOFFSET_APPLY    | Manual re-apply from console                     |
| Edit variables.cfg -> zoffset = {'z': 0}          | Permanent delete |

### 11.2 Bed Mesh Workflow (With Two-Stage Save)

When to recalibrate: After changing build plate, after nozzle/probe service,
when first layer uniformity drifts.

Full sequence:

    # Stage 1: Remove old profile
    BED_MESH_CLEAR
    BED_MESH_PROFILE REMOVE=default
    SAVE_CONFIG
    # [Klipper restarts]

    # Stage 2: Calibrate and save new profile
    G28
    BED_MESH_CALIBRATE
    BED_MESH_PROFILE SAVE=default
    SAVE_CONFIG
    # [Klipper restarts]

    # Verify
    BED_MESH_OUTPUT PGP=1

Why two stages: The SAVE_CONFIG after REMOVE is what commits the deletion
to disk. Without it, the stale profile is still present and the next SAVE
just overwrites it with another profile carrying the same stale dimensions.

Adaptive mesh (per-print): Handled automatically by START_PRINT when
OrcaSlicer passes MESH_MIN / MESH_MAX. PROBE_COUNT is always stripped. If
MESH_MIN/MESH_MAX are missing, START_PRINT falls back to full-bed
CX_PRINT_LEVELING_CALIBRATION.

### 11.3 Filament Load / Unload (With Temperature Override)

    # PLA
    LOAD_MATERIAL EXTRUDER_TEMP=210
    QUIT_MATERIAL EXTRUDER_TEMP=210

    # PETG
    LOAD_MATERIAL EXTRUDER_TEMP=230
    QUIT_MATERIAL EXTRUDER_TEMP=230

    # ABS / ASA (default)
    LOAD_MATERIAL
    QUIT_MATERIAL

### 11.4 M600 Filament Change

Triggered by M600 command or filament runout sensor. Prompt appears in
Mainsail/Fluidd/GuppyScreen with options:

- UNLOAD FILAMENT — runs _UNLOAD_FILAMENT
- LOAD FILAMENT — runs _LOAD_FILAMENT
- PURGE MORE — runs _PURGE_MORE
- CANCEL PRINT — runs CANCEL_PRINT
- IGNORE — runs _M600_IGNORE (verifies filament, reloads, stays paused)
- RESUME — runs RESUME (restores hotend temp, clears M600 state, resumes)

Temperature for _UNLOAD/_LOAD during M600 is read from
PRINTER_PARAM.hotend_temp, captured from the running print's target.
Already correct for the active material — no override needed.

---

## 12. Honest Corrections Log

Documented so future-you doesn't repeat the confusion.

### Correction 1 — SET_GCODE_VARIABLE and eMMC stutter

Claimed: "Every SET_GCODE_VARIABLE triggers a config-file save event in
Moonraker. On K1C's slow eMMC this can cause the 'Saving config...'
stutter during a print."

Reality: SET_GCODE_VARIABLE updates an in-memory dict and fires a status
broadcast. It does NOT write to disk. The "Saving config..." message comes
from SAVE_CONFIG and SAVE_VARIABLE. Qmode's writes have zero disk impact.

Impact: Fix #13 reframed from "performance fix" to "cosmetic dead-code
cleanup."

### Correction 2 — Skew profile existence check

Claimed: "Check printer.save_variables.variables.skew_profile to determine
if a skew profile exists."

Reality: SKEW_PROFILE SAVE=default writes to the SAVE_CONFIG block of
printer.cfg as a [skew_correction default] section. Correct check is
' skew_correction default' in printer.configfile.settings.

Impact: Fix #12's guard uses the correct check.

### Correction 3 — Klipper install path

Claimed: "Edit ~/klipper/klippy/mcu.py."

Reality: On K1C, Klipper is installed at /usr/share/klipper/, not in the
user's home directory. Every online guide referencing ~/klipper/ must be
translated.

Impact: All TRSYNC_TIMEOUT instructions reference the correct path.

### Correction 4 — BusyBox sed limitations

Claimed: Using bracket expressions like [0-9.] in sed commands is portable.

Reality: K1C uses BusyBox sed, which does not fully support bracket
expressions and reports cryptic "unmatched '/'" errors. Use nano, or
literal-value substitutions.

Impact: Editing instructions now default to nano.

### Correction 5 — Qmode writes and print stutter

Claimed: Consolidating Qmode's SET_GCODE_VARIABLE calls would reduce
stutter during print.

Reality: Qmode is a user-triggered macro run once or twice per print, not
a hot path. Even the 3 dead writes removed in fix #13 cost microseconds.
The macro should be considered stable and not further modified without a
specific reproducible reason.

### Correction 6 — TRSYNC_TIMEOUT is not the identify-handshake timeout

Claimed: "Increase TRSYNC_TIMEOUT to 0.05 s to eliminate Timeout on
connect. Watch the rto= value change in the Stats line as proof."

Reality: Two errors here.

First, TRSYNC_TIMEOUT does not govern the identify handshake. That is a
separate, hardcoded ~5-second window in serialhdl.py. Increasing
TRSYNC_TIMEOUT has no effect on the cold-boot identify race.

Second, `rto=` in the Stats lines is not connected to TRSYNC_TIMEOUT.
It is computed at runtime from baud rate and message size. At 230400 baud
it will read `rto=0.025` regardless of the source edit.

Impact: The startup race is self-healing via Klipper's automatic retry
(empirically confirmed). No config change is required for it. The
TRSYNC_TIMEOUT edit remains useful for the separate clock-sync phase but
should not be presented as a startup-timeout fix.

---

## 13. What NOT to Change

| Item                                | Why leave it                              |
|-------------------------------------|-------------------------------------------|
| Qmode further consolidation         | Not a hot path; load-bearing for motors   |
| Static PA (0.04 in [extruder])      | Fine for PLA at K1C speeds                |
| [output_pin LED] value: 1           | Cosmetic; leave if you use the LED        |
| useful-macros.cfg                   | Not included; don't add [include] casually|
| M600-support.cfg.disabled           | Don't rename back unless removing the new |
| save-zoffset.cfg.disabled           | Don't rename back under any circumstance  |
| [bed_mesh] probe_count: 5,5         | Do not raise without PRTouch verification |
| /rom/usr/share/klipper/...          | Read-only lower layer; edits routed via overlay |
| MCU firmware re-flash for startup   | Firmware is correct; race is self-healing |

---

## 14. Quick Command Reference

### Daily use

| Command            | Purpose                                    |
|--------------------|--------------------------------------------|
| FIRMWARE_RESTART   | Restart Klipper host without power cycling |
| STATUS             | Check printer state                        |
| GET_GCODE_OFFSET   | Read current Z-offset                      |
| ZOFFSET_APPLY      | Re-apply saved Z-offset                    |

### Calibration

| Command                                   | Purpose                        |
|-------------------------------------------|--------------------------------|
| G28                                       | Home all axes                  |
| BED_MESH_CALIBRATE                        | Probe and generate bed mesh    |
| BED_MESH_PROFILE SAVE=default             | Save as default profile        |
| INPUTSHAPER                               | Calibrate input shaper X and Y |
| SKEW_APPLY_AND_SAVE AC=... BD=... AD=...  | Calibrate skew from lengths    |

### Filament

| Command                          | Purpose              |
|----------------------------------|----------------------|
| LOAD_MATERIAL                    | Load at 240 C        |
| LOAD_MATERIAL EXTRUDER_TEMP=210  | Load at 210 C        |
| QUIT_MATERIAL                    | Unload at 240 C      |
| M600                             | Manual filament swap |

### Diagnostics (SSH)

| Command                                                       | Purpose                     |
|---------------------------------------------------------------|-----------------------------|
| tail -n 60 /usr/data/printer_data/logs/klippy.log             | Recent Klipper activity     |
| grep -E "Loaded MCU\|Timeout on connect" /usr/.../klippy.log  | MCU handshake history       |
| grep "Stats " /usr/data/printer_data/logs/klippy.log \| tail -2 | Latest health snapshot    |
| grep -rn "^\[respond\]" /usr/data/printer_data/config/        | Find duplicate sections     |
| grep "TRSYNC_TIMEOUT =" /usr/share/klipper/klippy/mcu.py      | Verify timeout modification |
| find / -name "mcu.py" -path "*klippy*" 2>/dev/null            | Locate Klipper source       |
| ls /usr/data/printer_data/config/Helper-Script/*.disabled     | List disabled config files  |

### Backups (SSH)

| Command                                          | Purpose                   |
|--------------------------------------------------|---------------------------|
| cp printer.cfg printer.cfg.bak                   | Back up before edits      |
| cd /usr/data/printer_data/config && git log      | View git-backup history   |

---

## Appendix A — Upload File Manifest

| Filename                  | Fixes Applied                | Last Updated |
|---------------------------|------------------------------|--------------|
| printer.cfg               | #4                           | 2026-09-27   |
| gcode_macro.cfg           | #5, #6, #11, #12, #13        | 2026-09-27   |
| sensorless.cfg            | #2                           | 2026-09-27   |
| printer_params.cfg        | #17                          | 2026-09-27   |
| useful-macros.cfg         | #3                           | 2026-09-27   |
| M600-support-custom.cfg   | Global [respond] removed     | 2026-09-27   |
| guppy_cmd.cfg             | Global [respond] removed     | 2026-09-27   |
| moonraker.conf            | Comments added               | 2026-09-27   |
| mcu.py (external)         | TRSYNC_TIMEOUT = 0.05        | 2026-09-27   |

---

## Appendix B — Console Error Message Decoder

| Error code / message                          | Meaning / Common cause                    |
|-----------------------------------------------|-------------------------------------------|
| key1                                          | Duplicate section, or serial handshake    |
| key60                                         | Internal error on command; often mesh     |
| key167                                        | G-code rename type mismatch               |
| MCU 'mcu' shutdown: Move queue overflow       | Host sent commands too fast               |
| Unknown skew profile: default                 | Skew not saved, or fix #12 not applied    |
| Timeout on connect                            | MCU did not respond to identify in time   |
| Heater extruder not heating at expected rate  | verify_heater check failed                |
| gcode_macro X is not defined in config        | Missing [include], or duplicate section   |
| TypeError: initializer for ctype 'struct      | Downstream symptom of failed identify     |
|   serialqueue *' must be a cdata pointer      | (see Timeout on connect)                  |

---

## Appendix C — Section Ownership Cheat Sheet

Every section in the loaded config belongs to exactly one file. If you ever
see a duplicate-section error, this is the list to check against.

| Section                              | Owned by                |
|--------------------------------------|-------------------------|
| [respond]                            | printer.cfg             |
| [save_variables]                     | zoffset-guppy.cfg       |
| [idle_timeout]                       | M600-support-custom.cfg |
| [filament_switch_sensor filament_]   | M600-support-custom.cfg |
| [pause_resume]                       | printer.cfg             |
| [bed_mesh]                           | printer.cfg             |
| [gcode_macro START_PRINT]            | gcode_macro.cfg         |
| [gcode_macro PAUSE]                  | gcode_macro.cfg         |
| [gcode_macro RESUME]                 | M600-support-custom.cfg |
| [gcode_macro M600]                   | M600-support-custom.cfg |
| [gcode_macro SET_GCODE_OFFSET]       | zoffset-guppy.cfg       |
| [gcode_macro BED_MESH_CALIBRATE]     | gcode_macro.cfg         |
| [homing_override]                    | sensorless.cfg          |

If any of these appear twice in:

    grep -rn "^\[section\]" /usr/data/printer_data/config/

...that's your shutdown cause.

---

## Appendix D — Health Snapshot Reference

Copy-paste this table when comparing a healthy printer to a suspect one.

| Indicator       | Healthy value      | Sick value (action)                    |
|-----------------|--------------------|----------------------------------------|
| stalled_bytes   | 0                  | >0 → check slicer settings (§7.2)      |
| ready_bytes     | 0                  | climbing → check cable or EMI          |
| bytes_invalid   | 0                  | >0 → check cable, power supply         |
| bytes_retransmit| 9 (frozen)         | climbing → check cable, EMI, ground    |
| mcu_awake       | ≤ 0.005            | > 0.05 → MCU overloaded or stuck       |
| mcu_task_avg    | ≤ 0.00002          | > 0.00005 → MCU busy or slow           |
| sysload         | < 1.0              | > 2.0 sustained → host overloaded      |
| memavail        | stable, > 100 MB   | falling → memory leak                  |
| print_stall     | 0                  | >0 during print → host can't keep up   |
| rto             | 0.025 at 230400    | n/a — this is a constant, not a signal |

---

**End of Reference Document**

Keep this file at /usr/data/printer_data/config/K1C_REFERENCE.md or similar.
When something breaks six months from now, start here — it will tell you
what was changed, why, and how to verify or roll back.

---

*Generated: 2026-09-27*
*Printer: K1C*
*Klipper config version: v2.0.9-corrections*