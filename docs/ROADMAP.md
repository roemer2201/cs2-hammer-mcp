# Roadmap

This roadmap deliberately starts with read-only inspection and entity editing before attempting arbitrary Hammer geometry.

## Phase 0 — establish a current CS2 fixture corpus

Goal: replace assumptions with current Hammer output.

Tasks:

- install/verify current CS2 Workshop Tools;
- record CS2/tool build information;
- create a tiny addon dedicated to tests;
- generate small one-change-at-a-time maps in Hammer;
- export/save text copies of each map;
- run `dmxconvert.exe` binary -> KeyValues2 -> binary round trips;
- record current Resource Compiler command lines for Hammer's build presets;
- collect current CS2/common FGD paths.

Suggested fixtures:

```text
fixtures/current/00-empty.vmap
fixtures/current/01-ct-spawn.vmap
fixtures/current/02-t-spawn.vmap
fixtures/current/03-light.vmap
fixtures/current/04-prop-static.vmap
fixtures/current/05-trigger.vmap
fixtures/current/06-box.vmap
fixtures/current/07-box-moved.vmap
fixtures/current/08-box-resized.vmap
fixtures/current/09-box-material.vmap
fixtures/current/10-prefab.vmap
fixtures/current/11-buyzone.vmap
fixtures/current/12-bombsite.vmap
```

Acceptance criteria:

- every fixture loads in Hammer;
- every fixture compiles;
- text exports are version-controlled and explainable;
- differences between consecutive fixtures are documented.

## Phase 1 — MCP skeleton and environment discovery

Implement:

```text
cs2_doctor
cs2_paths
addon_list
map_list
```

Capabilities:

- locate Steam + CS2;
- validate Workshop Tools installation;
- detect `dmxconvert.exe` and `resourcecompiler.exe`;
- find addon source/runtime roots;
- report versions/build metadata where available;
- enforce path sandboxing.

Acceptance criteria:

- clear output on a valid install;
- clear actionable errors on missing Workshop Tools;
- no writes outside configured addon root.

## Phase 2 — read-only VMAP support

Implement:

```text
map_info
map_to_text
map_list_objects
map_list_entities
map_get_object
map_validate
```

Start by using `dmxconvert.exe` as authoritative binary/text adapter.

Build or integrate a real KeyValues2 DMX parser that understands:

- typed scalar attributes;
- arrays;
- nested elements;
- top-level referenced elements;
- GUID references.

Acceptance criteria:

- read all Phase-0 fixtures;
- resolve object references;
- preserve/print source VMAP encoding + format version;
- structural validator catches broken references/duplicate IDs.

## Phase 3 — FGD index

Implement:

```text
entity_catalog
entity_describe
entity_validate
```

Tasks:

- discover installed common + CS2 FGD files;
- parse definitions/inheritance/includes;
- index classnames, properties, inputs and outputs;
- retain source-file provenance;
- cache by file hash/mtime.

Acceptance criteria:

- common test entities found with expected properties;
- invalid property names/types produce useful diagnostics;
- no hard-coded frozen CS2 entity list required for normal operation.

## Phase 4 — safe entity mutation

Implement:

```text
map_add_entity
map_update_entity
map_remove_entity
map_move_entity
```

Requirements:

- stable element GUID handling;
- transactional writes;
- backup/atomic replace;
- `dryRun` support;
- unknown attributes preserved;
- selection/editor metadata not destructively normalized where avoidable.

Add convenience wrappers after generic operations work:

```text
map_add_ct_spawn
map_add_t_spawn
map_add_light
map_add_prop
```

Acceptance criteria for every mutation:

1. generated VMAP parses;
2. `dmxconvert` round trip succeeds;
3. Hammer opens map;
4. entities-only compile succeeds where applicable;
5. map loads in CS2.

## Phase 5 — compiler integration

Implement:

```text
map_compile
compile_preset_list
compile_preset_import
```

Presets:

```text
entities-only
fast
full
custom
```

Return:

- exact command;
- stage selection;
- stdout/stderr;
- exit code;
- warnings/errors;
- output VPK path.

Acceptance criteria:

- Phase-0 maps compile from MCP without Hammer open;
- entities-only build is supported;
- `dryRun` returns exact intended command without execution.

## Phase 6 — template-driven map creation

Implement:

```text
addon_create
map_create_from_template
map_clone
```

Ship or generate only original/minimal fixtures with licensing/provenance understood. Another option is to let the user nominate a local template map.

Initial generated maps should rely on known-good geometry and support placing entities/props/prefabs.

Acceptance criteria:

- create new addon/map from scratch or template;
- compile and load it;
- AI can create a playable spawn test map without manual Hammer editing.

## Phase 7 — primitive Hammer geometry

This phase should preferably use a structured DMX backend (likely Datamodel.NET/C#) and a dedicated topology builder.

Implement in order:

```text
map_add_floor
map_add_wall
map_add_box
map_add_ramp
map_transform_object
map_set_material
```

For each primitive, define deterministic:

- winding;
- half-edge topology;
- material slots;
- UVs;
- normals/smoothing;
- collision expectations.

Acceptance criteria:

- primitive survives Hammer load/save without corruption;
- compiled rendering is correct;
- collision is correct;
- material/UV behavior is predictable;
- transformations are reversible/repeatable.

## Phase 8 — runtime feedback loop

Research and then implement CS2-specific runtime integration.

Potential tools:

```text
map_launch
cs2_console_command
cs2_read_console
cs2_screenshot
cs2_smoke_test
```

Acceptance criteria:

- MCP can load a compiled addon map;
- compile/runtime errors are surfaced automatically;
- basic smoke test confirms both teams can spawn.

Do not assume Dota 2 VConsole details apply unchanged; verify CS2 behavior first.

## Phase 9 — higher-level procedural construction

Only after primitive geometry is proven:

```text
map_add_cover
map_add_corridor
map_add_room
map_make_symmetric
map_create_aim_layout
map_measure_path_lengths
map_check_spawn_lines_of_sight
```

A high-level request such as:

```text
Create a 40 m x 20 m symmetric aim arena with five spawns per team,
a central divider and two offset cover blocks on each side.
```

should compile to a transparent sequence of primitive/entity operations that can be inspected, validated and retried.

## Phase 10 — generic mesh generation

This is intentionally late.

Possible API:

```text
map_add_mesh(vertices, polygons, material, uvMode, ...)
```

Required first:

- robust topology validator;
- deterministic UV/normals/tangents;
- non-manifold detection;
- scale limits;
- compile/Hammer round-trip tests;
- clear handling of physics/collision metadata.

## Suggested repository structure

```text
src/
  index.ts
  mcp/
  cs2/
    discovery.ts
    addon.ts
    compiler.ts
    runtime.ts
  dmx/
    backend.ts
    dmxconvert-backend.ts
    parser/
  fgd/
  vmap/
    model.ts
    entities.ts
    validation.ts
    geometry.ts
  security/
  util/

native/
  Cs2HammerMapHelper/      # optional future C#/Datamodel.NET helper

test/
  unit/
  integration/

fixtures/
  current/
  historical/

docs/
```

## First implementation milestone worth releasing

A genuinely useful v0.1 does **not** need procedural mesh generation.

A strong v0.1 target is:

- auto-detect Workshop Tools;
- inspect VMAP files;
- search current FGD entities;
- add/move/remove point entities;
- add CT/T spawns, lights and props;
- validate;
- compile entities-only or fast/full;
- return compiler diagnostics.

That already allows an AI agent to make many meaningful changes to existing CS2 maps while keeping geometry authoring in Hammer.
