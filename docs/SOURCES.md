# Sources and provenance

Research date: 2026-09-12

This file records the main sources used for the initial architecture research. Source 2 authoring internals are only partly documented by Valve, so the evidence base mixes official Valve material, current community documentation, open-source reverse engineering and observed Workshop Tools command lines.

## Primary / high-value sources

### Valve: CS2 Maps Workshop FAQ

https://www.counter-strike.net/workshop/workshopfaq

Why it matters:

- Valve confirms that the CS2 Authoring Tools include Hammer;
- confirms command-line compiling utilities;
- confirms the Workshop Map Publisher workflow.

Confidence: **official Valve source**.

### Source2 Wiki: File Formats

https://www.source2.wiki/FileFormats

Why it matters:

- clearly distinguishes text KV3, binary KV3, DMX and older KeyValues;
- explicitly identifies Hammer `.vmap` source files as DMX;
- documents the Source 2 content/compiled-resource layers.

Confidence: **high-value current community technical documentation**, not Valve official.

### Source2 Wiki: KV2 / DMX

https://www.source2.wiki/FileFormats/dmx

Why it matters:

- documents DMX's typed object graph;
- documents current CS2 VMAP headers such as binary DMX encoding 9;
- shows Hammer's KeyValues2 text representation;
- documents `dmxconvert.exe` binary/text conversion;
- lists DMX libraries;
- notes Datamodel.NET support and use by Source 2 Viewer.

Confidence: **high-value current community technical documentation**.

### Source2 Wiki: VMAP example

https://www.source2.wiki/FileFormats/vmap/example.vmap

Why it matters:

- concrete text-DMX VMAP example;
- useful for studying element names and current/near-current serialized structure;
- also demonstrates that VMAP format versions evolve, reinforcing the need to avoid hard-coding one version.

Confidence: **community technical documentation / example**.

### Source2 Wiki: Resource Compiler

https://www.source2.wiki/EngineTools/ResourceCompiler

Why it matters:

- identifies `resourcecompiler.exe` location;
- documents `-i` and `-game` behavior;
- explains the content-root requirement and general compilation model.

Confidence: **current community documentation based on shipped tools**.

### Source2 Wiki: Compiling maps

https://www.source2.wiki/Basics/working-on-content/compiling-maps

Why it matters:

- explains map compilation as multiple stages;
- documents world, visibility, entities, physics, lighting, LOD, nav and Steam Audio outputs;
- explains incremental compile behavior;
- explains why entities-only builds are particularly useful for spawn/entity iteration;
- confirms that Resource Compiler can build maps outside Hammer.

Confidence: **current community documentation based on shipped tools**.

### Source2 Wiki: Directory layout

https://www.source2.wiki/Basics/working-on-content/directory-layout

Why it matters:

- confirms the `Counter-Strike Global Offensive` Steam folder vs `csgo` mod directory naming;
- documents asset path semantics;
- documents maps under the source content hierarchy.

Confidence: **current community documentation**.

### Source2 Wiki: FGD files

https://www.source2.wiki/Basics/fgd

Why it matters:

- explains FGD's role as Hammer/editor metadata;
- emphasizes that FGDs are not runtime engine truth;
- provides examples of common Source 2 FGD locations such as `game/core/base.fgd`, `lights.fgd` and model definitions.

Confidence: **current community documentation**.

### Source2 Wiki: Schemas

https://www.source2.wiki/Basics/schemas

Why it matters:

- documents the distinction between FGD editor metadata and runtime Source 2 schema/class information.

Confidence: **current community documentation / reverse-engineered engine knowledge**.

## Open-source reverse engineering

### ValveResourceFormat / Source 2 Viewer

https://github.com/ValveResourceFormat/ValveResourceFormat

https://s2v.app

Why it matters:

- one of the most complete current open-source Source 2 resource parsers/decompilers;
- explicitly supports CS2;
- includes map extraction to editable VMAP;
- includes map entity reconstruction;
- includes Hammer mesh reconstruction;
- documents that the implementation is based on reverse engineering rather than official Source 2 SDK documentation.

Important files examined conceptually/directly through public source:

- `ValveResourceFormat/IO/MapExtract.cs`
- `ValveResourceFormat/IO/HammerMeshBuilder.cs`
- Source 2 resource-format documentation under `docs/`

Confidence: **very high technical value, reverse engineered**.

Attribution requested by the project and retained here:

> Powered by [Source 2 Viewer](https://s2v.app) ([ValveResourceFormat](https://github.com/ValveResourceFormat/ValveResourceFormat)).

### Datamodel.NET

https://github.com/ValveResourceFormat/Datamodel.NET

Why it matters:

- DMX reader/writer;
- used by ValveResourceFormat to emit VMAP documents;
- strong candidate for a future structured map backend.

Confidence: **high-value implementation source**.

### dmxparser (Rust)

https://github.com/leops/dmxparser

Why it matters:

- independently demonstrates reading binary DMX v9;
- includes VMAP-oriented structures;
- useful as cross-reference for map types.

Known limitation from its own documentation:

- primarily reader-focused;
- incomplete write support / incomplete format knowledge.

Confidence: **useful secondary implementation reference**.

## MCP / architecture precedent

### Dota2_Workshop_MCP

https://github.com/ex3lite/Dota2_Workshop_MCP

Why it matters:

- working MCP server against another Source 2 Workshop Tools environment;
- explicitly treats `.vmap` as DMX;
- uses Valve `dmxconvert.exe` for binary <-> KeyValues2 conversion;
- edits maps and invokes `resourcecompiler`;
- documents end-to-end tested map creation/compilation;
- demonstrates a practical agent-oriented tool surface;
- explicitly limits arbitrary bespoke mesh generation, which supports the phased geometry strategy proposed here.

Confidence: **strong architectural precedent**, Dota-specific details must not be assumed valid for CS2 without verification.

## Source 1 comparison only

### hammer-mcp

https://github.com/ProjectSocietyStudio/hammer-mcp

Why it matters:

- demonstrates a useful MCP tool design for Source-engine mapping;
- includes map inspection/editing/compiler orchestration concepts.

Important limitation:

- targets Source 1 workflows (`.vmf`, `.bsp`, FGD and Source 1 compilers);
- not a direct implementation base for CS2/Source 2 VMAP.

Confidence: **architectural inspiration only for CS2**.

## Observed current CS2 compiler command

A 2026 Intel support thread contains an actual CS2 Workshop Tools `resourcecompiler.exe` invocation reported while diagnosing a Hammer build issue:

https://community.intel.com/t5/Intel-Arc-Discrete-Graphics/CS2-and-Dota-2-hammer-failed-to-building-with-Vulkan-after/m-p/1758647

The command includes current-looking map build switches such as:

```text
-world
-bakelighting
-lightmapMaxResolution
-lightmapVRadQuality
-phys
-html
-retail
-nop4
```

This source is useful evidence of how a current Hammer build invocation looks, but it should **not** be treated as the authoritative permanent command line for this MCP. The project should capture or construct arguments based on the installed current toolchain.

Confidence: **empirical current example**, not normative documentation.

## Evidence classification used by this project

When documenting implementation behavior, prefer labels like:

- **Valve documented** — official Valve site/content;
- **tool-observed** — behavior or output from installed Valve Workshop Tools;
- **reverse-engineered** — Source 2 Viewer / Source2 Wiki / equivalent research;
- **community example** — forum/project example known to work in a particular version;
- **hypothesis / needs verification** — design inference not yet tested locally.

This matters because CS2 and Source 2 authoring formats/tools can change with game updates.

## Things intentionally not asserted as proven yet

The research does **not** yet claim that the following are verified end-to-end for this project:

- the exact current CS2 VConsole transport/port/launch flags;
- the exact minimal set of attributes required to create every `CMapEntity` from scratch;
- the exact current VMAP format version on the user's installed CS2 build;
- a generic Hammer mesh generator that Hammer accepts and compiles in all cases;
- a stable one-size-fits-all `resourcecompiler.exe` command for all CS2 builds.

Those should be established with the local experiment plan before code claims compatibility.