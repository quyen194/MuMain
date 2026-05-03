# Camera System

This document covers the camera architecture introduced by PR
[#335](https://github.com/sven-n/MuMain/pull/335) ("3d camera rework") - what
the cameras are, how to switch between them, what they expose to tuning, and
the gameplay-visible changes that came with the rework.

For the in-editor tuning UI, see [`dev-editor.md`](dev-editor.md). For the
options-window and windowing changes that landed alongside, see
[`options-window.md`](options-window.md).

---

## 1. Modes at a glance

| Mode | Where | How to activate | Notes |
|------|-------|-----------------|-------|
| **Default** | All gameplay scenes, login, character select | Active by default | Original third-person follow camera, refactored. |
| **Orbital** | `MainScene` only | Press **F9** to cycle | Spherical orbit around the hero with middle-mouse drag and wheel zoom. |
| **FreeFly** | Editor only | DevEditor → Scenes → camera mode | Spectator camera with WASD/QE + right-mouse look. Not built in Release. |

**F9 cycles cameras.** Outside MainScene, only the Default camera is available
(handler in `src/source/Camera/CameraUtility.cpp:23`). FreeFly is gated by
`#ifdef _EDITOR` and is unreachable in Release builds.

> **Heads-up:** The old in-game **F8 free-fly** has been removed. FreeFly is
> now editor-only (`src/source/Camera/FreeFlyCamera.h`).

---

## 2. Architecture

All camera code lives under `src/source/Camera/`.

### `ICamera` (`ICamera.h`)
Interface every camera implements. The contract:

- Lifecycle: `OnActivate()`, `OnDeactivate()`, `Reset()`, `Update()`.
- Identity: `GetName()`.
- Config + culling (added in this rework): `GetConfig()`, `SetConfig()`,
  `GetFrustum()`, `ShouldCullObject()`, `ShouldCullTerrain()`,
  `ShouldCullObject2D()`.

Rendering code asks the active camera through these methods instead of taking
an `ICamera*` parameter - see "What changed for renderers" below.

### `CameraManager` (`CameraManager.h` / `.cpp`)
Singleton that owns one instance of each concrete camera and exposes:

- `Initialize()`, `Update()` - wire up and tick the active camera each frame.
- `SetCameraMode(mode)`, `CycleToNextMode()` - switch cameras (also forces
  cleanup of the outgoing camera).
- "Spectated camera" tracking (editor only): when FreeFly is active, the
  manager keeps updating the gameplay camera in the background so you can fly
  around the scene without losing the player camera state.

Orbital is **scene-locked**: leaving `MainScene` automatically reverts to
Default (`CameraManager.cpp:74-75`).

### `DefaultCamera` (`DefaultCamera.h` / `.cpp`)
The third-person follow camera, refactored. Highlights:

- Hero-relative positioning with smooth mount-offset lerping
  (`m_CurrentMountOffset`).
- **Discrete zoom ladder** driven by the global `g_shCameraLevel` (0..5):

  | Level | Distance |
  |-------|----------|
  | 0 | 1300 |
  | 1 | 1400 |
  | 2 | 1500 |
  | 3 | 1600 |
  | 4 | 1700 |
  | 5 | Cutscene-driven (`g_Direction.m_fCameraViewFar`) |

  Constants `CAMERA_DISTANCE_LEVEL_BASE = 1300.f` and
  `CAMERA_DISTANCE_LEVEL_STEP = 100.f` (`DefaultCamera.cpp:56-57`).
- ViewFar scales with zoom level (`baseFarPlane × {1.0, 1.04, 1.08, 1.23,
  1.33}` for levels 0..4, `DefaultCamera.cpp:418-423`).
- Scene-aware reset via `ResetForScene(scene)` and the `m_LastSceneFlag`
  guard for activation transitions.

### `OrbitalCamera` (`OrbitalCamera.h` / `.cpp`)
Spherical orbit around the hero. MainScene-only.

- Continuous wheel-zoom radius in **[200, 2000]** with `DEFAULT_RADIUS = 1100`
  (`OrbitalCamera.h:98-100`). No discrete ladder - every wheel tick moves the
  camera.
- Middle-mouse drag rotates yaw/pitch (`m_DeltaYaw`, `m_DeltaPitch`); WASD
  movement is not used.
- Pitches and offsets shift slightly when zooming closer than the default
  radius so the hero stays framed (see `OrbitalCamera.cpp:625` ff).

### `FreeFlyCamera` (`FreeFlyCamera.h` / `.cpp`)  - _EDITOR only_
- WASD/QE movement, Shift to sprint (`SPRINT_MULTIPLIER = 3`), right-mouse to
  look around.
- Pitch clamped to `[-160°, -20°]`.
- Saves and restores its previous state on toggle so re-entering FreeFly puts
  you back where you left off.

---

## 3. Per-camera config

`CameraConfig` (`CameraConfig.h:59-235`) is a struct describing how a camera
projects and culls. Fields:

| Field | What it controls |
|-------|------------------|
| `hFov` | Horizontal field of view in degrees. Vertical FOV is derived. |
| `nearPlane` | Near clip plane (units). |
| `farPlane` | Far clip plane / 3D object cull range. |
| `terrainCullRange` | 2D terrain tile cull range. Defaults to `farPlane × 1.4`. |
| `objectCullRange` | 3D object cull radius (per-object sphere test). |
| `fogStart`, `fogEnd` | Fog density ramp (typically `farPlane × 1.0` to `× 1.25`). |
| `aspectRatio` | Live-calculated by `BeginOpengl()`; the field is informational. |

`CameraConfig` ships **scene-specific factory methods**:

- `ForMainSceneDefaultCamera()` - gameplay default (`hFov = 40`, `farPlane =
  3000`).
- `ForMainSceneOrbitalCamera()` - orbital in gameplay (`hFov = 90`, `farPlane
  = 3800`).
- `ForCharacterScene()` - character select.
- `ForLoginScene()` - login scene with a deliberately huge far plane.

`CameraState` (`CameraState.h`) is the global `g_Camera` struct that holds
*live* camera data (`Position`, `Angle`, `Matrix`, `ViewNear`/`ViewFar`,
`FOV`, `Distance`/`DistanceTarget`, `ZoomLevel`, cached `PerspectiveX/Y`
factors). The active camera writes to `g_Camera` each frame; rendering code
reads from it.

### What persists?

Only **orbital zoom radius** is persisted. It round-trips through
`GameConfig::GetZoom()` / `SetZoom()` (loaded from `[Camera]` in `config.ini`,
saved on wheel input and on scene transitions - `OrbitalCamera.cpp:59, 87,
353`). Everything else (FOV, fog, trapezoid widths, …) resets to the
scene-specific factory config on each launch.

DevEditor overrides are **runtime-only** - see [`dev-editor.md`](dev-editor.md).

---

## 4. Frustum culling

Files under `src/source/Camera/`:

- **`Frustum.h` / `.cpp`** - unified 3D frustum (6 planes, 8 corner verts) +
  2D ground projection. Build it once per frame with `BuildFromCamera()`,
  then call `TestSphere()` for objects and `TestPoint2D()` for terrain tiles.
  `SetCustom2DHull()` lets the DevEditor inject a tuned trapezoid for the
  Default camera's 2D cull region.
- **`ConvexHull2D.h`** - header-only utilities (`Point2D`, `SortPoints2D`,
  `ConvexHullCCW`, `Cross2D`). Previously duplicated inline in
  `ZzzLodTerrain.cpp`; now a single shared definition.
- **`FrustumRenderer.h` / `.cpp`** _(editor only)_ - wireframe + translucent
  frustum visualisation (green near plane, red far plane, yellow side planes,
  cyan camera marker, ground projection).
- **`CameraProjection.h` / `.cpp`** - `SetupPerspective()` (replacement for
  the old `gluPerspective2`), `ScreenToWorldRay()` for picking,
  `WorldToScreen()`, `TransformPosition()`, `TestDepthBuffer()`,
  `GetOpenGLMatrix()`. Caches the perspective factors on `CameraState` so we
  don't recompute trig per-projected-point.

`CullingConstants.h` (top-level under `src/source/`) defines the unified
fallback radii used when DevEditor isn't overriding them:

```cpp
constexpr float DEFAULT_CULL_RADIUS_ITEM   = 400.0f;  // legacy TestFrustrum value
constexpr float DEFAULT_CULL_RADIUS_OBJECT = 100.0f;
```

A 1.4× multiplier (`RENDER_DISTANCE_MULTIPLIER`) is applied to terrain
culling so terrain extends slightly beyond the visible frustum (`CameraConfig.h:32`).

### What changed for renderers

- `RenderTerrain()` and `RenderObjects()` no longer take an `ICamera*`. They
  read `g_Camera` directly and ask the active camera for cull decisions
  through `CameraManager`. Signatures: `void RenderTerrain(bool EditFlag)`
  (`ZzzLodTerrain.cpp:3242`) and `void RenderObjects()`
  (`ZzzObject.cpp:3274`).
- The static `s_pCachedCamera` pointer in `ZzzLodTerrain.cpp` is gone.
- `Point2D` / `SortPoints2D` / `ConvexHullCCW` are no longer duplicated in
  `ZzzLodTerrain.cpp`; the canonical copy is in
  `Camera/ConvexHull2D.h`.

---

## 5. `$details` overlay and `FrameProfiler`

`FrameProfiler` (`src/source/Utilities/FrameProfiler.h`) is a small
header-only timer with a `Pass` enum (`Terrain`, `Objects`, `Characters`,
`Items`, `Effects`, `Other`) and a RAII `Scope` that accumulates nanoseconds
into per-pass counters. `ResetFrame()` is called once per frame.

The `$details` chat command (`muConsoleDebug.cpp:95-100`) toggles
`SetShowDebugInfo()`. When enabled, `SceneManager::RenderDebugInfo()`
(`SceneManager.cpp:495+`) draws:

- A line with per-pass timings - `Frame ms  T:XX  O:XX  C:XX  I:XX  E:XX`
  (Terrain / Objects / Characters / Items / Effects).
- The active camera mode and basic scene-visibility stats.
- A 300-frame frame-time graph (5 s window).

`$details on` and `$fpscounter on` are mutually exclusive - enabling one
disables the other.

Other relevant chat commands (`muConsoleDebug.cpp`):

- `$fps <value>` - set FPS limit.
- `$vsync on|off` - toggle V-Sync.
- `$fpscounter on|off` - toggle the simple FPS counter.

---

## 6. Gameplay behaviour changes - what players will notice

These are the **user-visible** changes from the rework. List them in any
release notes / patch notes that go out.

- **Per-map camera overrides removed.** Castle Siege, PK Field, DoppelGanger
  1/2, the 6th-character home, and map 5 (idle wobble) used to have hardcoded
  camera tweaks (forced distance/zoom level, clamped Z, ViewFar multipliers,
  idle angle wobble). All gameplay maps now share the same default-camera
  zoom ladder (1300 / 1400 / 1500 / 1600 / 1700) and follow hero Z normally.
  Tour mode, cutscene "direction" mode, the per-tile `TW_HEIGHT` snap, the
  Chaos Castle / Kanturu 3rd-action animation guard, and Dinorant mount
  hover heights are **kept** - those are scene/cutscene/animation behaviour,
  not per-map camera overrides. (Source: commit `889eb9f0`.)
- **F8 in-game free-fly is gone.** FreeFly is editor-only (commit
  `f9ca71a7`). In Release builds it does not exist.
- **Press F9 to switch between Default and Orbital** in MainScene.
- **Orbital wheel-zoom now persists** across sessions via `config.ini`
  (`[Camera] Zoom`).
- **`$details` HUD** can be enabled in chat to see per-pass frame timings and
  the active camera mode.

---

## 7. File map

```
src/source/Camera/
  ICamera.h
  CameraManager.h / .cpp        # singleton, mode switching, spectator state
  CameraMode.h                  # enum + GetNextCameraMode()
  CameraConfig.h                # config struct + scene factory methods
  CameraState.h / .cpp          # global g_Camera live state
  CameraUtility.h / .cpp        # F9 handler, math helpers
  CameraDebugLog.h
  DefaultCamera.h / .cpp        # third-person follow
  OrbitalCamera.h / .cpp        # MainScene orbital
  FreeFlyCamera.h / .cpp        # editor-only
  Frustum.h / .cpp              # 3D + 2D culling
  FrustumRenderer.h / .cpp      # editor wireframe debug
  ConvexHull2D.h                # 2D hull utilities
  CameraProjection.h / .cpp     # perspective + picking + world↔screen
src/source/CullingConstants.h   # fallback cull radii
src/source/CameraMove.h / .cpp  # login-scene defaults shared with DevEditor
src/source/Utilities/FrameProfiler.h
```
