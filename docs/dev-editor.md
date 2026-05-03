# DevEditor

The DevEditor is an in-game tuning UI for camera, scene, and rendering
parameters. It was added as part of PR
[#335](https://github.com/sven-n/MuMain/pull/335).

For the camera architecture itself, see [`camera-system.md`](camera-system.md).

---

## 1. Who it's for

Developers and modders. The DevEditor lets you live-tune camera FOV, far
plane, fog, 2D culling trapezoids, terrain/object render distances, and a
suite of debug visualisations - without rebuilding.

It is **debug-only**:

- Built only when `_EDITOR` is defined (see [Build Configurations in
  `README.md`](../README.md#build-configurations)).
- Requires the ImGui submodule (`git submodule update --init`).
- Not present in Release builds.

---

## 2. Opening it

Two ways:

- **Press F12** in-game.
- Start the client with `--editor` to launch with the editor enabled (or
  toggle later with F12).

There is also an **on-screen toggle button** in the top-right of the window.

The editor is rendered with ImGui and consumes input over the game
(see `MuInputBlockerCore.cpp` for the input-routing logic).

---

## 3. Tabs

### 3.1 Scenes tab

#### Camera Mode controls
Switch between FreeFly and the active gameplay camera (Default / Orbital).
While FreeFly is active, the gameplay camera continues running in the
background (the "spectated camera" tracked by `CameraManager`) - toggling back
puts you exactly where the gameplay camera now is, not where you left off.

#### Summary line
Shows real-time camera position (world coords + tile coords) and pitch / yaw.
Useful when placing fixed camera shots or noting positions for screenshots.

#### Login Scene
Tunes the login scene only:

- **Offsets X / Y / Z** sliders.
- **Pitch / Yaw** inputs.
- **Tour mode** controls - pause, resume, restart.
- **Render distance** sliders for terrain and objects.

Defaults come from `LoginSceneCameraDefaults` constants in
`src/source/CameraMove.h` (`OFFSET_X/Y/Z`, `ANGLE_PITCH/YAW`,
`RENDER_TERRAIN_DIST`, `RENDER_OBJECT_DIST`).

#### Game Scene - Default Camera Override
Live-overrides `CameraConfig::ForMainSceneDefaultCamera()`:

- **Far Plane** slider (range ~0.5m to 20km).
- **Camera Offset X / Y / Z** - hero-relative.
- **2D Culling Trapezoid Width** - near and far multipliers on the hardcoded
  trapezoid the Default camera uses for terrain culling.
- **Fog override** checkbox - when ticked, the next two controls take effect.
- **Fog On / Off** toggle and **Fog Start / End** as a percentage of
  `ViewFar`.

#### Game Scene - Orbital Camera Override
Live-overrides `CameraConfig::ForMainSceneOrbitalCamera()`:

- **2D Culling Trapezoid** in world units - far distance, far width, near
  distance, near width.
- **Fog On / Off**, **Fog Start / End** as percentage of `ViewFar`.

The orbital trapezoid is **seeded from the camera's natural pyramid on first
enable** so the first tweak doesn't snap.

#### Debug Section
- **Debug Visualization** toggles - Character Pick Boxes, Item Pick Boxes,
  Item Cull Sphere, Tile Grid.
- **Item Cull Radius** slider.
- **Rendering toggles** - Terrain, Static Objects, Effects, Dropped Items,
  Weather, Item Labels.

The Tile Grid overlay is **batched into one `glBegin` pass**
(commit `ea62acb1`) - flipping it on while standing still in a busy scene is
no longer a frame-rate cliff.

### 3.2 Graphics tab

Read-only diagnostics:

- Real-time **window** and **OpenGL viewport** dimensions.
- Screen-rate scale factors.
- Window mode (windowed / fullscreen).
- A warning if the viewport and window dimensions disagree.
- A **Copy debug info to clipboard** button - handy for bug reports.

---

## 4. What persists, what doesn't

DevEditor overrides are **runtime-only**. They live on the `m_DefaultOverride`
and `m_OrbitalOverride` member structs and are applied each frame
(`ApplyDefaultOverrideToConfig()` / `ApplyOrbitalOverrideToConfig()`).
Closing the client throws them away.

The only camera value that survives a restart is the orbital wheel-zoom
radius - see [Camera System §3](camera-system.md#what-persists).

If you want to keep a tuned set of values, copy them out of the DevEditor
and feed them into the relevant `CameraConfig::For…` factory in
`src/source/Camera/CameraConfig.h`.

---

## 5. File map

```
src/MuEditor/
  Core/
    MuEditorCore.h / .cpp        # editor lifecycle, F12 toggle wiring
    MuInputBlockerCore.cpp       # routes input to the editor when active
  UI/
    Common/MuEditorUI.h / .cpp   # shared editor chrome
    Console/
      MuEditorConsoleUI.h / .cpp # debug console / logging output
    DevEditor/
      DevEditorUI.h              # struct + interface
      DevEditorUI.cpp            # all panels, overrides, debug toggles
```
