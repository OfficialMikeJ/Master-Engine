# Master Engine — Alpha

> A zero-dependency, Windows-native game engine and IDE built entirely from scratch.
> No STL. No CRT. No third-party libraries. Win32 + DirectX 12 only.

---

## Overview

Master Engine is a fully hand-written game engine and editor targeting feature parity with modern commercial engines. Every system — allocator, math library, UI toolkit, renderer, physics solver, audio mixer, scripting VM, and AI toolset — is implemented from scratch in C++17 using only the Win32 API and DirectX 12 SDK headers that ship with Windows.

The engine launches with a project wizard, then opens a full IDE-style editor with a DX12 viewport, dockable panels, a live console, and a built-in build pipeline.

**Current status:** Alpha — all core systems are implemented and building. Active development continues.

---

## Build

**Requirements**
- Windows 10 / 11 (64-bit)
- Visual Studio 2022 (Community, Professional, or Enterprise) — MSVC toolchain
- Windows SDK 10.0.19041 or later (ships with VS2022)
- Optional: `clang-cl` on PATH for faster incremental builds

**Build commands**

```bat
build.bat           :: debug build (default)
build.bat debug     :: debug build  — no optimisation, full symbols
build.bat dev       :: dev build    — debug + internal tools (MASTER_DEV_BUILD)
build.bat release   :: release build — /O2, NDEBUG, zero dev bytes
```

Output: `bin\MasterEngine.exe`

No CMake. No MSBuild projects. No NuGet. One batch file, one compiler invocation.

---

## Architecture

| Principle | Detail |
|---|---|
| Zero CRT | No `malloc`, `memcpy`, `sprintf`, `sin`, `rand` from the C runtime |
| Zero STL | No `std::vector`, `std::string`, `std::map`, or any `<algorithm>` |
| Zero third-party | No ImGui, SDL, PhysX, FMOD, OpenAL, or any external library |
| Win32 + DX12 only | OS calls use `kernel32` / `user32` / `gdi32`; rendering uses D3D12 / DXGI |
| Arena-first memory | All subsystems allocate from `Arena` bump allocators; zero heap fragmentation |
| Unity build | All `.cpp` files compiled in a single `cl.exe` invocation |
| Dev / Release split | `#ifdef MASTER_DEV_BUILD` guards all internal tooling — zero dev bytes ship |

---

## Feature List

### Phase 0 — Foundation

- **Types & Primitives** — `u8 / u32 / f32 / b32 / usize` and friends; `FORCE_INLINE`, `UNUSED`, `ARRAY_COUNT`
- **Memory System** — arena bump allocator, stack allocator, pool allocator, general-purpose heap; `MemBudget` per-system high-water tracking
- **SIMD Math** — `Vec2 / Vec3 / Vec4`, `Mat3 / Mat4`, `Quaternion`, `AABB`, `Ray`, `Plane`; camera + projection matrices; no `<math.h>`
- **String & Text** — `Str` / `StrBuf`, UTF-8 utilities, format helpers; zero CRT
- **Collections** — dynamic `Array`, `Queue`, `Stack`, `HashMap` (FNV-1a), `SlotMap`
- **File I/O** — path utilities, Virtual File System with mount points, async IOCP completion
- **Logging** — levelled log (`LOG_INFO / WARN / ERROR / FATAL`), per-system memory budget tracker

---

### Phase 1 — Platform

- **Win32 Window** — per-monitor DPI V2, fullscreen toggle, resize, cursor lock/clip
- **Input** — keyboard, mouse, XInput gamepad polling; raw button / axis state
- **ActionMap** — named actions bound to keys / buttons / axes; remappable at runtime
- **Timing** — `QueryPerformanceCounter` delta timer, 60 Hz fixed-step `FixedTimer`, configurable FPS cap (default 165)

---

### Phase 2 — Master IDE Editor

- **Custom UI toolkit** — buttons, checkboxes, text fields, tab bars, scroll areas, labels; hand-drawn into a DIB via GDI; no Win32 dialog controls, no ImGui
- **Dockable panels** — Hierarchy, Inspector, Viewport, Asset Browser, Console, Build Output
- **Menu bar** — File / Edit / View / Build menus with live dropdown navigation; all items functional
- **Accessibility** — screen-reader / MSAA support
- **Status bar** — real-time FPS counter, engine status messages

---

### Phase 3 — Core Rendering (DirectX 12)

- **DX12 Core** — device creation, command queues, FLIP_DISCARD swap chain, CBV/SRV/DSV/Sampler/RTV descriptor heaps, double-buffered frame fencing
- **Shader cache** — file-watcher hot-reload; edit an HLSL file and changes apply without restarting
- **PBR material system** — albedo, normal, roughness, metallic, emissive texture channels
- **Texture loader** — BC1 / BC3 / BC5 / BC7 block-compressed + RGBA8 upload pipeline
- **GLTF mesh loader** — positions, normals, UVs, tangents, skinning weights/joints
- **GPU skinning** — compute shader bone transform accumulation
- **Tonemap pass** — ACES-like HDR → SDR output
- **Viewport isolation** — DX12 targets a `WS_CHILD` window inside the editor; GDI chrome renders to the parent with `WS_CLIPCHILDREN` so both layers are independent

---

### Phase 4 — Scene & ECS

- **Archetype ECS** — component bitmask, entity records, `SlotMap` entity storage
- **Scene graph** — dirty-flag transform propagation, parent-child hierarchy
- **Components** — Transform, Light, Camera, Mesh, Rigidbody, Audio
- **Scene serialization** — save / load scene state to disk

---

### Phase 5 — Advanced Editor Integration

- **Viewport camera** — fly camera (RMB + WASD + QE), orbit (Alt + LMB), pan (MMB), zoom (scroll), focus (F)
- **Gizmo mode** — Translate / Rotate / Scale selector (W / E / R keys + toolbar buttons)
- **Hierarchy panel** — ECS-backed tree view with depth indent, search filter, visibility toggle per entity
- **Inspector panel** — reflects active ECS component mask; live field editing pushed back to ECS transforms, lights, cameras
- **Asset Browser** — folder navigation, hot-folder watcher, file-type icons, search/filter
- **Worldspace UI** — overlay UI elements positioned in 3D space
- **Flow Graph** — visual scripting node canvas
- **Project system** — project templates and generation

---

### Phase 5.9 — Foliage Painter

- Wang-hash scatter within brush radius, sqrt-distributed uniform disk sampling
- Surface-normal alignment via half-angle quaternion; combined align + yaw rotation
- Per-instance non-uniform scale ±10%, green-biased hue tint
- Slope rejection (`dot(normal, up) >= cos(slope_max_deg)`), optional raycast surface-snap callback
- Erase brush with compact-swap removal
- 16 layers × 65,536 instances per layer (VirtualAlloc slabs)
- Brush circle overlay rendered in the viewport

---

### Phase 6 — Physics & Character Movement

- **Rigid body dynamics** — semi-implicit Euler integration, angular momentum
- **BVH broadphase** — bounding volume hierarchy for fast pair culling
- **GJK + EPA narrowphase** — exact convex collision detection and contact manifold generation
- **Sequential-impulse solver** — iterative constraint resolution, joints
- **Raycasting** — world-space ray vs. BVH
- **Pre-fractured destruction** — Voronoi fracture, material-specific chunk physics (CONCRETE / METAL / WOOD / GLASS), debris lifetime despawn
- **Kinematic Character Controller** — slide-cancel state machine, step-up, slope limit, coyote time

---

### Phase 7.1–7.2 — Audio

- **WASAPI** — exclusive-mode output, automatic shared-mode fallback
- **PCM mixer** — f32 stereo, per-voice pitch shift, volume ramp, 48 kHz
- **WAV decoder** — PCM 8/16/24/32-bit + IEEE f32, mono→stereo expansion, linear-interpolation sample-rate conversion
- **3D Spatial Audio** — HRTF stereo panning + ILD, distance rolloff curves, Doppler shift, Schroeder reverb bus, reverb zones, occlusion

---

### Phase 7.3 — Script System

- **TypeRegistry** — `REFLECT()` macros for runtime type introspection
- **EventSystem** — FNV-1a typed subscribe / fire / unsubscribe; zero allocation dispatch
- **Game DLL hot-reload** — `CopyFile` + `LoadLibrary` + `GetProcAddress`; swaps live game code without restarting the editor

---

### Phase 7.4 — AI Core

- **State Machine** — FNV-1a StateIDs, transition table, enter / update / exit callbacks
- **Behavior Tree** — Sequence, Selector, Parallel, Inverter, Succeeder, Repeater, Cooldown, Action, Condition nodes; flat arena pool; FNV-1a Blackboard key-value store
- **World Event System** — one-shot and repeating timer callbacks, no heap allocation

---

### Phase 7.5–7.6 — Weapon Systems

- **Procedural Weapon Mechanics** — Spring1D / Spring3D damped harmonic oscillator; ADS spring blend; mouse-delta sway; sinusoidal walk bob; low-frequency breathing; recoil velocity kick
- **Data-Driven Attachment System** — FNV-1a AttribIDs; MOD_ADD / MOD_MUL / MOD_OVERRIDE modifier stack; attachment definition registry; `WeaponLoadout` with `Attach_Resolve`; hidden stat penalty metadata

---

### Phase 8 — Advanced Rendering

- **Render Graph** — Kahn topological sort, resource aliasing heap, automatic D3D12 barrier batching, DOT graph export
- **GPU Resident Drawer** — `ExecuteIndirect` multi-draw, GPU frustum culling, Hi-Z occlusion culling
- **Lighting & Global Illumination**
  - Cascaded Shadow Maps (CSM) — 4 cascades, PCF soft shadows
  - Ground-Truth Ambient Occlusion (GTAO)
  - Screen-Space Reflections (SSR)
  - Adaptive Probe Volume (APV) — L2 spherical harmonics irradiance probes
- **Production Ray Tracing (DXR 1.1)**
  - Inline ray tracing: contact shadows, reflections, ambient occlusion, global illumination
  - Spatial-temporal denoiser with ping-pong history buffers
  - Graceful fallback when DXR 1.1 is not supported
- **STP Post-Processing Stack**
  - YCoCg Temporal Anti-Aliasing (TAA)
  - Kawase multi-pass bloom
  - Hexagonal-bokeh Depth of Field
  - ACES LUT tonemap
  - Chromatic aberration, film grain, vignette, motion blur
- **GPU Compute Pipeline**
  - Hi-Z pyramid builder
  - 1,000,000-particle GPU system — emit, simulate, bitonic sort

---

### Phase 9.1 — Networking & Anti-Cheat

- UDP WinSock2 server + client
- Challenge-response session key handshake
- FNV-1a HMAC packet authentication + XOR stream cipher
- Server-authoritative snapshots, client-side prediction (128-tick ring buffer), state reconciliation, lag compensation (64-tick rewind)
- **Anti-cheat detectors**
  - Pearson correlation recoil pattern detector
  - Aim-snap angular velocity threshold
  - Macro / Cronus LFSR timing streak detector
- **Penalties** — DISARMED / DMG_NULL / FROZEN / SHAMED; public shame broadcast; 64K replay capture ring

---

### Phase 9.2 — Advanced Peripherals (Raw HID)

- IR camera blob centroid tracking → head yaw/pitch
- DualShock 4 / DualSense HID report parsing
- DualSense adaptive trigger resistance byte offsets
- Force-feedback output reports via `WriteFile`
- Racing wheel FFB support

---

### Phase 9.3 — Cinematic Director & Export

- **Cinematic Sequencer** — Catmull-Rom spline camera / entity tracks, stepped game-event / audio-cue tracks, offline FPS lock, VirtualAlloc track pool
- **CPU Path Tracer** — BVH SAH builder, Möller–Trumbore triangle intersection, cosine-weighted hemisphere sampling, multi-threaded tile render, ACES tonemap output
- **Video Exporter** — WMF `IMFSinkWriter`, H.264 hardware encode, RGB32 lossless, RGBA → BGRA Y-flip for correct orientation

---

### Phase 9.4 — MasterScript Language & VM

- **Bytecode VM** — stack-based, 33 opcodes, FNV-1a string pool, `VMFrame` call stack, native function registration, hot-reload
- **Compiler** — recursive-descent tokeniser, Pratt precedence climbing, backpatch jump fixups, two-pass function compilation

---

### Phase 9.5 — Tactical AI Toolset

- **NavMesh** — voxel rasterise → BFS region-grow → quad polygon mesh → A* pathfinding → funnel string-pull
- **Environment Query System (EQS)** — GRID / RING / NAVMESH generators; DISTANCE / DOT / LOS / COVER / HEIGHT / RANDOM scoring tests; weighted multi-criteria evaluation
- **Squad AI** — SUPPRESS / FLANK / LEAPFROG / PERIMETER / BREACH tactics; COLUMN / LINE / WEDGE / DIAMOND / STAGGER formations; shared blackboard; cascade-collapse member removal

---

### Phase 9.6 — Audio Synthesis & Music

- **Synthesizer** — SINE / SAW / SQUARE / TRI / NOISE / CUSTOM oscillators; ADSR envelopes; biquad LP / HP / BP / NOTCH filters; LFO targeting pitch / amplitude / filter / pan; 64-voice pool; Bhaskara I sin/cos; tanh soft-clip master limiter
- **DAW Tracker** — 64 patterns × 64 rows × 16 channels; FX_SLIDE / VIBRATO / ARPEGGIO / VOLUME / SPEED effects; mix node graph with per-node intensity for dynamic/adaptive music; BPM tick accumulator

---

### Phase 9.7 — Environmental Destruction

- **Voronoi Fracture** — nearest-site triangle assignment, material-specific chunk physics, debris lifetime despawn, deterministic Wang-hash site generation
- **Structural Integrity Simulation** — 512-node load-bearing graph; iterative load propagation; tension / compression edge failure thresholds; BFS cascade collapse; progressive material damage accumulation

---

### Phase 10 — Internal Developer Tools (`dev` build only)

All Phase 10 systems are compiled only with `build.bat dev`. Release builds produce zero bytes from this phase.

- **Three build configs** — `debug` / `dev` / `release`; the `MASTER_DEV_BUILD` macro gates all tooling
- **Noclip** — free-fly camera override; toggle with Ctrl+F1
- **ESP** — per-entity AABB wireframe outlines; toggle with Ctrl+F2
- **Combat Audit** — weapon-fire event subscriber with fading alpha HUD; toggle with Ctrl+F3
- **Hidden Dev Console** — activated by LCTRL + RSHIFT + ALT + INSERT (never logged, never listed in any UI); compiles and executes MasterScript input live; private 32-line ring buffer output

---

### Phase 12 — Qixual Texture Suite

- **Texture Paint Engine** — UV-space CPU brush, 5 channels RGBA8 (VirtualAlloc), 4 paint modes + stamp, smoothstep hardness curve, 32-slot undo ring, `.qixtex` export / import
- **Multi-Layer Compositor** — 16 layers, 8 blend modes (Normal / Multiply / Screen / Overlay / Add / Subtract / Darken / Lighten), R8 mask per layer, back-to-top composite, MergeDown / FlattenAll, `.qixmat` save / load
- **Qixual Asset Library** — 512 asset slots, 5 categories, `FindFirstFileA` directory scanner, suffix grouping, apply-to-mesh, search, manifest tracking
- **Texture Flow Graph** — 14 node types (noise, ramp, height-to-normal, Fresnel, UV transform, blend, output), Kahn topo-sort evaluation, 64×64 eval pool, 32×32 node previews, full-resolution bake
- **Texture Scripting** — 12 `tex_*` native functions registered into the MasterScript VM

---

### Phase 13 — Project Launcher

- Dark-themed Win32 popup wizard — runs before any engine system initialises
- Input fields: project name, studio name, description, game type (7 templates), project path with folder browser
- Owner-drawn accent-coloured Launch button
- **Scaffold generator** — creates full directory tree, pre-filled `scripts/main.ms`, type-specific script stubs (player, NPC, weapon, unit AI), `project.qix` config file, `README.txt`
- Main window title bar displays project name after launch

---

## Directory Layout

```
MasterEngine/
├── build.bat                   — build script (debug / dev / release)
├── main.cpp                    — WinMain entry point
├── bin/
│   └── MasterEngine.exe        — compiled output
├── engine/
│   ├── core/                   — types, memory, math, str, collections, file, log
│   ├── platform/               — win32_platform, hid, launcher
│   ├── editor/                 — ui, ui_renderer, editor, camera, asset_browser,
│   │                             worldspace_ui, flow_graph, project, accessibility,
│   │                             sequencer, video_export, foliage,
│   │                             texture_paint, tex_layer, qixual, tex_graph, tex_script
│   ├── render/                 — dx12_core, shader, texture, mesh, material, renderer,
│   │                             render_graph, grd, lighting, raytracing, stp,
│   │                             compute_pipeline, path_tracer
│   ├── scene/                  — ecs, scene
│   ├── physics/                — physics, kcc, fracture, structural
│   ├── audio/                  — audio, spatial, synth, daw, wasapi_guids
│   ├── script/                 — reflect, event, hotreload, vm, compiler
│   ├── ai/                     — statemachine, btree, worldevents, navmesh, eqs, squad
│   ├── weapon/                 — weapon_proc, attachment
│   ├── net/                    — net, anticheat
│   └── dev/                    — dev_tools  (MASTER_DEV_BUILD only)
└── shaders/
    ├── basic_vs.hlsl / basic_ps.hlsl
    ├── lighting_pass.hlsl / gtao.hlsl / ssr.hlsl
    ├── taa.hlsl / bloom.hlsl / dof.hlsl / tonemap.hlsl / postfx.hlsl
    ├── raytracing.hlsl / denoise.hlsl
    ├── cull_instances.hlsl / hiz_build.hlsl / particles.hlsl
    └── skinning.hlsl
```

---

## Linker Dependencies

Only OS-supplied libraries — nothing to install.

```
kernel32.lib    user32.lib      gdi32.lib       xinput.lib
d3d12.lib       dxgi.lib        d3dcompiler.lib
ole32.lib       ws2_32.lib      comdlg32.lib
mf.lib          mfreadwrite.lib mfplat.lib      mfuuid.lib
mmdevapi.lib    uuid.lib
```

---

## Design Goals

- **No runtime surprises** — every allocation is explicit; no hidden heap use inside engine systems
- **Fast iteration** — shader hot-reload, game DLL hot-reload, and MasterScript hot-reload all work without restarting
- **Offline-capable** — no cloud services, no telemetry, no internet connection required
- **Shippable by default** — release builds strip all developer tooling at the preprocessor level; zero overhead

---

## Roadmap

- [ ] HLSL shader compilation pipeline (PSO cache warm-up)
- [ ] Physical-based animation (IK, ragdoll, blend trees)
- [ ] Streaming world / level-of-detail system
- [ ] Linux/Vulkan port layer
- [ ] Multiplayer lobby & matchmaking layer
- [ ] Editor undo/redo system
- [ ] Prefab system
- [ ] Console platform support

---

## License

Source available. All rights reserved — contact for licensing.

---

*Built with no dependencies. Just a compiler and the Windows SDK.*
