# Options Window and Config

Behaviour changes that landed in PR
[#335](https://github.com/sven-n/MuMain/pull/335) for the options window,
windowing, audio sliders, and the runtime config file.

For the camera architecture see [`camera-system.md`](camera-system.md).

---

## 1. Resolution and windowed/fullscreen toggle work without a restart

`src/source/NewUIOptionWindow.cpp:175-340` handles the apply path:

- **Fullscreen → Windowed** (lines 211-230): calls
  `ChangeDisplaySettings(nullptr, 0)` to drop back to the desktop mode, then
  rebuilds the window with the canonical windowed style:
  `WS_OVERLAPPED | WS_CAPTION | WS_SYSMENU | WS_MINIMIZEBOX | WS_BORDER |
  WS_CLIPCHILDREN | WS_VISIBLE`.
- **Windowed → Fullscreen** (lines 232-319): drains the message queue, resets
  to desktop mode, then attempts the mode change up to 3 times (some drivers
  reject the first call right after a previous mode change). Sets the window
  to `HWND_TOPMOST` once accepted.
- **Window style is identical** across initial window creation (`Winmain.cpp`,
  CreateWindow path) and every subsequent toggle (line 219-221, line 810). No
  more "different style after toggle" surprises.

You can switch resolution and windowed/fullscreen at any time without
quitting and relaunching the client.

## 2. Combo-box clicks no longer leak through

The resolution combo-box dropdown used to allow clicks to fall through to the
checkboxes behind it (e.g. accidentally toggling **windowed** while picking
a resolution). The fix runs the combo-box's `UpdateMouseEvent()` first and
short-circuits the rest of the panel when the click hits the dropdown's
bounding box (`NewUIOptionWindow.cpp:175-187`, `IsMouseOverWidget()` check).

## 3. Volume / render-level sliders round to the nearest level

`HandleVolumeSlider()` (`NewUIOptionWindow.cpp:394-430`) and
`HandleRenderLevelSlider()` (lines 456-467) now round to the nearest discrete
level instead of truncating, so:

- Slider all the way left **does** reach 0 (mute).
- Slider all the way right **does** reach the configured maximum.

Truncation previously made the endpoints unreachable from the slider - you
could only drag near them.

## 4. Exclusive-fullscreen uses the actual desktop bit depth

`Winmain.cpp:1088-1095` defines `GetDesktopBitsPerPel()`:

```cpp
DEVMODE dm;
EnumDisplaySettings(nullptr, ENUM_CURRENT_SETTINGS, &dm);
return dm.dmBitsPerPel;          // fallback: 32 if the query fails
```

It's called from the fullscreen apply path (`NewUIOptionWindow.cpp:243`) and
from the initial fullscreen mode change (`Winmain.cpp:1275`). Previously
both call sites hardcoded 32 bits, which can fail on systems with a
non-32-bpp desktop.

---

## 5. Config file: `config.ini`

The client used to keep settings in the Windows registry. PR #335 finishes
the migration to a plain `config.ini` next to the executable.

### Where it lives

- Same directory as the executable. `GameConfig` resolves it via
  `GetModuleFileNameW()` and appends `config.ini`
  (`src/source/GameConfig/GameConfig.cpp:28-29`).

### Sections

`Load()` (`GameConfig.cpp:34-60`) reads:

- `[Window]` - width, height, windowed flag.
- `[Graphics]` - render levels, vsync, fps limit, etc.
- `[Audio]` - volumes.
- `[Login]`
- `[ConnectionSettings]`
- `[Camera]` - currently the orbital wheel-zoom radius (`Zoom`).

Missing keys fall back to compile-time defaults from
`src/source/GameConfig/GameConfigConstants.h`.

### Upgrade path for existing users

There is **no automatic migration** from the old registry values
(`Load()` does not read the registry). Users upgrading from a pre-rework
build will see the default settings the first time they launch - they need
to set their preferences again, after which `Save()` (lines 62-86) persists
them to `config.ini`.

If `config.ini` is missing, it's created on the first `Save()` call (any
options window apply, scene transition that saves orbital zoom, etc.).
