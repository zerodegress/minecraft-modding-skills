---
name: resource-pack-modding
description: Create, modify, review, and debug Minecraft Java Edition resource packs. Use when working with resource-pack pack.mcmeta; assets namespaces; PNG textures; OGG sounds; lang files; blockstates; block or item models; modern items model definitions; fonts; shaders; atlas files; sounds.json; pack overlays; or client-side resource reload failures.
---

# Resource Pack Modding

Build Java Edition resource packs against a declared Minecraft version. Resource paths, metadata, model systems, and image formats are client-versioned contracts.

## First Checks

- Confirm Java Edition, exact target release or snapshot, pack distribution method, and whether the goal is a vanilla override or assets for a mod/data-pack feature.
- Get the supported resource-pack format from the target game with `/version` or `F3+V`. Inspect that version's vanilla client assets before copying a path, identifier, model JSON, or item-rendering pattern.
- Use a lowercase custom namespace for new content. Use `assets/minecraft/...` only when intentionally replacing vanilla assets.
- Select the smallest applicable system: texture replacement, blockstate-to-model dispatch, raw model, modern item definition, language key, sound event, font, or shader. Read its section in [references/minecraft-wiki.md](references/minecraft-wiki.md) before writing it.
- Keep a development folder until visual and audio testing is complete; create a zip only for distribution. The zip root must contain `pack.mcmeta`, not an enclosing project directory.

## Build Or Modify A Pack

1. Create root `pack.mcmeta` and `assets/<namespace>/`. `pack.png` is optional and only supplies the pack-list icon.
2. Put each asset at its exact resource location. A model reference such as `example:block/marker` resolves under `assets/example/models/block/marker.json`; a texture reference omits `.png` and resolves under `assets/example/textures/...`.
3. Copy a vanilla asset's full relative path before overriding it. Resource packs are selected bottom-to-top: the lower pack loads first, then packs above it replace or merge it. Do not change an ID's path merely to make a cleaner folder layout.
4. Keep overrides narrow. Same-path textures, models, blockstates, and most JSON files replace their lower-priority counterpart. Language files merge per translation key; `sounds.json` has event-level merging and an explicit `replace` field.
5. Use forward slashes in resource locations and sound paths. Omit file extensions from model texture IDs and sound names; retain them in filesystem filenames.
6. Use the target release's item system. Modern releases use `assets/<namespace>/items/<id>.json` selected by an item's `item_model` component. Older releases can use legacy item-model `overrides`; do not combine schemas without verifying compatibility.

## Choose The Resource Type

- **Texture**: Add a correctly named PNG under `textures/`. Preserve the target asset's geometry and alpha expectations; a wrong or absent path produces the missing-texture checkerboard.
- **Animated texture**: Put frames in one atlas-compatible PNG and add a sibling `name.png.mcmeta` with an `animation` object. Make image dimensions divisible by each frame size.
- **Block appearance**: Map actual block-state values through `blockstates/<id>.json` to models. Use `variants` for mutually exclusive state combinations; use `multipart` for independent parts such as connections.
- **Raw model**: Put parent inheritance, texture variables, optional elements, faces, UVs, and display transforms in `models/`. A child `elements` array replaces inherited elements, so do not expect an additive merge.
- **Modern item appearance**: Put an items model definition in `items/`; compose or select raw models based on components and rendering properties. Reference raw geometry under `models/`.
- **Language**: Put translation maps in `lang/<locale>.json`. Preserve positional `%n$s` placeholders and escape literal percent signs as `%%`.
- **Sound**: Put OGG files under `sounds/` and define events in namespace-root `sounds.json`. Mark long audio as `stream`; use a mono file for positional sound and stereo for non-attenuated music-like audio.

## Validate In Layers

1. Parse `pack.mcmeta`, `sounds.json`, every model, blockstate, items, language, and `.mcmeta` JSON file with `jq empty path/to/file.json`.
2. Inspect filesystem case, namespaces, extensions, and resource locations. Confirm every referenced parent, model, texture variable, texture, sound, language key, and block-state value exists in the target installation or pack.
3. Verify image dimensions and alpha intentionally. Check every animation image divides cleanly into frames. Decode each OGG before shipping rather than assuming an extension proves it is playable.
4. Enable the pack in the target client and reload client resources with `F3+T`. Use a disposable test world for block and item states, and test UI scale plus both first- and third-person item rendering where relevant.
5. Check `latest.log` for JSON, parent-model, texture, atlas, shader, and sound errors. Test the pack again alongside likely higher/lower-priority packs to expose unintended replacement behavior.

## Diagnose Failures

- Treat a pack missing from the list as a `pack.mcmeta` or archive-root problem. Treat an incompatibility warning as a target-format problem, not an asset problem.
- Treat the magenta-and-black checkerboard as a missing or invalid texture path, filename case, resource location, or invalid image. Trace the reference from blockstate or items definition to raw model to texture variable to PNG.
- Treat a visually wrong block as an incomplete blockstate combination, incorrect state spelling, rotation, UV lock, or parent-model inheritance problem. Check the actual states in the target game before enumerating variants.
- Treat a missing item variation as a modern-versus-legacy item-model mismatch, a missing `item_model` component, or an incorrect `items` model ID.
- Treat silent audio as an event name, namespace, extension, OGG decode, or `sounds.json` path problem. Remember a sounds file path is relative to `sounds/` and excludes `.ogg`.
- Do not use `/reload` to test resource changes; it reloads server data. Use client resource reload or reselect the pack. A server-supplied pack must be a reachable, correctly rooted zip and is still rendered by clients.

## Source Discipline

Use the compact, sourced material in [references/minecraft-wiki.md](references/minecraft-wiki.md) for general rules. Prefer the exact target client's bundled vanilla assets, game logs, and in-game format query when documentation conflicts or snapshots change a schema.
