# OpenStarbound Development Guide

OpenStarbound is a community fork of the Starbound 1.4.4 source code. It fixes bugs, adds features, and improves performance while maintaining full mod compatibility with vanilla Starbound. The entire Starbound modding community uses OpenStarbound as the base layer — every change here ripples to thousands of mods. Treat this codebase accordingly.

## Fork-Specific Documentation

The following files exist in our fork for development convenience and should **not** be committed to upstream PRs:

| File | Purpose |
|------|--------|
| `AGENTS.md` | This file — development guide for AI agents and contributors |
| `doc/BUILD-STEAMDECK.md` | Steam Deck build environment setup (distrobox + vcpkg) |

If you create additional fork-specific documentation, add it to this table so future sessions know what to exclude from upstream submissions.

## Design Philosophy

**Mod compatibility is sacred.** OpenStarbound is the lowest-level foundation for all Starbound mod development. Breaking changes to existing systems break the entire ecosystem. When adding functionality:

- **Prefer additive systems over modifications.** New `.binds` files, new classes, bridge patterns that check both old and new systems. Don't rewrite working systems — extend them.
- **Don't modify vanilla behavior unless fixing a bug.** If vanilla Starbound does something a specific way, mods may depend on that exact behavior, even if it seems wrong.
- **Expose new functionality to Lua.** The scripting engine is the primary extension point for modders. New C++ features should have Lua bindings via `LuaCallbacks` wherever feasible, so modders can use them from runtime scripts without rebuilding the client.
- **Config over hardcoding.** Values that affect gameplay feel, entity behavior, or could be mod-dependent should be readable from configuration or assets, not hardcoded in C++. If a modder might want to change a value, it shouldn't require recompilation.
- **Code defensively for edge cases.** Modders will extend every system in ways you don't expect. Null-check entity lookups, handle missing config keys with sensible defaults, don't assume vanilla entity types or asset structures.
- **Keep the change surface small.** When you must modify existing files, minimize the diff. A reviewer should be able to understand what changed and why without reading hundreds of lines of context.

## Repository Map

```
README.md                           Feature overview, installation, build instructions
AGENTS.md                           ← You are here

source/
├── CMakeLists.txt                  Top-level build configuration
├── CMakePresets.json               Build presets (linux-release, windows-release, etc.)
├── vcpkg.json                      C++ dependency manifest (SDL3, GLEW, zstd, opus, etc.)
├── application/                    SDL3 window, input event loop, OpenGL renderer,
│   │                               Steam/Discord integration, audio
│   ├── StarMainApplication_sdl.cpp   Main loop, SDL event processing, gamepad handling
│   ├── StarApplicationController.hpp Virtual interface for window/cursor/audio/rumble
│   └── StarRenderer_opengl.cpp       OpenGL rendering backend
├── core/                           Foundation: data types, JSON, Lua engine, networking,
│   │                               threading, compression, crypto, image processing
│   ├── StarInputEvent.hpp            Input event types (Key, Mouse, Controller enums)
│   ├── StarJson.cpp                  JSON parser (strict — rejects duplicate keys)
│   └── StarLua.cpp                   Lua 5.2 engine integration
├── base/                           Asset loading, configuration, lighting, world geometry
│   ├── StarAssets.cpp                Asset source scanning, patching, caching
│   └── StarConfiguration.cpp        Runtime config (starbound.config) read/write
├── game/                           Core game logic: entities, world, items, player,
│   │                               physics, status effects, AI, quests, scripting
│   ├── StarPlayer.cpp                Player entity (movement, tools, aim, inventory)
│   ├── StarInput.cpp                 OpenStarbound bind system (.binds files)
│   ├── StarVehicle.cpp               Vehicle entities (mechs, hoverbikes, boats)
│   ├── StarMovementController.cpp    Physics and collision
│   ├── interfaces/                   Entity interface contracts (Loungeable, Interactive, etc.)
│   ├── items/                        Item type implementations
│   ├── objects/                      World object implementations
│   ├── scripting/                    Lua binding implementations for all game systems
│   │   ├── StarInputLuaBindings.cpp    input.* Lua callbacks
│   │   ├── StarPlayerLuaBindings.cpp   player.* Lua callbacks
│   │   └── StarRootLuaBindings.cpp     root.* Lua callbacks (config, assets)
│   └── terrain/                      World terrain generation
├── frontend/                       Game UI: menus, HUD, inventory, crafting, chat,
│   │                               voice, cinematics, options
│   ├── StarMainInterface.cpp         Main game HUD, cursor, aim, pane management
│   ├── StarOptionsMenu.cpp           Options menu with sub-panes
│   ├── StarBindingsMenu.cpp          Mod Binds UI (Lua script pane, handles controller)
│   ├── StarKeybindingsMenu.cpp       Legacy Game Binds UI (keyboard only)
│   └── StarBaseScriptPane.cpp        Base class for Lua-driven UI panes
├── windowing/                      Widget toolkit: panes, buttons, lists, sliders,
│   │                               text boxes, scroll areas, tab sets
│   ├── StarPane.cpp                  Window/pane base (drag, focus, mouse handling)
│   ├── StarPaneManager.cpp           Pane lifecycle and input routing
│   ├── StarWidget.cpp                Widget base class
│   └── StarWidgetParsing.cpp         JSON → widget construction
├── rendering/                      2D rendering: tiles, environment, text, drawables
├── client/                         Client entry point and main application class
│   └── StarClientApplication.cpp     Client game loop, input processing, state machine
├── server/                         Dedicated server entry point
├── extern/                         Bundled: Lua 5.2, fmt, curve25519, xxhash, ImGui
├── test/                           Google Test based tests (limited coverage)
├── utility/                        CLI tools (asset_packer, asset_unpacker, etc.)
├── json_tool/                      JSON manipulation CLI
├── mod_uploader/                   Steam Workshop upload tool
└── platform/                       Platform abstraction interfaces

assets/
└── opensb/                         OpenStarbound asset patches and additions
    ├── binds/                      Input binding definitions (.binds files)
    │   ├── opensb.binds              OpenStarbound keybinds (zoom, voice, building)
    │   └── controller.binds          Controller button mappings
    ├── interface/                  UI layout configs, images, Lua scripts
    │   ├── opensb/                   OpenStarbound-specific UIs
    │   │   ├── bindings/             Mod Binds menu (bindings.lua + config)
    │   │   ├── controller/           Controller settings pane
    │   │   ├── voicechat/            Voice chat settings
    │   │   └── shaders/              Shader settings
    │   ├── optionsmenu/              Options menu patches
    │   └── windowconfig/             Window config patches
    ├── humanoid/                   Character rendering patches
    ├── scripts/                    Script patches
    ├── rendering/                  Shader and rendering config
    └── *.patch                     Root-level asset patches

doc/
├── CONTROLLER_SUPPORT.md           Controller support architecture and footguns
├── BUILD-STEAMDECK.md              Steam Deck build guide (distrobox + vcpkg)
├── lua/openstarbound/              Lua API documentation (117 files)
│   ├── player.md, input.md, ...      Per-module API reference
└── json/                          JSON asset format documentation

lib/                                Platform-specific shared libraries
├── linux/                          libsteam_api.so, libdiscord_game_sdk.so
├── windows/                        Steam/Discord DLLs
└── osx/                            macOS dylibs

scripts/
├── ci/                             CI build and assembly scripts
├── linux/                          Linux run scripts
├── windows/                        Windows build scripts
└── packing.config                  Asset packer configuration
```

## Architecture Essentials

### The Two Binding Systems

This is a common source of confusion. OpenStarbound has two completely independent input binding systems:

**Legacy system** (`Game Binds` button in Options):
- Stored in `starbound.config` → `bindings` object
- Maps `KeyChord` (key + modifiers) → `InterfaceAction` enum
- Keyboard-only. What vanilla Starbound and all existing mods use.
- Implemented in `StarKeyBindings.cpp`. **Do not modify this system.**

**OpenStarbound system** (`Mod Binds` button in Options):
- Stored in `starbound.config` → `modBindings` object
- Loaded from `.binds` asset files, user customizations saved separately
- Supports keyboard, mouse, AND controller buttons
- Implemented in `StarInput.cpp`

When adding new input functionality, add it to the OpenStarbound system via `.binds` files. If you need to bridge to existing `InterfaceAction` checks, extend `isActionTaken()` to check both systems — don't modify the legacy system.

### Asset System

**Load order:** Assets load from sources listed in `sbinit.config:assetDirectories`, plus Steam Workshop mods (loaded via Steam API when `steam_appid.txt` is present). Later sources override earlier ones for the same asset path.

**Patching:** JSON assets can be patched with `.patch` files (JSON Patch operations: add, remove, replace, test, move, copy). Lua-based patching is also available via `.patch.lua` files for more complex transformations including image manipulation.

**Critical rule:** OpenStarbound's assets in `assets/opensb/` overlay vanilla assets. A file at `assets/opensb/interface/cockpit/cockpit.lua` completely replaces the vanilla `interface/cockpit/cockpit.lua`. A file at `assets/opensb/interface/optionsmenu/optionsmenu.config.patch` patches the vanilla config. Understand which you're doing.

### Pane/Widget System

All UI is built from JSON config files that define widget trees. The `StarWidgetParsing.cpp` file constructs widgets from JSON. Key widget types: `button`, `label`, `slider`, `canvas`, `list`, `scrollArea`, `textbox`, `image`.

**Panes** are windows that can be dragged, focused, and receive input events. The `PaneManager` routes mouse/keyboard events to the topmost pane. Panes can be `Lua-driven` via `BaseScriptPane` — the UI layout is JSON, the behavior is Lua.

**Slider widgets** require: `type: "slider"`, `position`, `gridImage` (mandatory — crashes without it), `range: [min, max, delta]` (must be 3 elements — crashes with 2), and a `callback`.

**BaseScriptPane** only gets `pane`, `widget`, and `config` Lua callbacks by default. If you need `root.getConfiguration`/`root.setConfiguration`, subclass it and call `m_script.setLuaRoot(make_shared<LuaRoot>())`. Do NOT also call `addCallbacks("root", ...)` — `LuaRoot` auto-registers the `root` table, and adding it again causes a "Duplicate callbacks" crash.

### Runtime Configuration

`starbound.config` is the main runtime config file. It's a JSON object read on startup and written on shutdown. The `Configuration` class provides `get()`/`set()` methods. Default values come from `AdditionalDefaultConfiguration` in `StarClientApplication.cpp`.

Lua scripts access it via `root.getConfiguration(key)` / `root.setConfiguration(key, value)`. C++ accesses it via `Root::singleton().configuration()->get(key)`.

Changes made via `root.setConfiguration` in Lua or `configuration->set()` in C++ take effect immediately in memory and persist to disk when the game exits normally.

### Lua Scripting Integration

New Lua callbacks are added via `LuaCallbacks`:
```cpp
callbacks.registerCallback("myFunction", [](String const& arg) -> Json {
    // implementation
    return result;
});
```

These are registered on script contexts via `addCallbacks("namespace", callbacks)`. The namespace becomes the Lua table name (e.g., `input.myFunction()`).

Script panes, active items, monsters, NPCs, vehicles, and status effects all have Lua scripting contexts with different available callback sets. Check the existing `scripting/Star*LuaBindings.cpp` files to see what's available in each context.

## Development Quick Reference

### Building

See `README.md` for full platform-specific build instructions. For Steam Deck builds, see `doc/BUILD-STEAMDECK.md`.

**Quick rebuild (Linux, after initial setup):**
```bash
export VCPKG_ROOT=/path/to/vcpkg
cd source/
cmake --build --preset=linux-release
```

**Repack OpenStarbound assets** (required after changing any file in `assets/opensb/`):
```bash
LD_LIBRARY_PATH=dist dist/asset_packer -c scripts/packing.config assets/opensb dist/assets/opensb.pak
```

**Unpack vanilla assets for inspection:**
```bash
LD_LIBRARY_PATH=dist dist/asset_unpacker dist/assets/packed.pak /tmp/unpacked_vanilla
```

### Runtime Setup

The game needs these in `dist/`:
- `packed.pak` — Vanilla Starbound assets (symlink or copy from your Steam install, typically at `~/.local/share/Steam/steamapps/common/Starbound/assets/packed.pak` on Linux)
- `opensb.pak` — Packed OpenStarbound assets (built with asset_packer above)
- `libsteam_api.so` / `libdiscord_game_sdk.so` — From `lib/linux/`
- `steam_appid.txt` — Contains `211820` (Starbound's Steam app ID, needed for Workshop mod loading)
- `sbinit.config` — Asset directory configuration

**Running the game:**
```bash
cd dist/
LD_LIBRARY_PATH=. ./starbound
```

### Steam Workshop Mods

Workshop mods are loaded automatically via the Steam API when `steam_appid.txt` is present and Steam is running. Mod files live at:
- **Linux:** `~/.local/share/Steam/steamapps/workshop/content/211820/<mod_id>/`
- **Windows:** `C:\Program Files (x86)\Steam\steamapps\workshop\content\211820\<mod_id>\`
- **macOS:** `~/Library/Application Support/Steam/steamapps/workshop/content/211820/<mod_id>/`

Each mod is either a directory with loose files or a `.pak` file (packed with `asset_packer`). To inspect a mod's contents:
```bash
# If it's a .pak file:
LD_LIBRARY_PATH=dist dist/asset_unpacker /path/to/mod/contents.pak /tmp/mod_contents

# If it's a directory, just browse it directly
ls ~/.local/share/Steam/steamapps/workshop/content/211820/729480149/  # Frackin Universe
```

**Frackin Universe** (mod ID `729480149`) is the largest and most popular mod. It's the baseline compatibility target — if your changes work with FU, they'll work with nearly everything.

**Important:** Workshop mods only load when Steam is running. If you launch the game without Steam (or via SSH where Steam can't connect), mods won't load and saves that depend on them will fail to validate characters.

### In-Game Debug Tools

There is no automated test suite. Testing is manual. These tools help:

```
/admin                          Toggle admin mode (bypasses restrictions, infinite items)
/debug                          Toggle debug overlay (entity IDs, collision, etc.)
/enabletech <name>              Enable a tech (dash, distortionsphere, multijump, etc.)
/spawnitem <name> [count]       Spawn an item
/spawnmonster <name>            Spawn a monster
/timewarp <multiplier>          Speed up/slow down time
/run <lua>                      Execute arbitrary Lua in the player's script context
```

**LogMap** is useful for real-time debug values displayed on screen:
```cpp
LogMap::set("my_debug_var", strf("value: {}", myValue));
```
These appear in the debug overlay. Remove them before committing.

**ImGui** is integrated for debug overlays. It's initialized in the SDL layer and drawn each frame. Useful for quick debug UIs but lives outside the normal pane/widget system.

## Footguns

### Build Issues

**GLEW static link order (Linux/GCC):** The upstream `CMakeLists.txt` has a link order bug where GLEW references GLX symbols, but `${OPENGL_LIBRARY}` is linked before GLEW. With static libraries, order matters. Fix: move GLEW before OpenGL and add explicit `-lX11`. This is an upstream bug — apply the patch locally, don't commit it to feature branches. See `doc/BUILD-STEAMDECK.md` for the exact diff.

**opensb assets must be packed into a .pak:** Pointing `assetDirectories` at the raw `assets/opensb/` directory does NOT work. The asset scanner treats subdirectories as individual sources and skips root-level files like `preload.config` and `*.patch` files, causing a fatal crash on startup. Always pack with `asset_packer`.

**CMake minimum version:** `CMakePresets.json` requires CMake 3.24+. Ubuntu 22.04's default is 3.22. Use the Kitware APT repo.

### JSON Parsing

**Starbound's JSON parser rejects duplicate keys.** Standard JSON validators (Python's `json.load`, browser consoles) won't catch this because the JSON spec is ambiguous on duplicates. If you have two entries with the same key in a JSON object, the game crashes on startup with a parse error.

**Slider widgets crash without `gridImage`** (error: "No such key in Json::get") and **crash with 2-element `range`** (error: "Improper conversion to int from null"). Always use `"range": [min, max, delta]` (3 elements) and include `"gridImage"`.

### Input System

**`processControls()` in StarPlayer.cpp has a priority conflict.** If `m_pendingMoves` already contains Left/Right (from keyboard `moveLeft()`/`moveRight()` calls), the analog stick's `setMoveVector()` is silently ignored. Both paths can fire in the same frame via the `isActionTaken` bridge.

**`setShifting()` is overwritten every frame.** The main game loop calls `setShifting(isActionTaken(PlayerShifting))` each tick. A one-shot toggle via `setShifting(!shifting())` gets immediately reset. Use a persistent toggle bool that's OR'd with the per-frame keyboard state.

**Steam Deck registers two gamepads.** The physical Neptune controller AND a Steam Virtual Gamepad both send SDL axis events. If you don't filter by controller ID, axis values oscillate between the real stick and the idle virtual pad every frame. Track an active controller ID and ignore events from others.

**Steam Deck trackpads generate constant MouseMoveEvents** even when untouched. In auto-detection logic, require significant mouse movement magnitude (>2px) before switching input modes, or trackpad noise will flip-flop the state every frame.

### Lua/Script Panes

**`BaseScriptPane` doesn't have `root` callbacks.** It only gets `pane`, `widget`, and `config`. If your Lua script needs `root.getConfiguration`/`root.setConfiguration`, subclass `BaseScriptPane` and call `m_script.setLuaRoot(make_shared<LuaRoot>())` in the constructor. `LuaRoot` auto-registers the `root` callback table — do NOT also call `addCallbacks("root", ...)` or you'll crash with "Duplicate callbacks named 'root'".

**Checkable buttons toggle independently.** Starbound's `checkable` button property creates a self-toggling button. The internal toggle fires after your callback, potentially undoing `setChecked()` calls. For radio-group behavior, use non-checkable buttons and swap images via `widget.setButtonImages()` in Lua.

### Multiplayer

The game has a `StarProtocolVersion` for multiplayer compatibility. Changes to networked state (`NetElement` fields, packet formats in `StarNetPackets`) can break cross-version multiplayer. Be aware of this when modifying entity state that's synchronized over the network.

## Troubleshooting

### Reading Crash Logs

Logs are at `dist/logs/starbound.log` (or `storage/starbound.log` depending on `sbinit.config`). Look for:
- `[Error] Fatal Exception caught:` — The primary crash cause
- `Caused by:` — The underlying exception (often more useful than the wrapper)
- Stack traces include file names and line numbers (in RelWithDebInfo builds)

### Common Crashes

| Symptom | Likely Cause |
|---------|-------------|
| `JsonParsingException: duplicate entry for key` | Two entries with the same key in a JSON asset file |
| `Improper conversion to int from null` | Slider `range` array has fewer than 3 elements |
| `No such key in Json::get("gridImage")` | Slider widget missing mandatory `gridImage` property |
| `Duplicate callbacks named 'root'` | Called both `setLuaRoot()` and `addCallbacks("root", ...)` |
| `No such asset '/preload.config'` | Raw opensb directory used instead of packed .pak |
| `Cannot create IPC pipe to Steam client` | Steam isn't running — Workshop mods won't load |
| `No such unique stat effect 'fu...'` | Character save requires Frackin Universe but FU isn't loaded |
| `Could not read JSON asset /binds/...` | Malformed JSON in a .binds file |
| Asset loading errors with mod paths | Mod expects assets from a different platform path |

### Validating Changes

Since there's no comprehensive test suite, verification is manual:

1. **Build succeeds** — necessary but not sufficient. Many issues only manifest at runtime.
2. **Game launches to title screen** — catches asset loading errors, JSON parse failures, widget construction crashes.
3. **Can load a character and play** — catches script errors, entity initialization failures.
4. **Test with Frackin Universe installed** — FU touches nearly every system. If it works with FU, it works with most mods.
5. **Check the log** — Even if the game runs, `[Error]` lines in the log may indicate silent failures that will break specific scenarios.
