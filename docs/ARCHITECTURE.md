# Proposed architecture

This document proposes an implementation architecture for `cs2-hammer-mcp` based on the current research.

## Design goals

The project should prioritize:

- correctness over cleverness;
- using Valve-provided tools where possible;
- preserving unknown VMAP data rather than rewriting it destructively;
- explicit dry-run behavior for destructive/build operations;
- compatibility with current CS2 Workshop Tools rather than assuming a fixed historical format;
- a small, deterministic MCP surface that can later grow into higher-level map generation.

## 1. High-level components

```text
+---------------------+
| MCP client          |
| ChatGPT/Claude/etc. |
+----------+----------+
           |
           | stdio / MCP transport
           v
+-------------------------------+
| cs2-hammer-mcp                |
|                               |
|  Tool layer                   |
|  Project/addon discovery      |
|  VMAP service                 |
|  Entity/FGD service           |
|  Geometry service             |
|  Compiler service             |
|  Runtime/VConsole service     |
|  Validation + diagnostics     |
+---------------+---------------+
                |
                +--> dmxconvert.exe
                +--> resourcecompiler.exe
                +--> installed FGD files
                +--> content/csgo_addons/...
                +--> game/csgo_addons/...
                +--> optional CS2 tools process
```

## 2. Language choice

Two approaches are attractive.

### Option A: C# MCP server

Pros:

- direct reuse of Datamodel.NET;
- easiest path to reuse/adapt ValveResourceFormat typed VMAP classes and mesh-generation logic;
- strong typing for DMX object graphs.

Cons:

- smaller MCP ecosystem than TypeScript;
- harder to borrow implementation patterns from existing Node-based Workshop MCP projects.

### Option B: TypeScript MCP frontend + C# map helper

Pros:

- excellent MCP ecosystem;
- can model tools rapidly;
- easy process management around Valve executables;
- lets a C# helper own DMX/mesh complexity.

Cons:

- two runtimes;
- IPC contract must be maintained.

### Recommendation

Start with **TypeScript/Node.js** for the MCP server and use `dmxconvert.exe` for VMAP conversion. Keep the VMAP backend behind an abstraction. Add a C# helper using Datamodel.NET once primitive mesh generation is implemented.

This gives the fastest route to useful entity/build tools without locking the project out of a structured map backend later.

## 3. Core interfaces

### 3.1 ProjectLocator

Responsibilities:

- find Steam libraries;
- find the CS2 install;
- verify Workshop Tools binaries;
- locate `content/csgo_addons` and `game/csgo_addons`;
- resolve addon roots;
- prohibit writes outside configured roots unless explicitly allowed.

Suggested model:

```ts
interface Cs2Environment {
  steamRoot?: string;
  cs2Root: string;
  contentRoot: string;
  gameRoot: string;
  resourceCompiler: string;
  dmxConvert: string;
  cs2Exe?: string;
}
```

### 3.2 VmapBackend

```ts
interface VmapBackend {
  inspect(path: string): Promise<VmapSummary>;
  toText(path: string): Promise<string>;
  fromText(text: string, outputPath: string): Promise<void>;
  load(path: string): Promise<VmapDocument>;
  save(document: VmapDocument, path: string): Promise<void>;
}
```

Initial implementation:

```text
DmxConvertTextBackend
```

Later:

```text
DatamodelNetBackend
```

The rest of the MCP should not care whether binary serialization is done by Valve or by Datamodel.NET.

## 4. Editing strategy

### 4.1 Never regex-edit arbitrary VMAP text

A text DMX document is still a graph with typed references. Regex may be acceptable for narrow experiments but should not be the production writer.

A production implementation should parse text DMX into an AST / object graph that preserves:

- element type;
- element GUID;
- attribute name;
- attribute type;
- attribute value;
- element references;
- arrays;
- unknown attributes.

Unknown fields must survive round trips.

### 4.2 Stable object identity

Every object exposed to an MCP client should have a stable identifier, preferably the DMX element GUID.

Example response:

```json
{
  "id": "7c2f9e0a-5a2c-4f0e-9a1e-2a6e1f6a2b3c",
  "type": "CMapEntity",
  "classname": "info_player_counterterrorist",
  "origin": [128, 64, 16]
}
```

Mutation tools should take `id` or a well-defined selector, not list indexes.

## 5. Transactional file writes

Map editing should be transactional:

1. read original;
2. serialize modified document to a temporary file;
3. validate with parser/`dmxconvert`;
4. optionally compile in validation mode;
5. atomically replace the original;
6. retain configurable backup or recovery file.

Suggested safety defaults:

- `dryRun: true` supported by all mutating tools;
- no overwrite outside addon roots;
- preserve source VMAP until the generated file parses;
- maximum generated-file size guard;
- operation log with input selector and modified element IDs.

## 6. FGD service

The FGD subsystem should parse the currently installed files rather than ship a permanently frozen CS2 entity database.

Suggested data model:

```ts
interface EntityDefinition {
  classname: string;
  kind: "point" | "solid" | "base" | "unknown";
  description?: string;
  properties: EntityPropertyDefinition[];
  inputs: IoDefinition[];
  outputs: IoDefinition[];
  sourceFiles: string[];
}
```

Features:

- scan mounted/common CS2 FGD paths;
- resolve `@include` relationships if applicable;
- merge inherited/base classes;
- cache an index;
- expose full source provenance per property;
- provide fuzzy search.

Initial MCP tools:

```text
entity_catalog
entity_describe
entity_validate
```

## 7. VMAP entity service

The entity service should understand `CMapEntity` enough to:

- read classname;
- read/write entity properties;
- read/write origin and angles;
- add/remove entities from the correct world/group child collection;
- preserve editor metadata;
- eventually handle outputs/connections.

MCP operations:

```text
map_list_entities
map_add_entity
map_update_entity
map_remove_entity
map_move_entity
```

High-level helpers should be wrappers, not separate persistence code:

```text
map_add_ct_spawn -> map_add_entity(classname=...)
map_add_t_spawn  -> map_add_entity(classname=...)
```

## 8. Geometry service

### 8.1 Separate geometry from generic VMAP mutation

Geometry should be its own module because its invariants are much stricter.

```ts
interface GeometryBackend {
  addBox(spec: BoxSpec): Promise<MapElementRef>;
  addFloor(spec: FloorSpec): Promise<MapElementRef>;
  addWall(spec: WallSpec): Promise<MapElementRef>;
  addRamp(spec: RampSpec): Promise<MapElementRef>;
  validateMesh(mesh: MapMesh): ValidationResult;
}
```

### 8.2 Primitive-first strategy

Do not expose arbitrary topology initially.

Primitive builders can guarantee:

- closed face loops;
- correct opposite half edges;
- deterministic winding;
- valid material slots;
- known UV generation;
- predictable normals.

This makes a reliable `map_add_box` much more valuable than a generic but fragile `map_add_mesh`.

### 8.3 Potential C# implementation

The C# geometry helper can reuse concepts/code from ValveResourceFormat's Hammer mesh reconstruction. Any copied/adapted MIT-licensed code must retain required attribution/license notices.

A helper process could use JSON on stdin/stdout:

```json
{
  "operation": "addBox",
  "inputVmap": "...",
  "outputVmap": "...",
  "origin": [0,0,0],
  "size": [512,512,32],
  "material": "materials/dev/..."
}
```

## 9. Compiler service

### 9.1 Explicit stage model

```ts
interface CompileStages {
  world?: boolean;
  physics?: boolean;
  visibility?: boolean;
  lighting?: boolean;
  navigation?: boolean;
  audio?: boolean;
  customData?: boolean;
}
```

The service maps a validated stage selection onto the installed `resourcecompiler.exe` command line.

### 9.2 Command capture

Because Hammer options change over time, an advanced feature should allow users to import/capture the command Hammer itself generated and convert it into a reusable preset.

Possible tool:

```text
compile_preset_import(commandLine)
```

This avoids pretending historical flags are permanently authoritative.

### 9.3 Compiler response

Return structured diagnostics:

```json
{
  "ok": false,
  "exitCode": 1,
  "durationMs": 12345,
  "command": "...",
  "errors": [
    { "stage": "world", "message": "..." }
  ],
  "warnings": [],
  "stdoutTail": "...",
  "outputVpk": null
}
```

## 10. Runtime service

Treat runtime control as optional and capability-detected.

Potential capabilities:

- launch CS2 in tools/development mode;
- load addon/map;
- send console commands;
- read console output;
- take screenshots;
- run smoke checks.

Do not assume Dota 2's exact VConsole configuration applies to CS2. Verify CS2 launch flags/protocol behavior experimentally and document it in fixtures/tests.

## 11. MCP tool taxonomy

### Diagnostics

```text
cs2_doctor
cs2_paths
addon_list
map_list
```

### Read-only map inspection

```text
map_info
map_to_text
map_list_objects
map_list_entities
map_get_object
map_validate
```

### Entity editing

```text
entity_catalog
entity_describe
entity_validate
map_add_entity
map_update_entity
map_remove_entity
```

### Geometry

```text
map_add_floor
map_add_wall
map_add_box
map_add_ramp
map_transform_object
```

### Build

```text
map_compile
map_compile_status
```

### Runtime

```text
map_launch
cs2_console_command
cs2_read_console
cs2_screenshot
```

## 12. Higher-level agent-friendly tools

After low-level primitives are reliable, add semantic operations:

```text
map_create_aim_layout
map_make_symmetric
map_add_spawn_group
map_add_cover_grid
map_measure_distances
map_check_spawn_visibility
```

These should compile down to deterministic low-level operations and return the exact changes made.

## 13. Observability

Every MCP call should emit structured internal diagnostics including:

- resolved addon/map path;
- detected VMAP encoding/version;
- object IDs touched;
- external process command;
- process exit code;
- duration;
- output artifact path;
- validation status.

A `verbose` flag can expose more of this to clients without making normal responses noisy.

## 14. Testing architecture

### Unit tests

- KeyValues2 DMX parser/serializer;
- FGD parser;
- path sandboxing;
- selector matching;
- command construction;
- primitive topology generation.

### Golden fixture tests

For each tiny Hammer-authored fixture:

- parse;
- serialize;
- compare structural equivalence;
- convert with `dmxconvert`;
- verify important object/property diffs.

### Integration tests

On a Windows machine with Workshop Tools:

1. create/copy fixture addon;
2. edit VMAP;
3. run `dmxconvert` round trip;
4. compile;
5. assert VPK exists;
6. optionally load in CS2.

### Compatibility tests

Record:

- CS2 build ID;
- Hammer/editor build field when present;
- VMAP format version;
- `dmxconvert` version/hash;
- resource compiler version/hash.

This lets regressions be tied to Workshop Tools updates.

## 15. Security boundaries

MCP servers execute local processes and edit files, so default constraints matter.

Recommended safeguards:

- allow writes only under configured addon content root;
- reject `..` traversal and symlink escapes;
- executable paths must resolve to expected CS2 tool locations unless explicitly configured;
- command arguments passed as arrays, never shell-concatenated strings;
- output-size/time limits for compiler processes;
- `dryRun` for all command-executing operations;
- do not publish to Workshop automatically in early versions;
- treat downloaded/untrusted VMAP/FGD text as data, not executable shell input.

## 16. Versioning philosophy

Source 2 authoring formats can change. Therefore:

- inspect, don't assume, VMAP format versions;
- preserve unknown attributes;
- rely on installed FGD files;
- prefer Valve's current converter/compiler;
- maintain fixtures from multiple known CS2 versions;
- fail with a clear compatibility diagnostic when a new format cannot be safely handled.

This architecture favors forward survivability over hard-coded knowledge of one CS2 build.