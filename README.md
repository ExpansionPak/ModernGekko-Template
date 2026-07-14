# GCWIIRecomp-template

A template for building a static-recompilation pipeline for GameCube and Wii games, built on:

- [DolRecomp](https://github.com/ExpansionPak/DolRecomp) - static recompiler that turns a GameCube/Wii DOL into split C source.
- [ModernGekko](https://github.com/ExpansionPak/ModernGekko) - the runtime the recompiled C links against (built on a Dolphin-derived core for video/audio/HLE).

Both are pulled in as git submodules under `lib/`. Everything is handled through the top-level `Makefile`. This repo has no game-specific code of its own, so it's a starting point for standing up a recompilation project for any GameCube or Wii title you own.

## Using This Template

Click **Use this template** on GitHub (or fork/clone it directly) to create your own project repo. Nothing in the `Makefile` or submodule setup needs to change to point it at a different game, just pass a different `ISO=` the first time you run it. Rename the repo to whatever fits your project.

## Dependencies

CMake, Ninja, and pkg-config, plus a C11/C++23 toolchain. DolRecomp and ModernGekko both build on macOS, Linux, and Windows; this repo's `Makefile` doesn't do anything platform-specific itself, it just drives the same CMake builds each submodule supports natively.

On macOS, via Homebrew:

```
brew install cmake ninja pkg-config
```

Xcode's command line tools are also required (AppleClang 14.0.3+; verified on AppleClang 17).

## Getting the Source

```
git clone --recurse-submodules git@github.com:<your-org>/<your-repo>.git
cd <your-repo>
```

If you already cloned without `--recurse-submodules`, `make` will fetch them for you on first run. To do it manually instead:

```
git submodule update --init --recursive
```

> [!NOTE]
> `lib/ModernGekko` vendors a large chunk of Dolphin's dependency tree (SDL, fmt, imgui, Vulkan headers, etc.), so the first submodule sync takes a while and pulls a few hundred MB.

## Recompile and Run

Bring your own legally-owned ISO, no game data is included in or downloaded by this repository. Works with any GameCube or Wii disc; there is no default game, so `ISO=` (or an already-extracted `GAME=`) is required. Point `ISO` at a dump and run:

```
make run ISO=/path/to/Your\ Game.iso
```

This builds DolRecomp and ModernGekko, extracts the ISO, recompiles `main.dol` to C, compiles the result into a native module, and launches the game in a window.

Each game gets its own directory under `extracted/<slug>/`, where `<slug>` is derived from the ISO's filename, so multiple games coexist without clobbering each other. `ISO` is only needed the first time per game, once extracted, run it again by slug instead:

```
make run ISO=iso/Your\ Game.iso        # first time for a new game
make run GAME=Your-Game-Slug           # afterwards
```

Drop ISOs under `iso/` at the repo root (gitignored) for a stable local path.

To just build the tools without touching a game, or to produce the compiled module without launching it:

```
make tools
make recompile ISO=/path/to/game.iso
```

`moderngekko-port` caches compiled modules by DOL hash and toolchain identity, so re-running `recompile`/`run` after the first build is cheap, it hits cache instead of recompiling.

> [!NOTE]
> Wii ISOs need [Wiimms ISO Tools](https://wit.wiimm.de/) (`wit`) for extraction, `make` downloads it automatically into `extern/wit` on first use. GameCube extraction is built into DolRecomp directly and doesn't need this.

## Makefile Targets

Run `make help` (or just `make`, the default target) for this list:

| Target       | Description                                                |
|--------------|--------------------------------------------------------------|
| `tools`      | Build DolRecomp and ModernGekko                             |
| `extract`    | Extract a GameCube/Wii ISO into `extracted/<slug>/`         |
| `recompile`  | Recompile + compile a runnable module                       |
| `run`        | Recompile (if needed) and launch the game                   |
| `clean`      | Remove all build output                                     |

## Variables

| Variable           | Default                                    | Description                                              |
|--------------------|--------------------------------------------|----------------------------------------------------------|
| `ISO`              | *(none)*                                   | Path to a game ISO. Required the first time per game, also determines that game's slug. |
| `GAME`             | *(none)*                                   | Select an already-extracted game by slug instead of `ISO=`. Required if `ISO` isn't given. |
| `JOBS`             | detected CPU count                         | Parallel build jobs passed to CMake/Ninja.                |
| `CMAKE_BUILD_TYPE` | `Release`                                  | Passed to both submodule builds.                          |
| `RUN_ARGS`         | *(empty)*                                  | Extra flags forwarded to `moderngekko-run` via `make run`, e.g. `--headless`, `--graphics Vulkan`. |

For example, to force a debug build with extra runner flags:

```
CMAKE_BUILD_TYPE=Debug make run ISO=/path/to/game.iso RUN_ARGS="--headless"
```

## Controller Input

ModernGekko has no in-app controller configuration UI (same as Dolphin's NoGUI frontend it's built on), bindings come from `Config/GCPadNew.ini` / `Config/WiimoteNew.ini` in ModernGekko's user directory (`~/.local/share/moderngekko/Config/` by default), which nothing in this pipeline creates for you. Without one, GameCube pad input silently does nothing. You'll need to hand-author or copy in a working ini, Dolphin's ini format and key names are stable and documented by the project itself.

## Cleaning Up

```
make clean           # everything below
make clean-extracted # extracted disc + compiled modules only, keeps built tools
make clean-tools     # DolRecomp/ModernGekko build trees only
```

## License

DolRecomp and ModernGekko are each distributed under their own upstream licenses (see `lib/DolRecomp/LICENSE` and `lib/ModernGekko/LICENSE`, the latter GPL-3.0-or-later due to its Dolphin-derived runtime). No Nintendo disc image, extracted game data, keys, or copyrighted assets are part of this repository, bring your own legally-owned dump.
