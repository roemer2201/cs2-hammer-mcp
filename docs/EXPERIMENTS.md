# Local verification experiments

These experiments are intended to turn the research into verified CS2-specific knowledge on the exact Workshop Tools build installed on the development machine.

## Experiment 1 — toolchain inventory

Record:

```text
CS2 install path
Steam build ID
resourcecompiler.exe file version/hash
dmxconvert.exe file version/hash
cs2.exe file version/hash
Workshop Tools/DLC installed state
```

Expected paths include:

```text
<CS2>/game/bin/win64/resourcecompiler.exe
<CS2>/game/bin/win64/dmxconvert.exe
<CS2>/content/csgo_addons/
<CS2>/game/csgo_addons/
```

Outcome:

- baseline for compatibility reports and future regression tests.

## Experiment 2 — determine current VMAP encoding/version

Create a minimal map in Hammer and inspect its first line/bytes.

Then run:

```powershell
dmxconvert.exe -i minimal.vmap -o minimal_text.vmap -oe keyvalues2
```

Record both headers.

Questions:

- current binary encoding version?
- current `vmap` format version?
- does Hammer output a `$prefix_element$` section?
- what editor-build metadata is stored?

## Experiment 3 — binary/text/binary identity

Round trip:

```text
Hammer binary vmap
 -> dmxconvert keyvalues2
 -> dmxconvert binary
 -> Hammer open/save
```

Compare:

- logical elements;
- GUID preservation;
- format version;
- file size;
- Hammer warnings;
- compiler result.

Exact byte identity is not required. Structural equivalence and tool acceptance are.

## Experiment 4 — one entity at a time

Create map A with no test entity and map B with one entity.

Start with:

```text
info_player_counterterrorist
info_player_terrorist
light_omni2 or current recommended point light
prop_static
```

Export both as KeyValues2 DMX and diff them.

For every entity, record:

- new DMX element type(s);
- `classname` storage;
- origin/angles storage;
- entity-property container representation;
- editor-only attributes;
- element ID/reference behavior;
- selection-set changes.

Goal:

- derive the smallest safe mutation that reproduces Hammer's structure.

## Experiment 5 — delete/move entity

Create a map with one spawn.

Save variants:

```text
spawn-created
spawn-moved
spawn-rotated
spawn-deleted
```

Diff text DMX.

Goal:

- verify which attributes represent transforms;
- verify whether deletion requires changes outside the world's child list (e.g. selection sets).

## Experiment 6 — FGD source discovery

Enumerate all `.fgd` files in relevant mounted game directories.

Record:

- file names;
- includes;
- class count;
- CS2-specific definitions;
- duplicates/overrides;
- base-class inheritance.

Then verify known entities from Hammer's UI against the parsed index.

Goal:

- build the `entity_catalog` source of truth from the installed toolchain.

## Experiment 7 — capture Hammer compile presets

For the same tiny map, execute each Hammer Build Map preset and capture the exact `resourcecompiler.exe` command line.

At minimum:

```text
entities-only
fast / preview equivalent
full
```

Record which flags map to:

```text
world
visibility
entities
physics
lighting
LOD
navigation
Steam Audio
```

Also record output files in the map VPK.

Goal:

- avoid stale/historical compile flags.

## Experiment 8 — direct Resource Compiler build

Close Hammer and execute the captured command directly.

Verify:

- exit code;
- map VPK output path;
- whether addon/game path needs explicit `-game`/`-outroot`;
- which environment assumptions Hammer normally supplies;
- whether stdout/stderr alone are sufficient for diagnostics.

Goal:

- prove the MCP can build without GUI automation.

## Experiment 9 — entities-only iteration

Compile a complete tiny map once.

Then add only a spawn and run the entities-only build.

Verify:

- previous world geometry remains;
- previous lighting remains;
- new spawn appears in game;
- compiler duration is materially smaller than full build.

Goal:

- establish the fast agent iteration loop.

## Experiment 10 — first generated entity

Take a Hammer-authored text VMAP and programmatically insert a new entity using a real parser/AST.

Then:

1. serialize text DMX;
2. convert to binary via `dmxconvert`;
3. open in Hammer;
4. save in Hammer;
5. compare text export before/after Hammer save;
6. entities-only compile;
7. load in CS2.

Pass condition:

- Hammer accepts it without repair/error;
- compiler accepts it;
- entity behaves correctly.

## Experiment 11 — prefab instance

Create a simple local prefab map and an instance of it in another map.

Diff DMX and record the exact `CMapPrefab`/instance representation and target path behavior.

Goal:

- provide powerful layout construction before generic mesh generation.

## Experiment 12 — simple box anatomy

Create exactly one rectangular editable Hammer mesh with a known size/material.

Create variants changing one thing at a time:

```text
position
size X
size Y
size Z
material
UV scale
rotation
```

Export all as text DMX.

Map the fields and data streams needed for:

- vertex positions;
- face loops;
- edge/opposite/next relationships;
- material indices;
- UVs;
- normals;
- tangents;
- smoothing/hard-edge state.

Cross-reference against ValveResourceFormat `HammerMeshBuilder`.

Goal:

- derive a minimal deterministic `addBox()` writer.

## Experiment 13 — generated box

Use the derived structure to generate a box from scratch in a structured DMX writer.

Validation sequence:

```text
internal topology validator
 -> DMX serialize
 -> dmxconvert round trip
 -> Hammer load
 -> Hammer save
 -> full compile
 -> in-game render
 -> in-game collision
```

Do not proceed to generic meshes until this is repeatable.

## Experiment 14 — material/UV behavior

For the generated box, test:

- one material all faces;
- different material per face;
- explicit UV scaling;
- rotated UVs if represented;
- hard vs soft normals.

Goal:

- define safe default behavior for `map_add_box`, `map_add_wall` and `map_add_floor`.

## Experiment 15 — CS2 runtime console integration

Launch CS2/tools mode manually while inspecting available console/VConsole behavior.

Verify:

- supported launch flags;
- whether VConsole2 accepts external commands in the CS2 tools environment;
- port/protocol details if applicable;
- command for loading an addon map;
- reliable way to read compiler/runtime error output;
- screenshot options.

Goal:

- replace Dota-derived assumptions with CS2-specific facts.

## Experiment 16 — end-to-end AI smoke test

Once entity + box primitives exist, perform one scripted scenario:

```text
Create new map from template.
Add 5 CT spawns.
Add 5 T spawns.
Add a floor.
Add two cover boxes.
Add basic lighting.
Validate.
Compile.
Launch/load.
```

Record every MCP call, generated DMX diff and compiler output.

Success means the project has crossed from “file-format tooling” into a genuinely useful AI map-authoring loop.

## Data that should be committed after experiments

Prefer metadata and small original fixtures that are safe to redistribute. Do not commit Valve game assets unless licensing clearly permits it.

Useful committed evidence:

```text
fixtures/README.md
fixtures/<tool-build>/manifest.json
fixtures/<tool-build>/text/*.vmap.txt
analysis/diffs/*.diff
docs/FORMAT-NOTES-<tool-build>.md
```

For Valve-authored example content or copyrighted assets, store only paths/hashes/observations unless redistribution is explicitly allowed.