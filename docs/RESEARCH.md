# Research: CS2 Hammer / VMAP automation

Research date: 2026-09-12

This document records the current technical understanding for a future `cs2-hammer-mcp` implementation. It deliberately separates facts that are directly evidenced by tools/source code from design assumptions that still need end-to-end testing against a locally installed copy of the current CS2 Workshop Tools.

## 1. Executive summary

The project is feasible.

The most important finding is that uncompiled Hammer map sources (`.vmap`) are **DMX / Valve Datamodel documents**, not KV3. Current CS2 Hammer commonly stores them using DMX binary encoding, while Hammer can export the same document as human-readable KeyValues2 DMX. Valve ships `dmxconvert.exe`, allowing lossless binary/text conversion.

This significantly lowers the risk of an MCP implementation because the first version does not need to implement the binary format itself.

The recommended automation loop is therefore:

1. locate the configured addon and `.vmap`;
2. convert binary `.vmap` to text DMX with Valve's `dmxconvert.exe`;
3. parse/manipulate the DMX object graph;
4. serialize text DMX;
5. convert back to binary if desired;
6. invoke `resourcecompiler.exe` using explicit build stages;
7. capture logs and surface errors to the MCP client;
8. optionally launch/load the map and interact with CS2 through VConsole or another supported development-console channel.

Entity editing is a relatively tractable first target. Arbitrary Hammer geometry is harder because editable geometry uses a half-edge mesh representation with multiple typed data streams and material/UV/normal metadata.

## 2. Correct file-format model

### 2.1 VMAP source files are DMX

Source 2 uses several serialization families. A crucial distinction is:

- most authored Source 2 assets such as `.vmat`, `.vpcf`, `.vdata` use text KV3;
- compiled `_c` resources typically carry binary KV3 data inside Source 2 resource containers;
- Hammer map source files `.vmap` use **DMX**;
- KeyValues2 is the *text encoding* of DMX, not KV3.

The Source2 Wiki documents current CS2 examples such as:

```text
<!-- dmx encoding binary 9 format vmap 35 -->
```

and the same map exported as text:

```text
<!-- dmx encoding keyvalues2 4 format vmap 35 -->
```

The exact VMAP format version is not stable across engine/tool revisions. Public examples already show different format versions. Consequently the MCP should inspect the header rather than assume one version.

### 2.2 DMX is an object graph

A DMX document consists of typed elements and typed attributes. Elements reference each other, forming a graph rather than a simple nested dictionary.

Typical element types visible in maps include:

- `CMapRootElement`
- `CMapWorld`
- `CMapEntity`
- `CMapMesh`
- `CMapGroup`
- `CMapPrefab`
- `CMapWorldLayer`
- `CMapSelectionSet`
- stored-camera/editor metadata types

Text DMX can inline elements that have a single reference. Elements referenced multiple times may be emitted as independent top-level blocks and referenced by GUID. Binary DMX stores the elements in a flat structure and uses indexes/references internally.

An MCP implementation must therefore use a real graph model or a parser that correctly resolves element references. Naive regex-only manipulation is unacceptable for general-purpose editing.

### 2.3 DMX primitive types matter

KeyValues2 DMX text includes explicit types. Common examples include:

```text
"origin" "vector3" "128 256 64"
"angles" "qangle" "0 90 0"
"referenceID" "uint64" "0x0"
"enabled" "bool" "1"
```

Other important types include arrays, colors, matrices, binary blobs and element references.

This means a typed parser has clear advantages over generic text editing.

## 3. Valve-provided conversion path

### 3.1 dmxconvert.exe

Source 2 Workshop Tools ship `dmxconvert.exe` under:

```text
game/bin/win64/dmxconvert.exe
```

Useful conversions include:

```powershell
dmxconvert.exe -i my_map.vmap -o my_map_text.vmap -oe keyvalues2
```

and:

```powershell
dmxconvert.exe -i my_map_text.vmap -o my_map.vmap -oe binary
```

The tool can also up-convert format versions.

### 3.2 Why use Valve's converter initially

Using `dmxconvert.exe` as an adapter gives us several advantages:

- Valve's own parser handles current binary DMX details;
- binary format-version churn does not have to be replicated immediately;
- generated text can be inspected and diffed in tests;
- failures are directly comparable with Hammer behavior;
- we can keep the MCP language independent from the map serializer.

It also gives us a strong compatibility test: every generated document should survive both directions through `dmxconvert.exe` and then load in Hammer.

### 3.3 Potential long-term replacement

A future implementation can operate directly on binary DMX if performance or fidelity justifies it. The leading candidate is Datamodel.NET because it already supports modern DMX encodings and is actively used by ValveResourceFormat.

The initial implementation should keep the DMX backend behind an interface so the converter-based implementation can later be replaced or complemented.

## 4. Existing open-source work

### 4.1 ValveResourceFormat / Source 2 Viewer

Repository:

https://github.com/ValveResourceFormat/ValveResourceFormat

This is currently the most important technical reference for Source 2 resource/map internals. It is based on reverse engineering and includes:

- parsing of compiled Source 2 resources;
- map decompilation;
- extraction of editable `.vmap` files;
- map entity reconstruction;
- editable Hammer mesh reconstruction;
- material and physics handling;
- support for current CS2.

The `MapExtract.cs` implementation creates a DMX `vmap` document and writes it using binary DMX encoding. It also builds `CMapEntity`, `CMapPrefab`, `CMapWorldLayer`, selection sets and editable meshes.

This is strong evidence that a programmatic VMAP writer is practical.

### 4.2 HammerMeshBuilder

`HammerMeshBuilder.cs` is particularly important.

Its own comments state that most of the work is handled through a half-edge mesh representation and that attribute data is attached through data streams. It handles data such as:

- vertex positions;
- texture coordinates;
- secondary texture coordinates;
- normals;
- tangents;
- vertex-paint blend parameters;
- vertex-paint tint color;
- per-face material indexes.

This confirms that arbitrary geometry creation is materially more complex than placing entities.

The MCP should either:

- reuse/adapt the relevant ValveResourceFormat data structures/algorithms, or
- implement a minimal geometry builder that produces valid half-edge meshes for a small primitive set.

Reinventing this without studying the existing implementation would be wasteful and risky.

### 4.3 Datamodel.NET

Repository:

https://github.com/ValveResourceFormat/Datamodel.NET

Source2 Wiki currently identifies it as a DMX implementation capable of reading/writing modern binary and KeyValues2 encodings. ValveResourceFormat uses it when exporting VMAP content.

This makes C# an attractive language for the eventual map-core library even if the MCP frontend is Node.js.

Possible integration options:

1. whole MCP in C#;
2. Node/TypeScript MCP server calling a small C# helper executable;
3. Node/TypeScript initially using `dmxconvert` + text parser, then adding C# for geometry.

Option 3 minimizes the initial dependency surface while preserving a migration path.

### 4.4 dmxparser (Rust)

Repository:

https://github.com/leops/dmxparser

It provides a VMAP-oriented reader and typed structures, but its documented limitations include binary-only read support and incomplete write support. It is useful as an independent specification/reference but is not the best current writer backend.

### 4.5 Dota 2 Workshop MCP

Repository:

https://github.com/ex3lite/Dota2_Workshop_MCP

This is a very important architectural precedent. It is a working MCP server for another Source 2 Workshop Tools environment and explicitly implements map operations by:

- treating `.vmap` files as DMX;
- converting them with Valve's `dmxconvert.exe`;
- editing the text representation;
- compiling with `resourcecompiler.exe`;
- exposing map tools through MCP.

Its documented limitation is also instructive: it avoids fully general bespoke geometry and instead builds maps from templates, terrain-specific data and entities.

For CS2, a similar staged approach is sensible.

## 5. CS2 addon/content layout

A normal CS2 addon uses the Source 2 split between authored content and compiled runtime content.

Typical source path:

```text
<CS2>/content/csgo_addons/<addon>/maps/<map>.vmap
```

Typical runtime/addon root:

```text
<CS2>/game/csgo_addons/<addon>/
```

Source 2 paths are semantically important. Asset references are stored by path, so moving a material/model or generating references outside the mounted content tree can cause failures.

The MCP should enforce a project root and avoid arbitrary filesystem writes by default.

Recommended project detection order:

1. explicit `CS2_ADDON_DIR` / MCP config;
2. explicit Steam/CS2 path;
3. Windows registry Steam location + `libraryfolders.vdf` discovery;
4. validate that `game/bin/win64/resourcecompiler.exe` and `dmxconvert.exe` exist;
5. validate matching `content/csgo_addons` and `game/csgo_addons` roots.

## 6. Compiling maps

### 6.1 resourcecompiler.exe

CS2 Workshop Tools ship:

```text
game/bin/win64/resourcecompiler.exe
```

General Source 2 asset compilation accepts an input via `-i` and determines the game/mod from the content path or an explicit `-game` option.

Maps require additional stage-specific switches.

### 6.2 Build stages

Current Source 2 documentation describes map builds as a sequence of stages rather than one monolithic compile. Stages may include world geometry, physics, visibility, baked lighting, navigation and acoustics/custom data.

A modern CS2 Hammer-generated command observed publicly in 2026 includes options such as:

```text
-world
-bakelighting
-lightmapMaxResolution ...
-lightmapVRadQuality ...
-phys
-vis
-nav
-sareverb
-sapaths
-sacustomdata
-retail
-nop4
```

The precise option set varies with the build preset and Workshop Tools version.

### 6.3 MCP compile design

Do **not** expose only one opaque `compile_map()` that silently picks expensive settings.

Prefer:

```json
{
  "map": "aim_test",
  "preset": "fast",
  "stages": {
    "world": true,
    "physics": true,
    "visibility": true,
    "lighting": false,
    "navigation": false,
    "audio": false
  },
  "dryRun": false
}
```

Possible presets:

- `entities-only`
- `fast`
- `full`
- `custom`

The exact command should always be returned to the caller along with exit code, stdout/stderr and detected error messages.

### 6.4 Entities-only iteration

Source 2 documentation explicitly notes that an entities-only build is useful for changes such as adding spawns. This is an important optimization for an AI iteration loop: entity manipulation should not trigger full lighting/world recompilation.

## 7. Entity model and FGD integration

### 7.1 FGD role

FGD files describe what Hammer exposes to map authors:

- entity class names;
- editor-visible properties;
- types/defaults;
- inputs;
- outputs;
- descriptions;
- helper visualization metadata.

Common Source 2 definitions exist under locations such as:

```text
game/core/base.fgd
game/core/lights.fgd
game/core/models_base.fgd
```

CS2 also contributes game-specific entity metadata.

### 7.2 FGD is not runtime truth

FGD metadata does not create engine entities. The actual runtime class/field behavior comes from engine/game code and Source 2's runtime schema system.

Therefore the MCP should attach a confidence/source field to validation results:

```text
FGD-defined
schema-observed
map-observed
community-documented
unknown
```

### 7.3 Initial entity tool surface

Recommended:

```text
entity_catalog(query?, category?)
entity_describe(classname)
map_list_entities(map, classname?)
map_add_entity(map, classname, origin, angles?, properties?)
map_update_entity(map, selector, patch)
map_remove_entity(map, selector)
entity_validate(classname, properties)
```

Selectors should use stable element GUIDs wherever possible rather than array positions.

### 7.4 CS2-specific convenience tools

High-level helpers should wrap generic entity editing:

```text
map_add_t_spawn
map_add_ct_spawn
map_add_light
map_add_prop
map_add_bombsite
map_add_buyzone
```

These should be implemented only after the actual current CS2 entity names/properties are harvested from installed FGDs and test maps.

## 8. Geometry

### 8.1 Source 2 is mesh-oriented

Source 2 maps are not Source 1 BSP brush maps. Editable Hammer geometry is represented as mesh data.

This changes the problem substantially. A simple cuboid still needs valid topology plus its data streams.

### 8.2 Half-edge representation

ValveResourceFormat's reconstruction code demonstrates a half-edge representation. At a high level, a generated mesh needs consistent relationships among:

- vertices;
- directed half edges;
- opposite edges;
- next edges around each face;
- face ownership;
- vertex/edge/face data indexes;
- materials;
- per-corner/per-vertex attribute streams.

A malformed relationship may load incorrectly, fail compilation or produce broken normals/collision.

### 8.3 Recommended geometry progression

#### Phase A: no new topology

- clone known-good template maps;
- place entities;
- place/transform props;
- instantiate prefabs;
- duplicate/translate known existing geometry if structure remains valid.

#### Phase B: primitives

Implement deterministic generators for:

- floor rectangle;
- vertical wall;
- axis-aligned box;
- ramp;
- simple wedge.

Each primitive should be tested with:

1. DMX serialization;
2. `dmxconvert` round trip;
3. Hammer load/save;
4. Resource Compiler build;
5. in-game collision/material verification.

#### Phase C: general meshes

Only after primitives are proven should the project expose a generic `map_add_mesh` or procedural mesh API.

### 8.4 Materials and UVs

A geometry tool cannot stop at positions/faces. It must specify at least a valid material and sensible texture coordinates. Normal/tangent generation may be required depending on how Hammer expects the mesh data streams.

A practical primitive API therefore needs inputs like:

```text
material
uvScale
hardEdges / smoothing
collision intent
```

with safe defaults based on known CS2 dev/tool materials.

## 9. Runtime testing / VConsole

A useful MCP should eventually close the loop after compile.

Potential flow:

```text
map edit
 -> validate
 -> compile
 -> launch/load
 -> read console
 -> detect errors
 -> screenshot or query state
 -> fix
```

The Dota 2 Workshop MCP demonstrates that VConsole2 can be used for a reliable Source 2 live-debug loop. CS2 requires its own verification because available commands, ports and launch flags can differ.

Until verified, runtime/VConsole support should be treated as a research item rather than an assumed feature.

## 10. Validation strategy

The MCP should validate at several layers.

### 10.1 Structural

- DMX parses successfully;
- root is a `CMapRootElement`;
- world element exists;
- element IDs are unique;
- references resolve;
- required attributes have expected types.

### 10.2 Entity

- classname exists in current FGD catalog;
- property names/types match FGD where known;
- model/material asset references resolve inside mounted content;
- team/gameplay constraints can be checked by higher-level rules.

### 10.3 Geometry

- all topology indexes are in range;
- half-edge pairs/loops are consistent;
- faces have >= 3 vertices;
- material indices are valid;
- no obvious zero-area faces;
- positions are finite;
- UV/normal streams have expected cardinality.

### 10.4 Toolchain

- `dmxconvert` round trip succeeds;
- Resource Compiler accepts the map;
- Hammer can open the generated VMAP without repair prompts/errors;
- final map loads in CS2.

## 11. Test corpus needed

Before implementing a writer, collect tiny Hammer-authored fixtures with one conceptual change per file.

Suggested corpus:

```text
00-empty-room.vmap
01-one-ct-spawn.vmap
02-one-t-spawn.vmap
03-one-light.vmap
04-one-prop-static.vmap
05-one-trigger.vmap
06-one-box.vmap
07-box-moved.vmap
08-box-resized.vmap
09-box-material-changed.vmap
10-two-boxes.vmap
11-prefab-instance.vmap
12-bombsite.vmap
13-buyzone.vmap
```

For every fixture, keep a text-DMX copy generated by `dmxconvert` or Hammer's `Save Copy As Text` and diff it against the previous fixture.

This will reveal the actual current CS2 editor output instead of relying on older examples.

## 12. Risk assessment

### Low risk

- MCP protocol itself;
- project/addon discovery;
- invoking `dmxconvert`;
- invoking `resourcecompiler`;
- listing maps/addons;
- read-only DMX inspection;
- entity catalog from FGD.

### Medium risk

- safe entity creation/deletion across current VMAP versions;
- I/O connection editing;
- selection-set/editor metadata preservation;
- prefab manipulation;
- runtime VConsole integration for CS2.

### High risk

- arbitrary Hammer mesh creation;
- UV/tangent/normal correctness for general meshes;
- physics/collision intent across all geometry/entity cases;
- guaranteeing forward compatibility with future Hammer VMAP versions.

The architecture should isolate the high-risk geometry layer from the rest of the MCP.

## 13. Key conclusions

1. A CS2 Hammer MCP is technically plausible now.
2. It does **not** require blindly reverse-engineering binary `.vmap` from scratch.
3. Valve's own `dmxconvert.exe` provides an authoritative conversion bridge.
4. Datamodel.NET and ValveResourceFormat provide deep, reusable technical knowledge.
5. The first useful MCP can focus on templates, entities, props, validation and compilation.
6. Full procedural geometry should be a later milestone backed by a fixture/diff corpus and Hammer round-trip tests.
7. Current installed CS2 FGDs and current Hammer-produced VMAP fixtures should be treated as the local source of truth for version-sensitive behavior.

## 14. Provenance

A substantial part of the low-level Source 2 understanding in this research comes from community reverse engineering, especially ValveResourceFormat / Source 2 Viewer and Datamodel.NET.

Powered by [Source 2 Viewer](https://s2v.app) ([ValveResourceFormat](https://github.com/ValveResourceFormat/ValveResourceFormat)).
