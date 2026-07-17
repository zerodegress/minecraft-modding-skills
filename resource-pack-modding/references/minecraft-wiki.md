# Minecraft Wiki Resource Pack Reference

Consult the matching section before writing a resource. These notes summarize Minecraft Wiki pages retrieved on 2026-07-17. Resource-pack schemas change across releases and snapshots, so use the target game's `/version` or `F3+V`, bundled vanilla assets, and `latest.log` as the final authority.

## Pack Root, Namespaces, And Priority

- A Java resource pack is a directory or `.zip` with root `pack.mcmeta`; `pack.png` is optional. Install it in `.minecraft/resourcepacks`, or select it by drag-and-drop in the resource-pack UI.
- Store resources under `assets/<namespace>/`. Use `minecraft` to replace vanilla resources; use a custom namespace for new mod or data-pack-associated resources.
- Selected packs load from the bottom upward: each selected pack above can replace or merge assets from packs below it. A world may provide `resources.zip`; a server can distribute a zip through an HTTP(S) `resource-pack` URL.
- Common namespace folders are `atlases`, `blockstates`, `equipment`, `font`, `items`, `lang`, `models`, `particles`, `post_effect`, `shaders`, `sounds`, `texts`, and `textures`. `sounds.json` lives directly inside the namespace directory.
- `pack.mcmeta` uses the resource-pack format supported by the game. Do not hard-code a current number from an example. The wiki reports a newer migration from `pack_format` to `min_format` and `max_format`; verify the metadata form and number in the target game.

Sources: [Resource pack](https://minecraft.wiki/w/Resource_pack), [Pack format](https://minecraft.wiki/w/Pack_format).

## Textures And Animation

- Textures are RGBA PNG files under `assets/<namespace>/textures/`. A texture resource location omits `.png`, for example `example:block/marker` maps to `textures/block/marker.png`.
- Most vanilla block and item textures are 16x16, but the required geometry is determined by the referenced model. Preserve the source asset's aspect ratio and alpha behavior when overriding it.
- An atlas-compatible animated texture holds frames vertically, horizontally, or in a grid. Every frame must have equal dimensions and the image must divide evenly by the declared frame width and height.
- Add animation metadata beside the PNG: `marker.png.mcmeta`. Its root contains `animation`; useful fields are `interpolate`, `width`, `height`, `frametime` in ticks, and optional `frames` entries or `{ "index": n, "time": ticks }` entries. Without this metadata, Minecraft renders the image as static.

Example:

```json
{
  "animation": {
    "frametime": 2,
    "frames": [0, 1, { "index": 2, "time": 4 }]
  }
}
```

Sources: [Resource pack](https://minecraft.wiki/w/Resource_pack#Textures), [Textures](https://minecraft.wiki/w/Texture).

## Raw Models And Blockstate Dispatch

- Store raw models at `assets/<namespace>/models/<path>.json`. A model can use `parent`, `ambientocclusion`, `display`, `textures`, and `elements`.
- Parent and texture IDs are resource locations. Texture variables are used with `#`, such as a face's `"texture": "#all"`; the model's `textures` map resolves `all` to an actual texture ID.
- Defining `elements` in a child replaces the parent's `elements`. Elements are cuboids with `from`, `to`, and faces; each face can set `uv`, `texture`, `cullface`, `rotation`, and `tintindex`.
- Store dispatch files at `assets/<namespace>/blockstates/<block-id>.json`. Use `variants` for a mapping of comma-separated state assignments, using `""` for a single-state block. A mapping may choose one model or a weighted model array.
- Use `multipart` when several model parts should independently apply. Its `when` accepts direct state constraints or `OR`/`AND`; `apply` supplies one model or a weighted array. Variant/apply model entries can rotate by 90-degree increments and set `uvlock`.

Sources: [Model](https://minecraft.wiki/w/Model), [Blockstates definition](https://minecraft.wiki/w/Blockstates_definition).

## Modern Items Models

- Modern item definitions are stored at `assets/<namespace>/items/<id>.json`, not in `models/item`. An item's `item_model` component chooses this definition by ID.
- The root has optional rendering controls and a required `model` object. The core model types include `minecraft:model`, `minecraft:composite`, `minecraft:condition`, `minecraft:select`, `minecraft:range_dispatch`, `minecraft:empty`, `minecraft:bundle/selected_item`, and `minecraft:special`.
- `minecraft:model` points to a raw model under `models/`; optional `tints` are applied by tint index. `condition`, `select`, and `range_dispatch` choose nested models from boolean, discrete, or numeric properties, including custom model data.
- In releases before the modern item-definition system, use the target version's legacy `models/item` format and its `overrides` instead. The wiki records that legacy overrides were removed in 1.21.4-era development; never assume the two systems are interchangeable.

Source: [Items model definition](https://minecraft.wiki/w/Items_model_definition).

## Language Files

- Store a locale map at `assets/<namespace>/lang/<locale>.json`, such as `en_us.json`. Keys are translation IDs and values are displayed text.
- Language files merge by key across selected packs, so omit unrelated keys when changing a translation.
- Use `%s` for sequential substitutions or `%<number>$s` for explicit positions. Escape a literal percent sign as `%%`; use indexed placeholders when translation grammar changes argument order.

Example:

```json
{
  "item.example.marker": "Marker",
  "message.example.count": "%1$s has %2$s markers"
}
```

Source: [Resource pack](https://minecraft.wiki/w/Resource_pack#Language).

## Sounds

- Store OGG files at `assets/<namespace>/sounds/<path>.ogg`; define events in `assets/<namespace>/sounds.json`.
- An event contains optional `replace`, `subtitle`, and `sounds`. `replace: true` replaces lower-priority sounds for that event; false or absent adds entries.
- A sound string is a path relative to `sounds/` with no `.ogg`, such as `example:ui/alert`. A sound object may set `name`, `volume`, `pitch`, `weight`, `stream`, `attenuation_distance`, `preload`, and `type` (`file` or `event`).
- Use `stream: true` for long sounds to avoid loading them all at once. Mono sounds attenuate spatially; stereo sounds remain non-positional.

Example:

```json
{
  "ui.marker_alert": {
    "subtitle": "subtitles.example.marker_alert",
    "sounds": [
      { "name": "example:ui/marker_alert", "volume": 0.8 }
    ]
  }
}
```

Source: [sounds.json](https://minecraft.wiki/w/Sounds.json).
