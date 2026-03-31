---
name: hytale-modding
description: Expert Hytale Java server-plugin development for creating, modifying, and debugging Hytale mods/plugins built on the `com.hypixel.hytale:Server` API. Use when requests mention Hytale, Hytale plugins/mods, `JavaPlugin`, `manifest.json`, `plugin-template`, `AbstractPlayerCommand`, `AbstractAsyncCommand`, `EntityStore`, `PlayerReadyEvent`, `HytalePermissions`, `blockymodel`, commands, events, ECS systems/components, or custom blocks/items.
---

# Hytale Modding

Treat Hytale mods as Java server-side plugins first. Default to Java 25, Gradle Kotlin DSL, and the official plugin template. Assume players do not need client mods unless the task explicitly involves asset-pack content such as custom blocks, items, textures, or models.

## Use This Skill For

- Scaffolding a new Hytale plugin from the official template
- Creating or fixing commands, permissions, events, and ECS systems
- Debugging `Store<EntityStore>`, `Ref<EntityStore>`, `PlayerRef`, or world-thread issues
- Adding custom blocks, items, textures, `.blockymodel` files, or language entries
- Updating `build.gradle.kts`, `manifest.json`, or the plugin entry point

## Core Rules

- Never invent Java classes or method names. If a needed API detail is not in this skill or in [references/documentation-sources.md](references/documentation-sources.md), say so explicitly.
- Prefer `AbstractPlayerCommand` for gameplay commands. Reach for `AbstractAsyncCommand` only when world or entity access is not required.
- Never hold raw entity objects across time. Use `Ref<EntityStore>` and call `validate()` before reuse.
- In ECS systems, mutate through `CommandBuffer` instead of editing the store unsafely.
- Set `IncludesAssetPack` to `true` only when the plugin ships custom content.

## Quick Scaffold

Use the official template unless the repo already exists:

```bash
git clone https://github.com/HytaleModding/plugin-template.git my-plugin
cd my-plugin
```

Use this Gradle repository and dependency:

```kotlin
repositories {
    maven("https://maven.hytale.com/release")
}

dependencies {
    compileOnly("com.hypixel.hytale:Server:+")
}
```

Create a root `manifest.json`:

```json
{
  "Name": "MyPlugin",
  "Version": "1.0.0",
  "Authors": ["YourName"],
  "IncludesAssetPack": false
}
```

Basic entry point:

```java
public class MyPlugin extends JavaPlugin {
    @Override
    public void setup() {
        // Register commands, events, and systems here
    }
}
```

## Command Workflow

### Threading Rule

`AbstractAsyncCommand` runs off the world thread. Do not directly access `Store` or `Ref` data there unless you safely obtain the relevant `World` first. For most gameplay logic, use `AbstractPlayerCommand`, which runs on the world thread and is safe for normal entity access.

### Command Type Selection

- `AbstractAsyncCommand`: background work with no world or entity mutation
- `AbstractPlayerCommand`: player-triggered gameplay logic on the world thread
- `AbstractTargetPlayerCommand`: commands that target another player via `--player`
- `AbstractTargetEntityCommand`: commands that act on the entity the player is looking at

### Minimal Player Command

```java
public class ExampleCommand extends AbstractPlayerCommand {
    public ExampleCommand() {
        super("test", "Super test command!");
    }

    @Override
    protected void execute(@Nonnull CommandContext ctx,
            @Nonnull Store<EntityStore> store,
            @Nonnull Ref<EntityStore> ref,
            @Nonnull PlayerRef playerRef,
            @Nonnull World world) {
        Player player = store.getComponent(ref, Player.getComponentType());
        TransformComponent transform = store.getComponent(ref, TransformComponent.getComponentType());
        player.sendMessage(Message.raw("Position: " + transform.getPosition()));
    }
}
```

Register commands inside `setup()`:

```java
this.getCommandRegistry().registerCommand(new ExampleCommand());
```

### Arguments And Permissions

```java
RequiredArg<Float> healthArg = withRequiredArg("health", "HP amount", ArgTypes.FLOAT);
OptionalArg<String> msgArg = withOptionalArg("message", "Optional msg", ArgTypes.STRING);
DefaultArg<Float> defArg = withDefaultArg("health", "HP amount", ArgTypes.FLOAT, 100f, "default: 100");
FlagArg debugArg = withFlagArg("debug", "Enable debug logs");

healthArg.addValidator(Validators.greaterThan(0)).addValidator(Validators.lessThan(1000));

requirePermission(HytalePermissions.fromCommand("admin"));
```

Available argument types:

- `ArgTypes.FLOAT`
- `ArgTypes.INTEGER`
- `ArgTypes.STRING`
- `ArgTypes.BOOLEAN`
- `ArgTypes.PLAYER_REF`

Useful command patterns:

- Use `addAliases("tp", "teleport");` for aliases
- Use `addUsageVariant(new GiveOtherCommand());` for alternate arg sets
- Use `PermissionRules.or(...)` when moderator-or-admin access is valid

## Events And ECS

### Global Events

```java
public class ExampleEvent {
    public static void onPlayerReady(PlayerReadyEvent event) {
        Player player = event.getPlayer();
        player.sendMessage(Message.raw("Welcome " + player.getDisplayName()));
    }
}

this.getEventRegistry().registerGlobal(PlayerReadyEvent.class, ExampleEvent::onPlayerReady);
```

### ECS Events

Use entity systems when the task is entity-level, cancellable, or query-driven:

```java
class ExampleCancelCraft extends EntityEventSystem<EntityStore, CraftRecipeEvent.Pre> {
    public ExampleCancelCraft() {
        super(CraftRecipeEvent.Pre.class);
    }

    @Override
    public void handle(int index,
            ArchetypeChunk<EntityStore> chunk,
            Store<EntityStore> store,
            CommandBuffer<EntityStore> commandBuffer,
            CraftRecipeEvent.Pre event) {
        event.setCancelled(true);
    }

    @Override
    public Query<EntityStore> getQuery() {
        return Archetype.empty();
    }
}
```

Register ECS systems in `setup()`:

```java
this.getEntityStoreRegistry().registerSystem(new ExampleCancelCraft());
```

## ECS Essentials

- `Store<EntityStore>`: the authoritative source for entity components
- `Ref<EntityStore>`: safe entity handle; prefer it over raw references
- `EntityStore`: the world-aware store that maps entities by UUID and network ID
- `ChunkStore`: chunk and block component storage
- `Holder`: blueprint before an entity is inserted into the store
- `CommandBuffer`: queued entity mutations for thread safety

Access components like this:

```java
Player player = store.getComponent(ref, Player.getComponentType());
UUIDComponent uuid = store.getComponent(ref, UUIDComponent.getComponentType());
TransformComponent transform = store.getComponent(ref, TransformComponent.getComponentType());
EntityStatMap stats = store.getComponent(ref, EntityStatMap.getComponentType());
```

Use `PlayerRef` for player identity and connection state. Use `Player` for the player's spawned world presence.

Mutate components through `CommandBuffer` when appropriate:

```java
commandBuffer.addComponent(ref, componentType, new MyComponent());
commandBuffer.removeComponent(ref, componentType);
```

Health pattern:

```java
EntityStatMap stats = store.getComponent(ref, EntityStatMap.getComponentType());
int healthIdx = DefaultEntityStatTypes.getHealth();
stats.addStatValue(healthIdx, 50f);
```

### Custom Components

If you create a custom ECS component, implement `clone()` and provide a codec. `KeyedCodec` identifiers must start with uppercase letters and remain unique across the mod.

```java
public static final BuilderCodec<PoisonComponent> CODEC = BuilderCodec
    .builder(PoisonComponent.class, PoisonComponent::new)
    .append(new KeyedCodec<>("DamagePerTick", Codec.FLOAT),
        (d, v) -> d.damagePerTick = v, d -> d.damagePerTick).add()
    .append(new KeyedCodec<>("RemainingTicks", Codec.INTEGER),
        (d, v) -> d.remainingTicks = v, d -> d.remainingTicks).add()
    .build();
```

## Custom Blocks And Items

If the plugin adds content, turn on the asset pack:

```json
{ "IncludesAssetPack": true }
```

Expected structure:

```text
resources/
  Server/
    Item/Items/my_new_block.json
    Languages/en-US/items.lang
  Common/
    Icons/
    Blocks/my_new_block/model.blockymodel
    BlockTextures/texture.png
```

Language entries:

```text
my_new_block.name = My New Block
my_new_block.description = My Description
```

Item definition skeleton:

```json
{
  "TranslationProperties": {
    "Name": "items.my_new_block.name",
    "Description": "items.my_new_block.description"
  },
  "Id": "My_New_Block",
  "MaxStack": 100,
  "Icon": "Icons/ItemsGenerated/my_new_block.png"
}
```

When the request involves block models, use Blockbench to produce `.blockymodel` assets.

## Common Patterns

Send a message:

```java
player.sendMessage(Message.raw("Hello!"));
```

Teleport a player by updating the transform:

```java
TransformComponent transform = store.getComponent(ref, TransformComponent.getComponentType());
transform.setPosition(new Vector3f(x, y, z));
```

Check a permission:

```java
if (playerRef.hasPermission(HytalePermissions.fromCommand("admin"))) {
    // privileged logic
}
```

## Documentation Strategy

Use the inline guidance in this skill first. Only load additional documentation when the request goes beyond the covered topics here, such as UI, world generation, NPC spawning, persistent data, inventory management, build/test details, or publishing.

For those cases, open [references/documentation-sources.md](references/documentation-sources.md) and fetch only the relevant raw document instead of browsing broadly.
