# cs2-hammer-mcp

Research and design work for an MCP server that can create, inspect, modify, compile and test Counter-Strike 2 Hammer maps.

> **Status:** research / architecture phase. No working MCP server is implemented yet.

## Goal

The long-term goal is to let an MCP-capable AI client perform tasks such as:

- create a new CS2 addon/map from a known-good template;
- inspect an existing `.vmap`;
- add or modify entities such as player spawns, lights, props and triggers;
- create simple editable Hammer geometry;
- validate map structure and entity properties;
- compile a map with the CS2 Workshop Tools;
- launch CS2 in tools/development mode and load the compiled map;
- eventually support higher-level operations such as “build a symmetric 5v5 aim map with two spawn areas and central cover”.

The initial research shows that this is technically realistic, but one important assumption must be corrected:

**Hammer `.vmap` source files are not KV3. They are Valve Datamodel/DMX documents.**

Current CS2 Hammer normally saves `.vmap` as binary DMX. Hammer can also save a text representation using the KeyValues2 DMX encoding. Valve ships `dmxconvert.exe`, which can convert between those representations. This gives us a practical automation path without trying to reverse-engineer a proprietary binary format from scratch.

## Proposed pipeline

```text
AI client
   |
   | MCP
   v
cs2-hammer-mcp
   |
   +-- inspect / create / modify map model
   |
   +-- dmxconvert.exe
   |      binary .vmap <-> KeyValues2 text DMX
   |
   +-- CS2 FGD / entity metadata
   |
   +-- resourcecompiler.exe
   |      .vmap -> compiled map VPK
   |
   +-- optional CS2/VConsole integration
          compile -> launch -> load -> inspect errors
```

For a first implementation, the safest approach is **not** arbitrary free-form mesh generation. Start by cloning a tiny, known-good map/template and provide deterministic operations for entities and simple geometry. This is the same general strategy proven by the Dota 2 Workshop MCP project, whose map pipeline also converts DMX to text for editing and compiles it afterward.

## Key findings

### 1. `.vmap` is DMX, not KV3

A CS2 `.vmap` is a Valve Datamodel (DMX) object graph. A current map may have a header similar to:

```text
<!-- dmx encoding binary 9 format vmap 35 -->
```

A text copy saved by Hammer is represented as KeyValues2 DMX, for example:

```text
<!-- dmx encoding keyvalues2 4 format vmap 35 -->
```

The exact `vmap` format version changes with tool/engine updates and **must not be hard-coded unless necessary**. We should prefer preserving the source format version or letting Valve's tools up-convert it.

### 2. Valve already ships a converter

The CS2 Workshop Tools include:

```text
game/bin/win64/dmxconvert.exe
```

Typical operations:

```powershell
dmxconvert.exe -i map.vmap -o map_text.vmap -oe keyvalues2

dmxconvert.exe -i map_text.vmap -o map.vmap -oe binary
```

This is likely the best initial implementation strategy: let Valve parse and serialize its own binary DMX and edit the readable form ourselves or through a DMX library.

### 3. Open-source DMX implementations exist

The strongest candidate for a structured implementation is **Datamodel.NET**, currently used by ValveResourceFormat / Source 2 Viewer. It supports reading/writing modern binary and text DMX, and ValveResourceFormat already contains typed classes and reconstruction logic for Source 2 maps.

Other useful implementations/references include:

- `ValveResourceFormat/Datamodel.NET` — C#, read/write DMX; strong candidate for a native structured backend.
- `ValveResourceFormat/ValveResourceFormat` — extensive Source 2 reverse engineering, map decompiler, Hammer mesh reconstruction and typed map structures.
- `leops/dmxparser` — Rust reader with VMAP structures; currently primarily useful as a reference because write support is limited.
- `datamodel.py` / Blender Source Tools — Python DMX implementation.
- `sourcepp` — C++ Source format library with Python bindings including DMX support.

### 4. Compiling can be automated

CS2 Workshop Tools ship `resourcecompiler.exe` in:

```text
game/bin/win64/resourcecompiler.exe
```

A map source under an addon generally lives at:

```text
content/csgo_addons/<addon>/maps/<name>.vmap
```

The compiled runtime result is written under the corresponding game addon tree and ultimately packaged as a map VPK. Hammer's Build Map dialog invokes the Resource Compiler with stage-specific arguments. The compiler can also be driven directly from the command line.

A representative modern CS2 map compile command observed in real Workshop Tools output contains stages/options such as:

```text
resourcecompiler.exe ... -i <map.vmap> -world -bakelighting -phys -vis -nav ... -retail -nop4
```

The MCP should not invent a single universal compile command. It should model build stages and either:

1. capture/replicate the current command produced by the installed Workshop Tools, or
2. use a conservative documented/minimal command and expose additional stages explicitly.

### 5. Entity support should be FGD-driven

FGD files describe the entities Hammer exposes, their editor properties, inputs and outputs. For CS2 they are available in the installed game/tool content (for example common Source 2 FGDs under `game/core` plus game-specific definitions).

This makes it possible to build tools such as:

```text
entity_catalog
entity_describe
entity_validate
map_add_entity
map_update_entity
map_remove_entity
```

The FGD is editor metadata rather than engine truth, so validation should distinguish:

- editor definition from FGD;
- runtime schema information where available;
- empirical CS2-specific behavior.

### 6. Hammer geometry is the difficult part

Entities are comparatively straightforward. Arbitrary editable Hammer meshes are substantially more complex.

Source 2 maps are mesh-based rather than Source 1 BSP brush maps. ValveResourceFormat's `HammerMeshBuilder` shows that editable Hammer meshes are represented through a half-edge mesh with data streams for positions, texture coordinates, normals, tangents, materials and other per-vertex/per-face data.

This means an MVP should support geometry in increasing levels of difficulty:

1. **template maps + entities**;
2. **clone/transform existing geometry/prefabs**;
3. **simple generated primitives** (box, floor, wall, ramp);
4. **general polygonal mesh generation**;
5. higher-level procedural map generation.

Trying to begin at step 4 would create unnecessary risk.

## Recommended implementation direction

A practical first implementation should be Windows-first because CS2 Workshop Tools and the authoritative conversion/compile utilities are Windows executables.

A good architecture is:

- MCP server: TypeScript/Node.js or C#;
- discovery layer: locate Steam library + CS2 Workshop Tools;
- addon layer: enforce work only inside configured `content/csgo_addons/...` roots;
- DMX adapter: initially `dmxconvert.exe` + text-DMX transformations;
- later structured backend: Datamodel.NET;
- entity metadata: parse installed FGDs, cache a searchable catalog;
- compile layer: wrapper around `resourcecompiler.exe` with dry-run and captured logs;
- optional runtime layer: CS2/VConsole command and log integration;
- test corpus: tiny maps saved by the current version of Hammer.

The MCP should always support `dryRun` for filesystem and compiler operations and should create backups or use atomic writes when changing maps.

## Proposed MVP tools

```text
cs2_doctor
addon_list
addon_create
map_list
map_info
map_to_text
map_from_text
map_validate
entity_catalog
entity_describe
map_add_entity
map_update_entity
map_remove_entity
map_compile
```

Second stage:

```text
map_create_from_template
map_add_spawn
map_add_light
map_add_prop
map_add_box
map_add_floor
map_add_wall
map_transform_object
map_launch
cs2_console_command
cs2_read_console
```

## Research documents

See:

- [`docs/RESEARCH.md`](docs/RESEARCH.md) — detailed technical research and evidence.
- [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md) — proposed implementation architecture and MCP surface.
- [`docs/ROADMAP.md`](docs/ROADMAP.md) — phased implementation/test plan.
- [`docs/SOURCES.md`](docs/SOURCES.md) — source list and provenance notes.

## Important provenance note

A significant portion of our understanding of Source 2 formats comes from community reverse engineering. In particular, this research relies heavily on Source 2 Viewer / ValveResourceFormat and its related Datamodel.NET work.

Powered by [Source 2 Viewer](https://s2v.app) ([ValveResourceFormat](https://github.com/ValveResourceFormat/ValveResourceFormat)).

This repository is not affiliated with Valve.