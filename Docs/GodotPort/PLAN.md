# MechCommander 2 on Godot: port plan

Status: draft, planning only. Nothing in this plan has been built yet.

## Goal

Run MechCommander 2 on current 64-bit Windows and Linux, using Godot 4 for graphics, sound, input and windowing, with the original C++ game code doing everything else. Then build a mission editor as a Godot editor plugin on top of the same code.

"Done" for the first release means: the full single-player campaign can be played start to finish on Windows and Linux, looking and behaving like the original.

## Approach in one paragraph

The game does almost all of its platform work through a library called GameOS, using two headers, `Source/GameOS/include/GameOS.HPP` and `ToolOS.hpp`. The direct Windows calls that remain are small things like timers, file copies and message boxes (handled in M2). GameOS itself was not released (headers only, no source, no `.lib`). So the port writes a new GameOS that implements those headers on top of Godot, and loads the game logic as a Godot GDExtension (a native C++ library that Godot loads). The game code gets portability fixes but no gameplay changes. Godot provides the window, the GPU, audio, input and the export pipeline. The game keeps doing its own simulation, AI, scripting, pathfinding and even its own 3D transforms.

## What is in the repo, and what is missing

| Piece | State | Notes |
|---|---|---|
| Game logic (`Source/Code`) | Present | `MechCmd2.vcproj`: 95 source files, ~131k lines |
| Engine library (`Source/MCLib`) | Present | 79 files, ~86k lines (terrain, objects, file formats, mission-script interpreter, pathfinding) |
| Renderer (`MCLib/MLR`) | Present | 52 files, ~28k lines. Does transforms and lighting on the CPU |
| Effects (`MCLib/GOSFX`) | Present | 19 files, ~11k lines |
| Math and containers (`MCLib/Stuff`) | Present | 77 files, ~27k lines |
| UI library (`Source/GUI`) | Present | 8 files, ~6k lines |
| Mission editor (`Source/Editor`) | Present | MFC, Windows only. Reference for the new editor, not ported |
| Game data | Present | `Source/Data` and `FinalBuild`, ~1.3 GB |
| **GameOS** | **Missing** | Headers only. Must be rewritten (this plan) |
| **Multiplayer transport** | **Stubbed** | 123 of 127 `MultiPlayer` methods in `Code/multplyr.cpp` are empty. Message definitions in `multplyr.h` remain |
| **Movie playback** | **Stubbed** | `MC2Movie::BltMovieFrame` and `MC2Movie::render` in `Code/mc2movie.cpp` are empty, and there are no movie files |
| UI strings | Present, Windows format | 3,756 strings in `Source/Script1.rc`, normally compiled into `mc2res.dll` |

Line counts are the `.cpp`/`.c` files listed in each `.vcproj`.

## Architecture

```
+------------------------------------------------------------------+
| Godot 4 (window, GPU, audio, input, export)                      |
|                                                                  |
|   MC2Game node (GDExtension, C++)                                |
|     _ready()    -> Environment.InitializeGameEngine()            |
|     _process()  -> feed input, set frameRate,                    |
|                    Environment.DoGameLogic(),                    |
|                    begin frame, Environment.UpdateRenderers(),   |
|                    submit frame                                  |
|     exit        -> Environment.TerminateGameEngine()             |
+------------------------------|-----------------------------------+
                               | calls
+------------------------------v-----------------------------------+
| Original game code, modernized in place                          |
|   Code (game logic)  GUI  MCLib  MLR  GOSFX  Stuff               |
+------------------------------|-----------------------------------+
                               | calls gos_* through the original headers
+------------------------------v-----------------------------------+
| New GameOS implementation (C++)                                  |
|   common:   memory heaps, files and paths, timers, .d3f fonts,   |
|             string table, settings, debug output                 |
|   godot:    renderer, textures, audio, input                     |
|   headless: no-op renderer/audio/input for tests and tools       |
+------------------------------------------------------------------+
```

### How the game is driven

MC2 does not own its main loop. At startup it fills in a `gosEnvironment` structure with four callbacks: `InitializeGameEngine`, `DoGameLogic`, `UpdateRenderers` and `TerminateGameEngine` (`Code/MechCmd2.cpp:2562`, declared at `GameOS/include/GameOS.HPP:261`). The `MC2Game` node calls them from Godot's lifecycle methods. The game is single-threaded, so all calls stay on Godot's main thread.

GameOS also defines globals the game reads, for example `frameRate` (declared `extern` at `Code/MechCmd2.cpp:153` and turned into `frameLength` at `Code/MechCmd2.cpp:1982`). The new GameOS controls game timing through it, which also gives tests a fixed timestep. The exact list of symbols GameOS must define comes out of the first link in M0.

### Rendering design

The game hands GameOS triangles that are already transformed to the screen. A `gos_VERTEX` (`GameOS.HPP:2036`) holds screen `x, y` in pixels, depth `z` from 0 to 1, `rhw` (1/w for perspective-correct texturing), a diffuse colour, a specular colour with fog in its alpha, and `u, v`. There are also 2-UV and 3-UV variants for multitexturing.

So the Godot side does not build a 3D scene. It rasterizes the game's triangle stream in submission order with the render state that was current for each call. That state space is small. Across the game code only 17 `gos_State_*` values are set:

- texture on or off
- blending: `OneZero`, `AlphaInvAlpha`, `OneOne`, `OneInvAlpha`
- alpha test
- filtering: none, bilinear, trilinear
- texture addressing: wrap or clamp
- depth test and depth write
- fog, specular, flat or Gouraud shading
- texture blend: modulate, modulate-alpha, decal
- wireframe, and multitexture in three places

Stencil is never used.

Plan:

1. Record every draw call into a per-frame list: vertices, indices, texture handle and a compact render-state key.
2. At the end of the frame, replay the list in order into an offscreen colour and depth target, then show that target full-screen.
3. One shader handles all texture-blend, fog and specular combinations through uniforms or specialization constants. A small cache holds one pipeline per blend, depth, alpha-test, wireframe and vertex-format combination.
4. In the vertex shader, convert pixels to clip space: `ndc = (x / width * 2 - 1, 1 - y / height * 2)`, `w = 1 / rhw`, `position = (ndc * w, z * w, w)`.

Two ways to do step 2 in Godot. A spike at the start of M3 picks one:

- **A, preferred: `RenderingDevice`.** Godot's low-level GPU API. It gives exact control over draw order and pipeline state. It needs the Forward+ or Mobile renderer (Vulkan, Direct3D 12 or Metal), and the work must run on Godot's render thread.
- **B, fallback: per-frame meshes.** Use `ArrayMesh` surfaces with a spatial shader that writes `POSITION` directly, and force order with render priority. Works with every Godot renderer, including OpenGL, but draw order and state changes are harder to keep exact.

Textures: `gos_NewTextureFromMemory`, `gos_NewEmptyTexture` and `gos_NewTextureFromFile` create GPU textures and keep a CPU copy, so `gos_LockTexture` and `gos_UnLockTexture` can read and modify pixels and then re-upload. Formats used are `gos_Texture_Solid`, `Alpha`, `Keyed` and `Detect`.

Text: GameOS draws all text from `.d3f` bitmap fonts (28 files in `FinalBuild/assets/Graphics`). The format is documented in `GameOS/include/Font3D.hpp`. Text is drawn through the same triangle path.

### Everything else GameOS does

The full list, with counts and calling libraries, is in [GameOS-API-Usage.md](GameOS-API-Usage.md). Regenerate it with `python3 Docs/GodotPort/gameos_usage.py`. In summary, 107 of the 219 declared names are used:

| Group | Names used | Godot side |
|---|---:|---|
| Rendering | 11 | Design above |
| Textures | 6 | GPU texture plus CPU copy |
| Text and fonts | 8 | `.d3f` loader plus triangle path |
| Input | 17 | Key and mouse state tables filled from `InputEvent`s. Joystick via `Input`. The 7 IME (East Asian text input) calls are stubbed at first |
| Audio | 8 | One `AudioStreamPlayer` per channel, WAV loaded from memory. Volume, pan and frequency "sliders" mapped to player properties. Music streams through the same path |
| Files | 20 | Plain C++ file I/O with path translation (see below) |
| Memory | 8 | Heaps become tagged wrappers over `malloc`, with optional tracking |
| Time | 5 | `std::chrono`, or a fixed clock in test mode |
| Networking | 2 | Stubbed until multiplayer is rebuilt (M8) |
| Debug and errors | 10 | `STOP`, `PAUSE`, `SPEW` and `gosASSERT` log through Godot and break into the debugger. Statistics are no-ops at first |
| System | 12 | Settings in a config file under `user://` (replaces the registry). Strings from the converted `.rc` table. `gos_TerminateApplication` quits Godot |

### Files and paths

The game loads data relative to its working directory (`data\missions\...`, `assets\...`, `*.fst` archives) using Windows paths with mixed case. MCLib reads files with C runtime calls (`_open`, `_read`, `_lseek` in `MCLib/File.cpp`), and its own `.fst` archive reader (`MCLib/fastfile.cpp`) sits on top. The port adds one path function used by both GameOS and `File.cpp`. It converts backslashes and resolves each path component case-insensitively, with a cache. The data directory is a command-line option or project setting. Original data stays in its original formats and is read by the original loaders. Nothing is imported into Godot's resource system.

### Strings

All UI text goes through `cLoadString` (`MCLib/Utilities.h:99`), which calls `gos_GetResourceString(gosResourceHandle, id)`. The handle comes from `gos_OpenResourceDLL("mc2res.dll")` (`Code/MechCmd2.cpp:817`). A build-time tool converts `Source/Script1.rc` (302 string tables, 3,756 strings) into a plain table the new GameOS loads. That also makes translation possible later.

## Decisions

Defaults chosen so work can start. Change any of them before the milestone that depends on it.

| Decision | Default | Needed by |
|---|---|---|
| Target platforms | Windows x64 and Linux x64. macOS later, since Apple Silicon also needs the x86 assembly removed, which M2 does anyway | M2 |
| Godot version | Latest stable 4.x at the start of M3, pinned. Upgrade deliberately | M3 |
| Renderer backend | A (`RenderingDevice`), unless the M3 spike fails | M3 |
| Visual target | Faithful to the original first. Enhancements after M5 | M3 |
| Where the code lives | This repo. Modernize `Source/` in place. Tag the original import (`4ba28ca5`) before the first change so it stays easy to diff | M0 |
| Build system | CMake for the game libraries and the extension, with `godot-cpp` as a pinned submodule | M0 |
| Multiplayer | Out of scope until single-player parity (M8) | M8 |
| Project name for public builds | Not "MechCommander". The license grants no trademark rights (`EULA.md` 3(B)), and the original art is non-commercial only (3(A)) | First public build |

## Repository layout (proposed)

```
CMakeLists.txt                 new: builds everything below
Source/                        original code, modernized in place
  GameOS/include/              original headers, kept as the API contract
Port/                          new code
  Platform/                    small replacements for Win32 calls outside GameOS
  GameOS/Common/               heaps, files and paths, timers, fonts, strings, settings
  GameOS/Headless/             no-op backend for tests and command-line tools
  GameOS/Godot/                Godot backend (renderer, audio, input)
  Extension/                   GDExtension entry point, MC2Game node
  ThirdParty/godot-cpp/        submodule, pinned
  Tools/                       rc-to-strings converter, .pak round-trip test, and others
Godot/                         the Godot project (project.godot, scenes)
  addons/mc2_editor/           mission editor plugin (M7)
Docs/GodotPort/                this plan and generated inventories
```

## Milestones

Estimates assume one experienced C++ developer working full time. Double them for half time. Each milestone ends with something that runs or a test that passes.

### M0: Build system and first link (1–3 weeks)

1. Tag the original import.
2. Generate CMake targets from the `.vcproj` file lists for `Stuff`, `MLR`, `gosFX`, `MCLib`, `GUI` and `MechCmd2`, with a script so the lists don't drift. Skip `EditorMFC`, `Viewer`, `aseconv` and `ablt` for now.
3. Compile on MSVC 2022, 32-bit, with legacy switches (`/Zc:forScope-`, `/Zc:wchar_t-`, `/permissive`, `_CRT_SECURE_NO_WARNINGS`). This separates old-dialect problems from 64-bit problems.
4. Link against an empty GameOS where every function stops with "not implemented". The linker's list of unresolved symbols is the exact set GameOS must provide; check it into `Docs/GodotPort/`.
5. Add GitHub Actions: build on `windows-latest` (MSVC) on every push, so work can proceed without a local Windows machine.

Exit: the game links on MSVC x86 against stub GameOS in CI.

### M1: Headless boot (2–4 weeks)

1. Implement the platform-independent parts of GameOS in `Port/GameOS/Common`: memory heaps, files and paths, timers, settings, string table (with the `.rc` converter), debug output, and `Environment` setup.
2. Implement `Port/GameOS/Headless`: rendering, audio and input calls accept their arguments and do nothing.
3. Add a test mode with a fixed `frameRate`, a fixed random seed (today it is seeded from `timeGetTime()` at `Code/MechCmd2.cpp:923`, and `Code/warrior.cpp:366` seeds `srand` from the clock) and a frame limit.
4. Run `-mission mc2_01` (switch handled at `Code/MechCmd2.cpp:2377`) for a fixed number of frames without crashing.
5. Have the headless renderer count draw calls, vertices and render-state changes per frame. These numbers size the M3 renderer spike.

Exit: headless runs load and simulate every shipped mission for 10 minutes of game time without crashing (Windows x86, CI).

### M2: 64-bit and Linux (4–8 weeks)

1. Rewrite the inline x86 assembly (34 files, mostly `MCLib/Heap.cpp`, `MCLib/lzcomp.cpp`, `MCLib/vfxtile.cpp` and the MLR and Stuff math headers) as portable C++.
2. Fix pointer-to-integer casts (around 270 `DWORD` or `long` casts of pointers). Replace MCLib's own heap (`MCLib/Heap.cpp`) with a `malloc`-backed version keeping the same interface.
3. On-disk structures: switch every struct read from or written to files (29 `#pragma pack` sites, about 23 raw struct reads) to fixed-width types (`int32_t`, `uint32_t`), so the layout stays byte-identical on 64-bit and on Linux, where `long` is 64 bits.
4. Fix `#include` case mismatches (990 of 3,248 project includes) with a script, and convert backslash include paths.
5. Replace the remaining Win32 calls outside GameOS (about 190 call sites) with `Port/Platform`. The largest are `QueryPerformanceCounter` (30), `CopyFile` (23), `_splitpath` (17), `OutputDebugString` (14), `MessageBox` (13) and `timeGetTime` (12). Remove the crash reporter (`Code/dw.cpp`) and stub the Windows-only parts of the script debugger (`MCLib/Abldbug.cpp`), which the interpreter still references.
6. Remove the legacy compiler switches: fix code that relies on `for` loop variables leaking out of the loop (29 project configurations had `ForceConformanceInForLoopScope="false"`).
7. Add Linux CI (GCC and Clang) running the headless tests, plus an AddressSanitizer and UndefinedBehaviorSanitizer job. Replacing the custom heaps will expose memory bugs they used to hide.

Exit: the headless test from M1 passes on Windows x64 and Linux x64, and per-second state checksums match between the two (see Testing).

### M3: Godot host and renderer (6–10 weeks)

1. Spike (about 1 week): a GDExtension that draws a stream of pre-transformed, textured, alpha-blended triangles through `RenderingDevice` in submission order. Load it with the per-frame draw-call and vertex counts recorded by the headless build (M1, step 5). Choose backend A or B.
2. Build `Port/Extension` with the `MC2Game` node and the frame loop described above.
3. Implement rendering, viewport and screen-mode calls, the texture API and render-state handling in `Port/GameOS/Godot`.
4. Handle resolution. The game's UI layouts are built for fixed modes from 640×480 to 1600×1200 (`Code/prefs.cpp:427`). Render at one of those and scale to the window, keeping the aspect ratio. Widescreen comes later.

Exit: the main menu renders correctly, and mission `mc2_01` renders terrain, buildings, mechs and effects that match the original's look.

### M4: Input, text and audio: playable (4–6 weeks)

1. Keyboard and mouse state from Godot input events, mapped to `gosEnum_KeyIndex`. Text entry (`gos_GetKey`), cursor position and warping, joystick axes.
2. `.d3f` font loader and the `gos_Text*` functions.
3. Audio resources and channels, sliders, looping, and streamed music.
4. Settings persistence (`gos_LoadDataFromRegistry` and `gos_SaveDataToRegistry`) in a `user://` config file.

Exit: from the splash screen, through logistics and mission briefing, play `mc2_01` to victory with sound, then save and load.

### M5: Single-player parity (6–10 weeks)

1. Play the whole campaign on Windows and Linux. Fix crashes, rendering differences and gameplay bugs.
2. Profile frame time on modest hardware.
3. Package Godot export templates for Windows and Linux with the data directory beside the executable.

Exit: the campaign is completable on both platforms with no known crashes. Tag it as the first release.

Total to here: roughly 6–10 months of full-time work.

### M6: Quality-of-life (optional, 2–4 weeks)

Widescreen and high-DPI UI scaling, remappable keys, windowed and borderless modes, and better texture filtering. Each item is independent.

### M7: Mission editor as a Godot editor plugin (2–4 months)

1. File library: read and write `.pak` and `.fit`. Load and re-save all 44 shipped missions and compare the output byte for byte.
2. Viewer: terrain, objects and minimap in the editor viewport, using the same C++ loaders through the extension.
3. Mission editing: place units and buildings, teams, players, and objectives and conditions (`Source/Editor/Objective.cpp` is the reference).
4. Terrain editing: height, texture, water and forest brushes. On save, run the original pathfinding builder (`MOVE_buildData` in `MCLib/Move.cpp`) unchanged.
5. A "Test" button that saves under a test name and runs the project with `-mission <name>`. Godot runs the game in a separate process, so each test starts from clean state.

Exit: create a new mission, test-play it and complete it.

### M8: Extensions (optional, each independent)

| Item | Estimate | Notes |
|---|---|---|
| Maps up to 240×240 vertices | 1–3 weeks | Code is sized for it (`MAX_MAP_CELL_WIDTH = 720` at `MCLib/Terrain.h:58`). Raise `COLOR_MAP_HEAP_SIZE` (`MCLib/terrtxm2.cpp:52`). Pathfinding limits fit: the busiest shipped map uses 658 areas and 1,377 doors against limits of 10,000 each |
| Maps larger than 240×240 | weeks+ | Object IDs encode cells as `4096 + row × 720 + col` (`Code/unitdesg.h:67`), so the ID scheme must change. Also fixed arrays such as `Code/trigger.h:49`, and pathfinding limits |
| Multiplayer | 2–3 months | Rebuild the stubbed `MultiPlayer` transport on ENet (Godot ships it). Lobby, joining and sync from scratch |
| Movies | 1–2 weeks | Implement `MC2Movie` with Godot's video player if replacement videos are made |
| Effects polish, modern lighting | open-ended | Modern lighting needs real 3D geometry instead of pre-transformed triangles, a large change to MLR |

## Testing

- **Format round-trip** (from M7, but useful earlier): load and re-save every `.pak` and `.fit` and compare bytes.
- **Headless soak** (from M1): every shipped mission runs for N minutes of game time with fixed `frameRate` and seed, in CI.
- **State checksums** (from M2): once a game-second, hash every unit's position, health and order. Compare x86 against x64 and Windows against Linux. A mismatch points to a portability bug, usually integer width or uninitialized memory. Floating-point differences between compilers may need a small tolerance or a per-platform baseline.
- **Sanitizers** (from M2): AddressSanitizer and UndefinedBehaviorSanitizer builds run the headless soak on Linux.
- **Screenshot comparisons** (from M3): capture fixed frames from a few missions in test mode and compare against approved images.
- **Manual playthrough** (M4 and M5): campaign checklist, one row per mission.

## Risks

| Risk | Impact | Mitigation |
|---|---|---|
| GameOS behaviour is only documented by header comments | Subtle differences in text layout, texture handling, heaps or timing | Implement in small steps behind logging. `GameOS.HPP` is about 3,200 lines with detailed comments. Start each function as a loud stub |
| Many small draw calls per frame | Slow frames | Measure real counts in M1, spike first in M3, and batch consecutive calls with identical state |
| Hidden 64-bit bugs | Crashes or corruption that only show up in play | Compiler warnings for pointer truncation treated as errors, sanitizers, state checksums |
| Replacing custom heaps exposes use-after-free | Crashes | Sanitizers in CI from M2 |
| Floating-point results change with the compiler | AI or balance edge cases shift slightly | Accept small drift. Checksums compare x64 builds against each other once M2 is done |
| Godot API changes between versions | Rework | Pin the Godot and `godot-cpp` versions, and upgrade on purpose |
| Data files with mismatched case on Linux | Missing assets | Case-insensitive path resolution with a cache from M1 |
| Licensing | Takedown | Non-commercial distribution of original art only, and no Microsoft trademarks in names, logos or store pages |

## First steps

1. Agree on or change the defaults in **Decisions**.
2. Tag the original import.
3. M0, step 2: generate CMake targets from the `.vcproj` files.
4. M0, step 5: Windows CI building the game against a stub GameOS.

## Key code references

| What | Where |
|---|---|
| GameOS callbacks set up | `Source/Code/MechCmd2.cpp:2562` |
| Callback declarations | `Source/GameOS/include/GameOS.HPP:261` |
| Pre-transformed vertex | `Source/GameOS/include/GameOS.HPP:2036` |
| Frame length from `frameRate` | `Source/Code/MechCmd2.cpp:1982` |
| `-mission` switch | `Source/Code/MechCmd2.cpp:2377` |
| Resource strings | `Source/MCLib/Utilities.h:99`, `Source/Script1.rc` |
| `.d3f` font format | `Source/GameOS/include/Font3D.hpp` |
| File I/O | `Source/MCLib/File.cpp`, `Source/MCLib/fastfile.cpp` |
| Pathfinding builder | `Source/MCLib/Move.cpp` (`MOVE_buildData`) |
| Mission save (original editor) | `Source/Editor/EditorData.cpp:1004` |
| Stubbed multiplayer | `Source/Code/multplyr.cpp` |
| Stubbed movies | `Source/Code/mc2movie.cpp` |
