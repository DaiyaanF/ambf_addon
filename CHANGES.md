# AMBF Blender Addon — Blender 5.x Changes

This document describes all new features and fixes added to the AMBF Blender addon for Blender 5.x compatibility. It is written so that someone new to the project can understand what was added and why.

---

## Background

The AMBF Blender addon (`ambf_addon.py`) is a tool that lets you build robot simulation scenes inside Blender and export them as ADF (AMBF Description Format) YAML files. These files are loaded by the AMBF simulator to run physics-based robotics simulations.

The original addon supported rigid bodies and joints. This update adds **camera objects**, **light objects**, a **simulator launch button**, and fixes several bugs for Blender 5.x.

---

## 1. Camera Object

### What it does
Users can now create an AMBF camera directly inside Blender, configure its simulation properties, and have those properties saved to and loaded from ADF files.

### How to use it
1. In the AMBF panel, click **Create Camera**. This adds a Blender camera object tagged as an AMBF camera.
2. Select the camera and open the **AMBF Camera** sub-panel to configure its properties.
3. Save your scene as an ADF file — the camera and all its properties are included automatically.

### Properties available

| Property | Description |
|---|---|
| **Near Plane** | The closest distance the camera can see (clipping start). Syncs live with Blender's camera. |
| **Far Plane** | The furthest distance the camera can see (clipping end). Syncs live with Blender's camera. |
| **Monitor Number** | Which monitor the simulator renders this camera's view to. |
| **Publish Image** | Toggle to publish the camera image over ROS. |
| **Publish Image Interval** | How often (in frames) the image is published. |
| **Publish Image Resolution** | Width and height of the published image in pixels. |
| **Publish Depth** | Toggle to publish a depth image over ROS. |
| **Publish Depth Interval** | How often (in frames) the depth image is published. |
| **Publish Depth Resolution** | Width and height of the published depth image in pixels. |

### Optional: Camera Intrinsics
Enable this to define the camera's optical properties manually. Useful for matching a real camera.

| Parameter | Description |
|---|---|
| `fx`, `fy` | Focal lengths in pixels |
| `cx`, `cy` | Principal point (optical center offset) |
| `s` | Skew coefficient (usually 0) |

### Optional: Projection Matrix
Enable this to define a custom 4×4 projection matrix. This overrides the intrinsics and gives full control over how the scene is projected onto the image.

---

## 2. Light Object

### What it does
Users can now create an AMBF light (spotlight) inside Blender, configure its simulation properties, and have them saved to and loaded from ADF files.

### How to use it
1. In the AMBF panel, click **Create Light**. This adds a Blender spotlight tagged as an AMBF light.
2. Select the light and open the **AMBF Light** sub-panel to configure its properties.
3. Save your scene as an ADF file — the light and all its properties are included automatically.

### Properties available

| Property | Description |
|---|---|
| **Type** | Always Spotlight (the only light type supported by AMBF). Displayed as a read-only label. |
| **Direction** | The direction vector the light points toward. |
| **Spot Exponent** | Controls how concentrated the beam is. Higher = sharper falloff at the edges. |
| **Cutoff Angle** | The half-angle of the spotlight cone in radians. |
| **Shadow Quality** | Controls the resolution/quality of shadows cast by this light. |
| **Parent Body** | Optional. Name of a rigid body this light is attached to. It will move with that body. |

### Attenuation
Controls how the light intensity fades with distance. Three values can be adjusted:

| Parameter | Description |
|---|---|
| **Constant** | Base intensity that does not fade with distance (default: 1.0) |
| **Linear** | Intensity fades linearly with distance |
| **Quadratic** | Intensity fades with the square of distance (most physically realistic) |

---

## 3. AMBF Simulator Launch Button

### What it does
Adds a **Launch in AMBF Simulator** button to the Blender panel so you can open your scene in the simulator without switching to a terminal.

### How to use it
1. In the AMBF panel, scroll to the **Launch in AMBF Simulator** section.
2. Set the **ADF to Launch** path to your saved ADF file. This field auto-fills when you save an ADF from Blender.
3. Set the **Simulator Path** to the location of your `ambf_simulator` executable (e.g. `~/ros_ambf_ws/install/AMBF/bin/ambf_simulator`).
4. Click **Launch in AMBF Simulator**. The simulator opens with your scene loaded.

### Auto-populate
When you save an ADF file using the **Save ADF** button, the ADF path is automatically copied into the **ADF to Launch** field so you can launch immediately without extra steps.

---

## 4. Bug Fix: Mesh Path Resolution When Loading ADF Files

### The problem
When loading an ADF robot model, Blender would throw a `FileNotFoundError` like:

```
FileNotFoundError: No such file or directory: '/home/user/blender-5.0.1-linux-x64/high_res/Chassis.STL'
```

Relative mesh paths (like `high_res/Chassis.STL`) were being resolved against Blender's install directory instead of the folder containing the ADF file.

### The fix
The loaded ADF file's path is now stored before any path resolution happens. All relative mesh paths are resolved relative to the ADF file's own directory, which is the correct and expected behavior.

---

## 5. Blender 5.x Mesh Import/Export Compatibility

### The problem
Blender 4.0 replaced all legacy mesh import/export operators with new ones under `bpy.ops.wm`. The old operators (`import_mesh.stl`, `export_scene.obj`, etc.) no longer exist in Blender 5.x, causing errors when saving or loading meshes.

### The fix
All import and export calls now try the new Blender 5.x operator first, and fall back to the legacy operator if running on an older version. This makes the addon work on both Blender 4.x and 5.x.

| Format | Old Operator (pre-4.0) | New Operator (4.0+) |
|---|---|---|
| STL import | `import_mesh.stl` | `wm.stl_import` |
| STL export | `export_mesh.stl` | `wm.stl_export` |
| OBJ import | `import_scene.obj` | `wm.obj_import` |
| OBJ export | `export_scene.obj` | `wm.obj_export` |
| PLY import | `import_mesh.ply` | `wm.ply_import` |
| PLY export | `export_mesh.ply` | `wm.ply_export` |
| DAE (Collada) | `wm.collada_import` | `wm.collada_import` (unchanged) |

> **Note:** 3DS format (`.3ds`) was removed from Blender 4.0 onward and is no longer supported by this addon.

---

## ADF Save/Load Roundtrip

All new objects (cameras and lights) are fully integrated into the ADF save/load system:

- **Saving**: When you click Save ADF, camera and light objects in the scene are automatically detected and written into the ADF YAML file under `cameras:` and `lights:` sections.
- **Loading**: When you load an existing ADF file, any cameras and lights defined in it are reconstructed in Blender with all their properties restored.

---

## 6. Bug Fix: Launch Button Used Wrong Command-Line Flag

### The problem
Clicking **Launch in AMBF Simulator** silently ignored your ADF file and opened the simulator's default demo scene (a toy car) instead.

### The cause
`AMBF_OT_launch_simulator` called the simulator with `--a <path>` (double dash). AMBF's actual flag for loading a multibody file directly is `-a` (single dash) — `--a` isn't a recognized option, so it was silently ignored and the simulator fell back to its default `launch.yaml`.

### The fix
Changed the subprocess call to use `-a` instead of `--a`.

---

## 7. Bug Fix: Sensor's Parent Was Never Exported

### The problem
Setting a Sensor's **Parent** field in Blender had no effect — the exported ADF always wrote `parent: ''` for sensors, so a sensor never moved with the body it was supposed to be attached to (Actuators worked correctly; only Sensors were affected).

### The cause
In `generate_sensor_data_from_ambf_sensor`, the line that writes the parent field was commented out (`# TODO: Fix parent not set issue`) — the parent was computed but never actually assigned to the ADF data.

### The fix
Uncommented and implemented the assignment, mirroring the working pattern already used for Actuators: resolve the parent object's name (stripping any namespace prefix) and write it as `parent: BODY <name>`.

---

## 8. Bug Fix: Crash When Loading a Ghost Object's Collision Shape From an ADF

### The problem
Loading an ADF file containing a Ghost Object with a `collision shape` field crashed with:
```
AttributeError: 'Object' object has no attribute 'ambf_collision_shape'
```

### The cause
`load_ambf_ghost_body` tried to set `obj_handle.ambf_collision_shape` directly — but `ambf_collision_shape` was never registered as a property on the Blender `Object` type. It only exists inside `AMBF_PG_CollisionShapePropGroup`, accessed through `obj_handle.ambf_collision_shape_prop_collection`.

### The fix
Changed the assignment to add an entry to `ambf_collision_shape_prop_collection` (if none exists) and set `ambf_collision_shape` on that entry instead, matching the pattern already used everywhere else in the file (e.g. rigid body loading).

---

## Important: How This Add-on Is Actually Loaded By Blender

Blender does **not** run this repo file directly — it runs whatever is installed at:
```
~/.config/blender/<version>/scripts/addons/ambf_addon.py
```

**As of this update, that installed path is a symlink pointing to this repo's `ambf_addon.py`.** This means:
- Editing `ambf_addon.py` in this repo *is* editing what Blender runs — no manual copying needed.
- You still need to reload the add-on for Blender to pick up a change: either restart Blender, or `Edit → Preferences → Add-ons` → disable then re-enable the AMBF add-on.
- If a fresh Blender install or a different machine ever shows this add-on **not** picking up repo changes, check whether `~/.config/blender/<version>/scripts/addons/ambf_addon.py` is still a symlink (`ls -la` on it) — if it's a real file again, someone (an installer, a Blender update, a manual "Install from Disk") likely overwrote the symlink with a fresh copy, silently reintroducing this whole class of bug. Re-create the symlink and re-diff against the repo file if that happens.
- The previous stale, disconnected copy (dated June 23, missing all of the fixes in sections 6-8 above and possibly others) was preserved at `ambf_addon.py.stale_backup` in that same addons folder rather than deleted, in case anything unexpected referenced it.

---

## File Modified

All changes are contained in a single file:

- **`ambf_addon.py`** — the main Blender addon script
