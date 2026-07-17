# Minecraft Wiki Data Pack Reference

Consult the section matching the file being changed. These notes summarize the Minecraft Wiki pages retrieved on 2026-07-17; Minecraft data formats change frequently, especially for snapshots. Verify the exact target game with `/version` or `F3+V` and test it in game.

## Pack Roots, IDs, And Reloading

- A Java data pack is a directory or zip with `pack.mcmeta` at its root. `pack.png` is optional. Put content under `data/<namespace>/...`.
- A regular data file at `data/<namespace>/<registry>/<path>.json` defines `<namespace>:<path>` in that registry. Functions use `.mcfunction`; structures use `.nbt`. A tag at `data/<namespace>/tags/<registry>/<path>.json` is `#<namespace>:<path>`.
- Use a custom namespace for new content. Files in `data/minecraft` target vanilla IDs and are overrides.
- For identical non-tag paths, the later-loaded data pack wins. Tags normally append entries from lower-priority packs; `"replace": true` discards lower-priority tag contents.
- `/reload` reloads tags, loot tables, recipes, advancements, item modifiers, predicates, functions, and structure templates. Dynamic registries and experimental/world-generation data require leaving and reopening a single-player world or restarting a server.

Sources: [Data pack](https://minecraft.wiki/w/Data_pack), [Tag (Java Edition)](https://minecraft.wiki/w/Tag_(Java_Edition)).

## pack.mcmeta And Format Changes

Use the metadata form supported by the target release. Do not put a guessed `pack_format` into a pack.

```json
{
  "pack": {
    "description": "Example data pack",
    "pack_format": 48
  }
}
```

The example is only representative of 1.21-1.21.1, not a default. Minecraft Wiki reports that newer versions changed metadata from `pack_format` to `min_format` and `max_format`; its format history also warns that its table is incomplete. Query the game (`/version` or `F3+V`) for the authoritative compatible format.

Important migration point: 1.21 introduced singular data directories such as `function`, `loot_table`, `recipe`, and `tags/item`, replacing older names such as `functions`, `loot_tables`, and `tags/items`. Use the layout from the target version's vanilla data pack.

Sources: [Data pack](https://minecraft.wiki/w/Data_pack#pack.mcmeta), [Pack format](https://minecraft.wiki/w/Pack_format).

## Functions And Lifecycle Tags

- Store a function at `data/<namespace>/function/<path>.mcfunction`; invoke it as `<namespace>:<path>`.
- Write one command per line without `/`; start a comment line with `#`. A trailing backslash continues a command line.
- Functions retain the invoking entity, position, rotation, dimension, and anchor. An `execute` context change affects its `run` branch, not later lines.
- `data/minecraft/tags/function/load.json` runs listed functions when a world/server loads and on `/reload`, before players join. `data/minecraft/tags/function/tick.json` runs listed functions at the start of every tick.
- A function tag runs functions in listed order and runs only the first duplicate occurrence. All nested function work runs in the same tick and is capped by `maxCommandChainLength`.
- Macros are lines beginning with `$`; they expand from the compound passed to `/function`. Missing substitutions or an unparseable expanded command prevents the entire function invocation.

Example lifecycle tag:

```json
{
  "values": ["example:setup"]
}
```

Source: [Function (Java Edition)](https://minecraft.wiki/w/Function_(Java_Edition)).

## Registry And Function Tags

```json
{
  "replace": false,
  "values": [
    "minecraft:oak_log",
    "#minecraft:logs",
    { "id": "other_pack:optional_log", "required": false }
  ]
}
```

- Put it at `data/<namespace>/tags/<registry>/<path>.json`; `<registry>` can be `function`, `block`, `item`, `entity_type`, and any valid registry for the target version.
- String entries are required by default. Prefix another tag with `#`. A missing required entry or circular reference can fail loading.
- Set `replace` only when the intended behavior is to remove lower-priority contents. Tags are recursively resolved in listed order.

Source: [Tag (Java Edition)](https://minecraft.wiki/w/Tag_(Java_Edition)).

## Predicates

- Store a predicate at `data/<namespace>/predicate/<path>.json`.
- Its root can be a predicate object or an array; an array is an implicit all-of.
- Invoke named predicates with `execute if|unless predicate <id>` or `@e[predicate=<id>]`. `minecraft:reference` can invoke one from another predicate.
- A condition may require a loot-context parameter such as an attacker, origin, block state, or entity. Such a predicate fails when that required context is unavailable.
- Predicate condition naming and shapes are version-sensitive. The wiki records an upcoming `condition` to `type` change, so check the target schema rather than mechanically porting snapshots.

Source: [Predicate](https://minecraft.wiki/w/Predicate).

## Loot Tables

- Store tables at `data/<namespace>/loot_table/<path>.json`.
- A root may have `type`, `functions`, `pools`, and `random_sequence`. Pools and root functions apply in order.
- `type` defaults to `generic`, but explicitly selecting the correct context makes invalid predicates and item modifiers visible in logs.
- A file at `data/minecraft/loot_table/entities/zombie.json` replaces vanilla zombie loot at that ID; it does not merge with vanilla.
- Exercise new tables with `/loot` and test real invocation contexts such as a block break, entity death, or container opening.

Source: [Loot table](https://minecraft.wiki/w/Loot_table).

## Advancements

- Store files at `data/<namespace>/advancement/<path>.json`.
- `criteria` is required. `requirements` is an array of alternative groups: every inner group needs one completed criterion; omitting it requires all criteria.
- `rewards` can grant experience, recipes, loot tables, and one function. Reward functions use a function ID, not a function tag.
- Keep `display` absent for invisible logic advancements. When present, it requires `icon`, `title`, and `description`; an advancing display tree needs valid parent IDs and a root background.
- Avoid circular `parent` references, which cause a loading failure. Test both grant and revoke behavior with `/advancement`.

Source: [Advancement definition](https://minecraft.wiki/w/Advancement/JSON_format).
