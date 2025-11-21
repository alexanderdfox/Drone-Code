# G-code+ (G+) Specification

## Version 1.0

---

## Overview
G-code+ (G+) is an **extension to standard G-code** designed to introduce multitasking, metadata, sensor feedback, CPU-cycle-based timing, and higher-order motion primitives while remaining **backwards compatible**. Legacy firmware ignores unknown commands and comments.

G+ aims to improve safety, precision, and programmability of CNC, 3D printing, laser engraving, and other toolpath-based machines.

---

## Design Principles
1. **Backwards compatible:** Unknown commands are treated as comments.
2. **Small-step adoption:** Can be implemented via preprocessor, host plugin, or firmware module.
3. **Readable:** Human-friendly syntax mapping to existing G-code primitives.
4. **Deterministic:** Optional cycle-accurate timing on supported hardware.
5. **Safe by default:** Emergency stop triggers and conflict resolution policies.

---

## 1. Metadata Block
File-level metadata to provide slicer, host, or firmware context.

```
;G+META {
;  "material": "PETG",
;  "nozzle_temp": 245,
;  "bed_temp": 85,
;  "author": "A. Fox",
;  "uuid": "123e4567-e89b-12d3-a456-426614174000"
;}
```

Legacy firmware ignores this; G+ preprocessors can extract and apply parameters.

---

## 2. Tasks & Parallel Execution
Define tasks for concurrent execution with optional priority and background flags.

```
;G+TASK START id=print_body priority=5
G1 X100 Y100 F3000
G1 X150 Y100 F3000
;G+TASK END id=print_body

;G+TASK START id=camera_monitor priority=1 background=true
M200 CAMERA ON
;G+TASK END id=camera_monitor
```

**Parameters:**
- `id`: Unique task identifier.
- `priority`: Integer to indicate scheduler order.
- `background=true`: Marks task as non-blocking.

**Implementation:**
- Preprocessor: Interleaves tasks or splits into files.
- Firmware: Native scheduler handles concurrency and conflicts.

---

## 3. Sensor & Conditional Control
Read sensors into variables and create conditional execution.

```
;G+SENSE READ sensor=force_front -> $force_front
;G+IF $force_front > 8 THEN
;  M112 ; emergency stop
;G+ENDIF
```

**Variables:** `$name` stored at runtime.
**Conditional evaluation:** Runtime or preprocessor.

---

## 4. CPU-Cycle & Deterministic Timing
For cycle-accurate operations on supported hardware.

```
;G+CYCLES WAIT cycles=500000
;G+CYCLES SYNC_MOVE X10 Y0 F2000 cycles=1000000
```

**Fallback:** Convert to approximate millisecond delay on unsupported firmware.

---

## 5. Advanced Motion Primitives
Support higher-order motion like Bezier or NURBS curves.

```
;G+BEZIER X0 Y0 X30 Y60 X60 Y0 SAMPLES=48
```

Preprocessors tessellate into linear `G1` segments. Firmware can implement native curve handling.

---

## 6. Parameter / Object Blocks
Define reusable toolpaths or objects.

```
;G+OBJECT name=gear_module_02
;  G+BEZIER ... ; path definition
;G+OBJECT END

;G+CALL OBJECT=gear_module_02 X=40 Y=20 ROT=90
```

**Semantics:** Instantiates toolpath at specified position/rotation.

---

## 7. Safety & Conflict Policy
Define handling for task conflicts and sensor failures.

```
;G+SAFETY {
;  "conflict": "preempt",
;  "on_sensor_fail": "pause"
;}
```

**Options:**
- `conflict`: `preempt` or `queue`
- `on_sensor_fail`: `pause`, `stop`, or `ignore`

---

## 8. Example Combined Usage
```
;G+META {"material":"PLA","nozzle_temp":210}

;G+TASK START id=camera_mon background=true
;G+SENSE READ sensor=camera_health -> $cam_ok
;G+IF $cam_ok == 0 THEN
;  M25 ; pause print
;G+ENDIF
;G+TASK END id=camera_mon

;G+TASK START id=print_shell priority=10
G28
M104 S210
M140 S60
G1 Z0.2 F1200
;G+BEZIER X0 Y0 X50 Y50 X100 Y0 SAMPLES=36
;G+TASK WAIT id=camera_mon ; optional join
;G+TASK END id=print_shell
```

---

## 9. Implementation Levels
1. **Preprocessor:** Parses G+ comments, tessellates curves, splits tasks, outputs legacy G-code.
2. **Host runtime + plugin:** Manages concurrency and sensor callbacks on host PC.
3. **Firmware module:** Native scheduler, variable store, sensor reads, cycle timers.

---

## 10. Migration & Ecosystem
- Slicer support: Export G+ blocks.
- Host UI: Visualize tasks and sensors.
- Conversion service: Flatten G+ to standard G-code for legacy printers.
- Test suite: Validate conflict, emergency stop, and sensor handling.

---

## 11. Notes on Future Directions
- **G-code 3.0:** Structured binary or JSON format for full-featured toolpaths.
- **G-Script:** High-level programming language compiling to G+ or G-code, enabling loops, variables, and conditional logic.

---

*End of G-code+ (G+) Specification v1.0*

