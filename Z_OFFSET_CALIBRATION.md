# K1C Z-Offset & Bed Mesh Calibration Workflow

**Printer:** Creality K1C
**Stack:** GuppyScreen + OrcaSlicer + Klipper + PRTouch
**Last updated:** 2026-09-27

This printer uses **two separate Z offsets** that must NOT be confused.

---

## The Two Offsets

### 1. PROBE Z-Offset — `[prtouch_v2] z_offset`

The physical distance between the probe trigger point and the nozzle tip.

- **Calibrated:** Once with `PROBE_CALIBRATE`
- **Stored:** In the `SAVE_CONFIG` block at the bottom of `printer.cfg`
- **Do NOT touch** for first-layer tuning — it affects the entire
  coordinate system
- Changing this is rare: only after nozzle replacement, probe service, or
  major mechanical work

### 2. GCODE Z-Offset — `Helper-Script/variables.cfg`

A fine-tuning offset applied **on top of** the probe offset.

- **Persisted by:** `zoffset-guppy.cfg`
- **Adjustable:** Live during a print from GuppyScreen
- **This is what you tune** for first-layer perfection, per filament and
  per build plate

### Golden Rule

> Calibrate the **PROBE Z-offset** first (rarely, one time only).
> Tune the **GCODE Z-offset** for every filament / build plate combo.

---

## Step 1 — Calibrate PROBE Z-Offset

*One time, or after nozzle/probe service.*

1. Home the printer: