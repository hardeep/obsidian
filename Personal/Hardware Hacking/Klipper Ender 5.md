Perfect — here’s a **clean, repeatable guide** you can keep and reuse. This matches *exactly* what worked for you and avoids the common Klipper Z-offset traps.

## Klipper Z Offset Calibration (CR Touch / BLTouch)

**Ender 5–style workflow with final micro-adjustments**

This method uses **`PROBE_CALIBRATE` for the coarse offset** and then **applies precise micro-adjustments correctly** so they persist across reboots.

---

## Prerequisites

* X/Y homing works correctly
* Probe deploys and triggers reliably
* `[stepper_z]` uses:

```ini
  endstop_pin: probe:z_virtual_endstop
  position_min: -5
```

* `[safe_z_home]` is configured

---

## Part 1 — Coarse calibration (probe-based)

### 1️⃣ Home everything

```gcode
G28
```

### 2️⃣ Start probe calibration

```gcode
PROBE_CALIBRATE
```

Klipper will:

* Probe the bed
* Move the nozzle to the calibration position
* Enter manual Z adjustment mode

### 3️⃣ Paper test (coarse)

Adjust using:

```gcode
TESTZ Z=-0.1
```

Then fine tune:

```gcode
TESTZ Z=-0.01
```

**Stop when:**

* Paper moves with *firm drag*
* No force required
* Paper does not lock up

### 4️⃣ Accept and save

```gcode
ACCEPT
SAVE_CONFIG
```

Klipper restarts and writes a `z_offset` to `[bltouch]`.

At this point you are **close, but not perfect** — this is expected.

---

## Part 2 — Fine micro-adjustment (the critical part)

This fixes the common situation where:

* `PROBE_CALIBRATE` feels right
* but `G1 Z0` is still slightly off

### 5️⃣ Home and go to Z=0

```gcode
G28
G1 Z0
```

Check paper feel **now** (this is the real reference).

---

### 6️⃣ Apply a micro adjustment

If the nozzle is **too tight**, raise Z:

```gcode
SET_GCODE_OFFSET Z=0.46 MOVE=1
```

(Use *your* required value — 0.46 is just an example.)

This is **temporary** until applied.

---

### 7️⃣ Commit the adjustment to the probe

This is the key step most guides miss:

```gcode
Z_OFFSET_APPLY_PROBE
```

This command:

* Transfers the live Z offset
* Into `[bltouch] z_offset`
* So you don’t need to keep babystepping

---

### 8️⃣ Save permanently

```gcode
SAVE_CONFIG
```

Klipper restarts and writes the updated value.

---

## Verification (important)

After reboot:

```gcode
G28
G1 Z0
```

Paper should now:

* Move with firm drag
* Not require force
* Allow tiny ±0.01 adjustments without locking

If needed, do a **final trim**:

```gcode
SET_GCODE_OFFSET Z=0.01 MOVE=1
Z_OFFSET_APPLY_PROBE
SAVE_CONFIG
```

---

## Why this workflow works (conceptually)

* `PROBE_CALIBRATE` gets you **95% there**
* Real Z=0 behavior is slightly different than calibration mode
* `SET_GCODE_OFFSET` alone is temporary
* `Z_OFFSET_APPLY_PROBE` is what *actually* updates the stored probe offset

This avoids:

* “It keeps resetting”
* Huge offsets like `+0.5` every boot
* Fighting `position_min` clamps
* Endless re-running of `PROBE_CALIBRATE`

---

## TL;DR (the essential commands)

```gcode
PROBE_CALIBRATE
ACCEPT
SAVE_CONFIG

G28
G1 Z0
SET_GCODE_OFFSET Z=0.46 MOVE=1
Z_OFFSET_APPLY_PROBE
SAVE_CONFIG
```
```gcode
G28
M140 S60
M190 S60
M104 S200
M109 S200
BED_MESH_CALIBRATE
```
