# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Build Commands

```bash
# Configure (Release recommended)
cmake -G Ninja -B build -DCMAKE_BUILD_TYPE=Release

# Build
cmake --build build

# Debug build (enables PVZ_DEBUG and console by default)
cmake -G Ninja -B build -DCMAKE_BUILD_TYPE=Debug

# Release with cheat keys enabled
cmake -G Ninja -B build -DCMAKE_BUILD_TYPE=Release -DPVZ_DEBUG=ON

# With native CPU optimizations
cmake -G Ninja -B build -DCMAKE_C_FLAGS="-march=native" -DCMAKE_CXX_FLAGS="-march=native" -DCMAKE_BUILD_TYPE=Release

# macOS: build and run
cmake -G Ninja -B build -DCMAKE_BUILD_TYPE=Release && cmake --build build && ./build/pvz-portable
```

No test suite exists for this project. CI validates compilation only (see `.github/workflows/ci.yml`).

## Architecture

This is a C++20 reimplementation of Plants vs. Zombies GOTY Edition using SDL2 + OpenGL ES 2.0 (desktop GL 2.1 fallback). The codebase is organized into four layers:

### Entry Point
`src/main.cpp` — Creates `LawnApp`, calls `Init()` then `Start()`. Handles platform-specific bootstrap (Windows UTF-8 argv, iOS resource checks, 3DS stack size).

### Application Layer (`src/LawnApp.*`)
`LawnApp` (extends `Sexy::SexyApp`) is the central application object. It owns all game state: the current `Board`, screen widgets (`TitleScreen`, `GameSelector`, `SeedChooserScreen`, `AwardScreen`, etc.), `ProfileMgr`, `PlayerInfo`, `Music`, `ReanimatorCache`, `EffectSystem`, and all cheat/typing check states. It manages game scene transitions and the loading thread.

### Game Logic (`src/Lawn/`)
- `Board.h/.cpp` — The main gameplay board. Manages the grid, all game objects, wave spawning, and rendering. The single largest and most complex file.
- `GameObject.h/.cpp` — Base class for all in-game entities (`mX`, `mY`, `mRow`, `mRenderOrder`). Parent of Plant, Zombie, Projectile, Coin, LawnMower, GridItem.
- `Plant.h/.cpp`, `Zombie.h/.cpp`, `Projectile.h/.cpp`, `Coin.h/.cpp`, `LawnMower.h/.cpp`, `GridItem.h/.cpp` — Game entity classes.
- `Widget/` — All UI screens: `TitleScreen`, `GameSelector`, `SeedChooserScreen`, `StoreScreen`, `AlmanacDialog`, `AwardScreen`, `CreditScreen`, `ChallengeScreen`, `NewOptionsDialog`, etc.
- `System/` — `SaveGame.h/.cpp` (v4 portable save format, TLV serialization), `Music.h/.cpp`, `PlayerInfo.h/.cpp`, `ProfileMgr.h/.cpp`, `DataSync.h/.cpp`, `ReanimationLawn.h/.cpp`.

### Engine/Framework (`src/SexyAppFramework/`)
Originally based on the PopCap Games Framework, heavily modernized to C++20. Provides:
- `SexyAppBase` — Core app lifecycle, window management, event loop, resource loading.
- `graphics/` — OpenGL ES 2.0 rendering (with GLAD loader), `Graphics`, `Image`, `GLImage` classes.
- `widget/` — UI widget system (`Widget`, `WidgetManager`, `ButtonWidget`, `Dialog`).
- `sound/` — Audio via vendored SDL-Mixer-X fork (adds MO3/libopenmpt support).
- `paklib/` — PAK file reader (the game's asset archive format).
- `platform/` — Platform-specific window/input implementations (default/SDL, 3DS/citro3d, Switch/EGL, Emscripten).

### Engine Utilities (`src/Sexy.TodLib/`)
`Tod` = "Tools of Death." Standalone engine utilities:
- `TodParticle.h/.cpp` — Particle system.
- `Reanimator.h/.cpp` — Sprite animation system (reads compiled reanimation definitions).
- `EffectSystem.h/.cpp` — Visual effect management.
- `FilterEffect.h/.cpp` — Screen-space filter effects.
- `TodStringFile.h/.cpp` — Localized string loading.
- `TodCommon.h/.cpp` — Shared math/utility functions.

### Key Data Files
- `src/ConstEnums.h` — All game enumerations (SeedType, ZombieType, GameMode, AdviceType, etc.). Very large file (~1000+ lines).
- `src/GameConstants.h` — Tuning constants.
- `src/Resources.h/.cpp` — Resource IDs mapping to game assets in `main.pak`.

## Naming Conventions

From the SexyAppFramework coding philosophy:
- `m` prefix — class member variable (e.g., `mWidth`, `mBoard`)
- `the` prefix — function/method parameter (e.g., `theGameMode`, `theX`)
- `a` prefix — local variable (e.g., `aLen`, `aUtf8`)
- Public member data is preferred over getter/setter methods.
- `using namespace Sexy;` is used throughout (the framework lives in the `Sexy` namespace).

## CMake Options

| Option | Default | Purpose |
|--------|---------|---------|
| `PVZ_DEBUG` | ON in Debug, OFF otherwise | Enables cheat keys and debug displays |
| `LIMBO_PAGE` | ON | Access to hidden limbo levels |
| `DO_FIX_BUGS` | OFF | Community fixes for original 1.2.0.1073 bugs |
| `CONSOLE` | ON in Debug, OFF otherwise | Show console window (Windows only) |
| `BUILD_STATIC` | OFF | Static link for standalone exe (MinGW Windows only) |
| `LOW_MEMORY` | OFF (ON for 3DS) | Disable caching for low-RAM platforms |

## Save System

- **Global user data** (`user*.dat`): Compatible with original PvZ GOTY. Stores profile, progress, coins, Zen Garden.
- **Mid-level saves** (`*.v4`): Custom portable format using TLV (Type-Length-Value) serialization. Portable across architectures (32/64-bit, LE/BE, different compilers).
  - Key implementation: `src/Lawn/System/SaveGame.cpp` (~128KB, one of the largest files).
  - Add new fields as new TLV IDs; never reuse IDs. Provide defaults for missing fields when loading old saves.
  - `scripts/pvzp-v4-converter.py` — Python tool to inspect/export/import `.v4` files as YAML.

## Platform Abstraction

Platform-specific code is in `src/SexyAppFramework/platform/`:
- `default/` — Desktop (Linux, macOS, Windows) via SDL2
- `3ds/` — Nintendo 3DS with citro3d
- `switch/` — Nintendo Switch with EGL
- `emscripten/` — WebAssembly/browser

CMake auto-selects platform sources based on `NINTENDO_SWITCH`, `NINTENDO_3DS`, `EMSCRIPTEN`, `ANDROID`, `IOS` variables. The default desktop path is the fallback.

## CI

Single workflow (`.github/workflows/ci.yml`) builds for Arch Linux, Windows (MSYS2 UCRT64, MSVC x64), macOS, Android (arm64-v8a), iOS (arm64), and WebAssembly. Tagged commits trigger automatic GitHub Release creation and AUR publish.
