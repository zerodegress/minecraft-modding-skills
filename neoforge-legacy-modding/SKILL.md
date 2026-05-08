---
name: neoforge-legacy-modding
description: NeoForge Legacy Forge / MinecraftForge 1.17-1.20.1 mod development guidance using NeoForged ModDevGradle's legacyforge plugin. Use when working on 1.20.1-legacy MDK projects, net.neoforged.moddev.legacyforge, Forge 47.x, net.minecraftforge.* APIs, mods.toml, DeferredRegister with RegistryObject, FMLJavaModLoadingContext, MinecraftForge.EVENT_BUS, SimpleChannel networking, Forge datagen, access transformers, mixins, reobfuscation, or porting between legacy Forge 1.20.1 and modern NeoForge.
---

# NeoForge Legacy Modding

Use this skill for MinecraftForge-compatible legacy projects maintained with NeoForged tooling, especially Minecraft 1.20.1 / Forge 47.x projects generated from the `neoforged/MDK` `1.20.1-legacy` branch. Treat this as Forge API modding with NeoForged build tooling: code imports are usually `net.minecraftforge.*`, not modern `net.neoforged.*`.

## First Checks

- Confirm the target Minecraft, Forge, Java, and Gradle plugin versions from `gradle.properties`, `build.gradle`, and `settings.gradle`.
- Expect `id 'net.neoforged.moddev.legacyforge'` in `build.gradle`; do not replace it with modern `net.neoforged.moddev` or NeoGradle unless explicitly porting.
- For Minecraft 1.20.1, target Java 17. Do not apply modern NeoForge 1.21+ Java 21 assumptions.
- Confirm the mod id in `gradle.properties`, the `@Mod` value, resource namespaces, and `src/main/templates/META-INF/mods.toml` all match.
- Use generated Gradle tasks first: `./gradlew build`, `./gradlew runClient`, `./gradlew runServer`, `./gradlew runData`, and `./gradlew gameTestServer` when configured.
- When checking APIs, prefer Forge 1.20.1 docs and project source over current NeoForge docs. Modern NeoForge docs are useful for concepts, but many package names and APIs differ.

## Build Tooling

- Configure Forge through the `legacyForge { version = "${minecraft_version}-${forge_version}" }` block.
- Keep common metadata in `gradle.properties`: `minecraft_version`, `forge_version`, `minecraft_version_range`, `forge_version_range`, `loader_version_range`, `mod_id`, `mod_version`, and display fields.
- Use `legacyForge.runs` for `client`, `server`, `gameTestServer`, and `data`; set `forge.enabledGameTestNamespaces` to the mod id when using game tests.
- Include generated datagen output with `sourceSets.main.resources { srcDir 'src/generated/resources' }`.
- Legacy Forge production jars need reobfuscation to SRG. Upload the reobfuscated `reobfJar` output, not the development jar under `build/devlibs`.
- For mod dependencies, use `modImplementation`, `modCompileOnly`, `modRuntimeOnly`, `modApi`, or a custom remapping configuration created with `obfuscation.createRemappingConfiguration`. These remap published SRG jars to official mappings for development.
- Use `localRuntime` plus `modLocalRuntime` for optional runtime-only test dependencies that should not become published dependencies.
- Configure mixins with the Mixin annotation processor, a refmap, a mixin config file, and a `MixinConfigs` manifest attribute.

## Project Shape

- Put Java code in `src/main/java`, runtime resources in `src/main/resources`, metadata templates in `src/main/templates`, and generated resources in `src/generated/resources`.
- Use `src/main/resources/assets/<modid>` for client assets and `src/main/resources/data/<modid>` for server data.
- Keep `META-INF/mods.toml` generated from templates when the MDK already has `generateModMetadata`; edit `gradle.properties` for basic metadata instead of hardcoding expanded files.
- Use `mods.toml` dependency ids `forge` and `minecraft` for Forge 1.20.1 projects. Do not rename them to `neoforge`.
- Keep common code free of `net.minecraft.client` imports. Isolate client-only setup in client classes or `@Mod.EventBusSubscriber(..., value = Dist.CLIENT)`.

## Registration

- Prefer `net.minecraftforge.registries.DeferredRegister` with `RegistryObject<T>` for normal registry entries.
- Create registers with `DeferredRegister.create(ForgeRegistries.ITEMS, MODID)`, `DeferredRegister.create(ForgeRegistries.BLOCKS, MODID)`, or `DeferredRegister.create(Registries.CREATIVE_MODE_TAB, MODID)`.
- Register each `DeferredRegister` on `FMLJavaModLoadingContext.get().getModEventBus()` in the mod constructor.
- Store registry references as `RegistryObject<T>` and call `.get()` only after registration is available or inside suppliers/events that run after registration.
- Do not use modern NeoForge helpers such as `DeferredRegister.Items`, `DeferredItem`, `DeferredBlock`, or `Item.Properties#setId` unless the project already targets a newer NeoForge version.
- Use `RegisterEvent` only when the project needs low-level control or an API does not fit deferred registration.

Typical legacy registration shape:

```java
public static final DeferredRegister<Item> ITEMS =
    DeferredRegister.create(ForgeRegistries.ITEMS, MODID);

public static final RegistryObject<Item> EXAMPLE_ITEM =
    ITEMS.register("example_item", () -> new Item(new Item.Properties()));

public ExampleMod() {
    IEventBus modBus = FMLJavaModLoadingContext.get().getModEventBus();
    ITEMS.register(modBus);
}
```

## Events And Lifecycle

- Use `MinecraftForge.EVENT_BUS` for most gameplay events.
- Use `FMLJavaModLoadingContext.get().getModEventBus()` for lifecycle, registry, datagen, client setup, creative tab, color/model, and other initialization events.
- Register handlers with `IEventBus#addListener`, `IEventBus#addGenericListener`, `@SubscribeEvent`, or `@Mod.EventBusSubscriber` according to local style.
- Event handler methods take one event parameter and return `void`.
- Do not subscribe to broad parent events unless intentionally handling all subevents.
- Check `event.isCancelable()` before canceling an unfamiliar event, and only use `setResult` on events that document results.
- Lifecycle events such as `FMLCommonSetupEvent` and `FMLClientSetupEvent` are parallel dispatch events; use `event.enqueueWork(...)` when touching main-thread state or other mods.
- Use `BuildCreativeModeTabContentsEvent` to add entries to vanilla creative tabs.

## Sides

- Use `Level#isClientSide` for logical-side decisions. Run authoritative gameplay only when it is `false`.
- Use `Dist`, `FMLEnvironment.dist`, `DistExecutor`, or `@Mod.EventBusSubscriber(value = Dist.CLIENT)` for physical-side classloading boundaries.
- Never reference `Minecraft`, screens, renderers, key mappings, or other `net.minecraft.client` classes from common classes loaded on a dedicated server.
- Remember singleplayer contains both a logical client and logical server inside the physical client. `Dist.CLIENT` does not mean "not server logic."
- Test `runServer` for every change that touches common code, registration, events, networking, config, resources, or class initialization.

## Networking

- Use Forge `SimpleChannel` from `net.minecraftforge.network.simple`.
- Create channels with `NetworkRegistry.newSimpleChannel(new ResourceLocation(MODID, "main"), protocolSupplier, clientPredicate, serverPredicate)`.
- Register packets with unique per-channel discriminators. Keep IDs deterministic, usually with `int id = 0; id++`.
- Packet registration supplies encoder, decoder, and handler functions using `FriendlyByteBuf`.
- Packet handlers run on the network thread by default; call `ctx.get().enqueueWork(...)` before touching world, player, block entity, or client state, then call `ctx.get().setPacketHandled(true)`.
- For serverbound packets, validate all client-provided data. Check `Level#hasChunkAt` before reading block entities or blocks at client-sent positions.
- Handle clientbound packets in client-only classes and cross the physical-side boundary with `DistExecutor` or an equivalent client-only event subscriber.
- Send to the server with `CHANNEL.sendToServer(message)` and to clients with `CHANNEL.send(PacketDistributor.PLAYER.with(...), message)`, `TRACKING_CHUNK`, `ALL`, or other Forge packet distributors.

## Data, Assets, And Metadata

- Use Forge 1.20.1 datagen providers for models, blockstates, lang, tags, recipes, loot tables, global loot modifiers, advancements, and datapack registry objects.
- Run datagen through the configured `data` run, typically emitting to `src/generated/resources`.
- Keep JSON paths lowercase and namespace them under the mod id.
- Put blockstates and models under `assets/<modid>`, recipes/tags/loot under `data/<modid>`, and Forge tags under `data/forge/tags/...` when contributing to shared Forge tags.
- Keep `mods.toml` display metadata, dependency ranges, `displayTest`, and `clientSideOnly` aligned with the actual networking and side behavior.

## Porting Notes

- Legacy Forge 1.20.1 uses `net.minecraftforge.*`; modern NeoForge uses `net.neoforged.*`. Do not mix imports in a single target unless compatibility shims are already present.
- `RegistryObject<T>` maps conceptually to modern holders/suppliers, but APIs and helper types differ.
- Forge `SimpleChannel` is not the same as modern NeoForge custom payload registration with `RegisterPayloadHandlersEvent`.
- Forge 1.20.1 items and blocks usually do not require explicit `ResourceKey` assignment in properties; modern examples that call `setId` are not legacy-compatible.
- `mods.toml` and `neoforge.mods.toml` are different metadata formats. Use the one the target loader expects.
- When backporting modern NeoForge examples, translate package names, event names, registration helpers, networking APIs, Java version, and metadata before coding.

## Validation Checklist

- Run `./gradlew build` and inspect whether the produced publishable jar is reobfuscated.
- Run `./gradlew runClient` for client setup, assets, models, screens, renderers, key mappings, and networking.
- Run `./gradlew runServer` for classloading and server safety, especially after touching common code.
- Run `./gradlew runData` after datagen changes and review generated JSON for namespace/path mistakes.
- Check logs for registry errors, missing translations, missing models, packet discriminator conflicts, invalid mod metadata, mixin refmap problems, and client-only class loading on the server.

## Official References

- NeoForged MDK legacy branch: `https://github.com/neoforged/MDK/tree/1.20.1-legacy`
- ModDevGradle legacy plugin: `https://github.com/neoforged/ModDevGradle/blob/main/LEGACY.md`
- NeoForged docs: `https://docs.neoforged.net/`
- Forge 1.20.1 docs: `https://docs.minecraftforge.net/en/1.20.1/`
- Forge 1.20.1 registries: `https://docs.minecraftforge.net/en/1.20.1/concepts/registries/`
- Forge 1.20.1 events: `https://docs.minecraftforge.net/en/1.20.1/concepts/events/`
- Forge 1.20.1 sides: `https://docs.minecraftforge.net/en/1.20.1/concepts/sides/`
- Forge 1.20.1 SimpleImpl networking: `https://docs.minecraftforge.net/en/1.20.1/networking/simpleimpl/`
