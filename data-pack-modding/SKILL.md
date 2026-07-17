---
name: data-pack-modding
description: Create, modify, review, and debug Minecraft Java Edition data packs. Use when working with pack.mcmeta; namespaced data files; .mcfunction functions; function or registry tags; advancements; predicates; loot tables; recipes; item modifiers; structures; world generation; /datapack; or /reload. Apply this skill for version-compatible vanilla data pack projects, including overriding vanilla data and investigating pack-load errors.
---

# Data Pack Modding

Build Java Edition data packs against a declared Minecraft version. Treat every JSON schema, directory name, command syntax, and `pack.mcmeta` format as versioned.

## First Checks

- Confirm Java Edition, exact target release or snapshot, and whether the deliverable is a development folder or distributable `.zip`.
- Obtain the target data-pack format from that game with `/version` or `F3+V`; do not infer a current format number from an older example. For snapshot work, inspect the target build's bundled data pack as well.
- Choose a lowercase, unique namespace. Put new content in it; use `data/minecraft/...` only to deliberately override a vanilla ID or extend a vanilla tag.
- Identify the feature before choosing files: commands and scheduling use functions; reusable boolean tests use predicates; grouped IDs or lifecycle hooks use tags; event-driven player logic can use advancements; item generation uses loot tables; terrain and registry data require reload/restart planning.
- Read [references/minecraft-wiki.md](references/minecraft-wiki.md) before creating version-sensitive paths or schemas. It is a compact, sourced reference, not a substitute for checking the target version.

## Create Or Change A Pack

1. Start with a root `pack.mcmeta` and `data/<namespace>/`. Keep `pack.mcmeta` at the archive root, not inside an extra top-level directory when zipping.
2. Use the folders required by the target version. Current paths commonly include `function`, `advancement`, `predicate`, `loot_table`, `recipe`, `item_modifier`, `structure`, and `tags/<registry>`. Older packs used several plural names; do not mix path generations.
3. Assign a resource location from the path: `data/example/function/util/setup.mcfunction` is `example:util/setup`. Reference tags as `#example:...` where the consumer accepts tags.
4. Keep initialization idempotent. Register `example:setup` in `data/minecraft/tags/function/load.json`; load handlers also run on every `/reload` and before players join. Use `minecraft:tick` only for work that must happen every game tick, and make it bounded and cheap.
5. Write one command per `.mcfunction` line without `/`. Use `#` for full-line comments. Preserve the invoking execution context deliberately with `execute as`, `at`, `positioned`, and `in`.
6. Add only the smallest override necessary. Same-path non-tag files in a higher-priority pack replace lower-priority files. Tags merge unless the higher-priority tag has `"replace": true`.

## Pick The Data System

- **Function**: Create imperative behavior, commands, scheduling, or reusable subroutines. Check command syntax against the target release.
- **Function tag**: Run ordered groups of functions, including `minecraft:load` and `minecraft:tick`.
- **Registry tag**: Extend or replace a group of blocks, items, entities, or other registry entries. Use `{ "id": "...", "required": false }` only when an optional integration genuinely may be absent.
- **Predicate**: Put a named condition in `data/<namespace>/predicate/` and invoke it with `execute if predicate` or a selector's `predicate=` filter. Verify the available loot context before using context-dependent conditions.
- **Advancement**: React to player criteria and grant rewards or one function. Keep logic-only advancements displayless; give user-facing trees valid display objects, parents, and requirements.
- **Loot table**: Generate or modify item drops and container loot. Declare the appropriate `type` so predicates and item modifiers are validated against the correct loot context.

Read the matching section of [references/minecraft-wiki.md](references/minecraft-wiki.md) before authoring JSON for one of these systems.

## Validate In Layers

1. Parse every JSON file with `jq empty path/to/file.json`; validate `pack.mcmeta` too. Do not parse `.mcfunction` files as JSON.
2. Inspect all identifiers, namespaces, case, extensions, and paths. Verify every function, tag, predicate, loot table, advancement reward, and optional dependency reference resolves for the target installation.
3. Install the folder or correctly rooted zip in `<world>/datapacks/`. Run `/datapack list` to confirm it is enabled and ordered as expected.
4. Run `/reload` for reloadable data, then inspect the game or server log for parse, missing-reference, circular-tag, and invalid-loot-context errors. Reopen the world or restart the server after dynamic-registry or experimental/worldgen changes.
5. Exercise each entry point: reload setup, tick behavior, direct `/function`, advancement trigger/revoke loop, predicate branches, and relevant `/loot` invocation. Test on a disposable world before replacing vanilla behavior in a production world.

## Diagnose Failures

- Treat an absent pack as a root-layout or `pack.mcmeta` problem first.
- Treat a missing ID as a namespace, singular/plural directory, extension, load-order, or target-version problem before changing gameplay logic.
- Treat a tag failure as potentially fatal: a missing required member or a circular tag can prevent loading. Use `required: false` only for optional cross-pack content.
- Treat malformed function lines as load failures; one non-macro line that cannot parse can stop the function from loading. Remember that macro expansion can make the entire invocation fail when a value is missing or produces an invalid command.
- Avoid issuing player-facing commands from `minecraft:load`; there are no joined players at that point. Avoid uncontrolled per-tick selectors, scans, or recursion; functions share the command-chain budget.
- If a vanilla override is unexpectedly ignored, inspect `/datapack list` and confirm the intended pack is later in load order. If a tag seems additive when replacement was intended, add `"replace": true` only after accounting for every required vanilla member.

## Source Discipline

Use Minecraft Wiki as the primary conceptual and JSON-format reference captured in the bundled reference. For a changing target version, prefer the target game's `/version`, `F3+V`, logs, and bundled vanilla data over any static format table. Cite the target release and wiki URLs in user-facing technical documentation when version behavior matters.
