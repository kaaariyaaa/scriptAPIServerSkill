---
name: minecraft-server-scriptapi
description: Guidance for working with the Minecraft Bedrock Script API @minecraft/server module. Use when answering questions about server-side scripting, world/system APIs, events, entities, components, or module versions for Minecraft Creator ScriptAPI.
---

# Minecraft Server ScriptAPI

Follow these steps to answer requests about the `@minecraft/server` module.

## Establish scope

- Identify the requested task (events, world/system, entities, components, commands, tick timing, etc.).
- Determine the target API version if the user specifies one; otherwise prefer the latest stable version on the docs page.
- If the user mentions beta/preview features, use the beta version and call out instability.

## Find the right docs

- Start from the module index page and navigate to the specific class, interface, or event.
- Prefer official Microsoft Learn pages as the source of truth.
- If a method signature or property type is unclear, read the specific class page rather than relying on memory.
- Use the Microsoft Docs MCP tools to search and fetch pages when details are missing.

## Provide precise guidance

- Quote the exact class, method, property, and event names as shown in the docs.
- Include the minimal example needed to solve the task (TypeScript or JavaScript as requested).
- Call out required imports, especially `world`, `system`, and types from `@minecraft/server`.
- Mention important constants (e.g., `TicksPerSecond`, `TicksPerDay`) only when relevant.

## Common patterns

### Event handling
- Subscribe/unsubscribe on `world` or `system` events and guard logic for performance.

### World interaction
- Use `world.getDimension(MinecraftDimensionTypes.Overworld)` or target dimensions explicitly when needed.

### Entity and component usage
- Check for component existence before access to avoid errors.
- Use typed component IDs from `@minecraft/server` instead of raw strings:
  - `EntityComponentTypes` for entity components (e.g., `entity.getComponent(EntityComponentTypes.Health)`)
  - `BlockComponentTypes` for block components
  - `ItemComponentTypes` for item components

### Scheduling
- Use `system.run`, `system.runTimeout`, or tick-based scheduling with `TicksPerSecond`.

### Identifiers and enums
- **MUST use `@minecraft/vanilla-data` enums** when available instead of raw string IDs:
  - `MinecraftBlockTypes` for blocks
  - `MinecraftEntityTypes` for entities
  - `MinecraftItemTypes` for items
  - `MinecraftDimensionTypes` for dimensions
  - `MinecraftEffectTypes` for effects
  - Other available enums: `potionEffect`, `potionDelivery`, `feature`, `enchantment`, `cooldownCategory`, `cameraPresets`, `biom`
- **Custom command and scriptEvent IDs** must include a namespace prefix (e.g., `example:testCommand`):
  - The prefix is per addon; use one consistent prefix throughout the addon
  - Cannot mix multiple prefixes within a single addon

Example:

```ts
import { world, system } from "@minecraft/server";
import { MinecraftDimensionTypes, MinecraftBlockTypes } from "@minecraft/vanilla-data";

system.runInterval(() => {
  const dimensions = [MinecraftDimensionTypes.Overworld, MinecraftDimensionTypes.Nether];
  const blocks = [MinecraftBlockTypes.Stone, MinecraftBlockTypes.Sand, MinecraftBlockTypes.GrassBlock];
  for (const dimension of dimensions) {
    for (const block of blocks) {
      world.getDimension(dimension).setBlockType({ x: 0, y: 0, z: 0 }, block);
    }
  }
});
```

## Validate assumptions

- If the request depends on game version or experimental features, ask a single clarifying question.
- If the user’s request conflicts with API limitations, explain the constraint and provide a supported alternative.

## Outputs

- Prefer concise, working snippets with imports and minimal scaffolding.
- Explain only the non-obvious parts and cite the relevant API names.

## Versioning and compatibility

- If the user does not specify a version, use the latest stable `@minecraft/server` and matching `@minecraft/vanilla-data` version.
- If the user specifies a beta/preview feature, target the beta docs and mention instability.
- If a symbol is missing in the user environment, suggest upgrading the package version or using the closest stable alternative.

## Permission modes (read-only, early, restricted)

- Permission errors usually mean a native method/property was called in read-only or early-execution mode.
- Check API docs for notes like:
  - "This function can't be called in read-only mode."
  - "This property can't be edited in read-only mode."
  - "This function can be called in early-execution mode."
  - "This property can be read in early-execution mode."

### Read-only mode

- Scripts are read-only before simulation starts, before events fire, or at the start of a script tick.
- World state cannot be changed in this mode.
- Fix: move restricted calls to `system.run` / `system.runTimeout` / `system.runJob` and capture event data you need before deferring.

### Early-execution mode

- Runs before the world is fully loaded, so many world APIs are unavailable.
- Fix: move world-dependent code into `world.afterEvents.worldLoad` or defer with `system.run`.
- Keep event subscriptions at root context to avoid missing events.

Early-execution safe APIs (from docs):

- `world.beforeEvents.*.subscribe/unsubscribe`
- `world.afterEvents.*.subscribe/unsubscribe`
- `system.beforeEvents.*.subscribe/unsubscribe`
- `system.afterEvents.*.subscribe/unsubscribe`
- `system.clearJob`, `system.clearRun`, `system.run`, `system.runInterval`, `system.runJob`, `system.runTimeout`, `system.waitTicks`
- `BlockComponentRegistry.registerCustomComponent`
- `ItemComponentRegistry.registerCustomComponent`

## Custom commands

- Define a command with the `CustomCommand` interface: `name`, `description`, `permissionLevel`, plus `mandatoryParameters` and `optionalParameters`.
- Parameters use `CustomCommandParameter` with `name` and `type` (and optional `enumName` when `CustomCommandParamType` is `Enum`).
- Register enums with `CustomCommandRegistry.registerEnum(name, values)` before using them in parameters.
- Register commands with `CustomCommandRegistry.registerCommand(customCommand, callback)`; callback receives `CustomCommandOrigin` and `any[]` args and returns `CustomCommandResult`.
- Reference docs: `CustomCommand`, `CustomCommandParameter`, `CustomCommandRegistry`, `CustomCommandOrigin`, `CustomCommandResult`, `CommandPermissionLevel`.

Example:

```ts
import {
  system,
  StartupEvent,
  CommandPermissionLevel,
  CustomCommandParamType,
  CustomCommandStatus,
} from "@minecraft/server";

system.beforeEvents.startup.subscribe((init: StartupEvent) => {
  init.customCommandRegistry.registerEnum("example:mode", ["on", "off"]);

  init.customCommandRegistry.registerCommand(
    {
      name: "example:demo",
      description: "Command demo",
      permissionLevel: CommandPermissionLevel.GameDirectors,
      cheatsRequired: true,
      mandatoryParameters: [
        { type: CustomCommandParamType.String, name: "msg" },
        { type: CustomCommandParamType.Enum, name: "mode", enumName: "example:mode" },
      ],
      optionalParameters: [
        { type: CustomCommandParamType.Boolean, name: "silent" },
        { type: CustomCommandParamType.Int, name: "count" },
      ],
    },
    (origin, args) => {
      const msg = String(args[0] ?? "ok");
      const mode = String(args[1] ?? "off");
      const silent = Boolean(args[2] ?? false);
      const count = Number(args[3] ?? 1);

      return {
        status: CustomCommandStatus.Success,
        message: silent
          ? undefined
          : `[${origin.sourceType}] ${msg} mode=${mode} count=${count}`,
      };
    }
  );
});
```

## Script events

- Use `system.sendScriptEvent(id, message)` to emit a script event; `id` is the namespaced identifier and `message` is the payload string.
- Receive events via `system.afterEvents.scriptEventReceive.subscribe(callback, options?)`.
- Callback argument is `ScriptEventCommandMessageAfterEvent` with `id`, `message`, `sourceType`, and optional `sourceEntity`/`sourceBlock`/`initiator`.
- `ScriptEventMessageFilterOptions.namespaces` can be used to filter inbound events by namespace.

Example:

```ts
import { system, world, ScriptEventSource } from "@minecraft/server";

system.afterEvents.scriptEventReceive.subscribe((event) => {
  const { id, message, sourceType, initiator, sourceEntity, sourceBlock } = event;
  if (id !== "example:say") return;

  switch (sourceType) {
    case ScriptEventSource.Block:
      world.sendMessage(`sendBy:${sourceBlock?.typeId ?? "unknown"} ${message}`);
      break;
    case ScriptEventSource.Entity:
      world.sendMessage(`sendBy:${sourceEntity?.typeId ?? "unknown"} ${message}`);
      break;
    case ScriptEventSource.NPCDialogue:
      world.sendMessage(`sendBy:${initiator?.typeId ?? "unknown"} ${message}`);
      break;
    case ScriptEventSource.Server:
      world.sendMessage(`sendBy:server ${message}`);
      break;
  }
});

```

## Performance safety

- For expensive work inside events, use `system.runJob(generator)` to spread work across ticks and avoid blocking the event pipeline.
- Use short-circuit checks before iterating large entity/block sets.

## system.run guide notes

- `system.run` queues a callback for the next tick; safe for deferring read-only work from before-events.
- `system.runTimeout` lets you choose tick delay; `0` can run in the same tick and can create infinite loops if misused.
- `system.runInterval` repeats every N ticks until `system.clearRun`.
- `system.runJob` is for long-running work; use generator functions with fine-grained yields.
- Keep generator iterations small and consistent so the job system can scale across devices.

Example (deferring heavy work):

```ts
world.beforeEvents.playerBreakBlock.subscribe((event) => {
  system.runJob(function* () {
    // heavy work here
  });
});
```

## Type guards and safety

- Use `typeId` checks for branch logic and ensure expected components exist before access.
- Use `instanceof` only when the type is a class from `@minecraft/server` and documented as such.



## Minimal templates

Event subscription:

```ts
world.afterEvents.playerJoin.subscribe((event) => {
  const player = event.player;
});
```

Tick loop:

```ts
system.runInterval(() => {
  // tick logic
}, 1);
```

Get dimension:

```ts
const overworld = world.getDimension(MinecraftDimensionTypes.Overworld);
```

Spawn entity:

```ts
const overworld = world.getDimension(MinecraftDimensionTypes.Overworld);
overworld.spawnEntity(MinecraftEntityTypes.Zombie, { x: 0, y: 80, z: 0 });
```

Give item:

```ts
const item = new ItemStack(MinecraftItemTypes.Diamond, 1);
player.getComponent("inventory")?.container?.addItem(item);
```

Apply effect:

```ts
player.addEffect(MinecraftEffectTypes.Speed, 200, { amplifier: 1 });
```

Get component (typed):

```ts
const health = entity.getComponent(EntityComponentTypes.Health);
```

## Common pitfalls

- **Null components**: Always check component existence before accessing properties or methods.
- **Heavy work inside events**: Use `system.runJob` to defer expensive operations and avoid blocking.
- **Missing permissions**: Ensure operations are not called in read-only or early-execution mode.
- **Raw string IDs**: Always prefer typed enums from `@minecraft/vanilla-data` and `@minecraft/server`.
- **Inconsistent namespace prefixes**: Use a single consistent prefix for all custom commands and script events in an addon.

## Microsoft Docs MCP usage

- MCP endpoint reference: https://learn.microsoft.com/api/mcp
- Search with `microsoft_docs_search` using specific queries: module name + class + task (e.g., "@minecraft/server Player class", "Minecraft Script API BlockComponent", "world events @minecraft/server").
- Fetch with `microsoft_docs_fetch` when the excerpt is too short or you need full signatures.
- For types and definitions, search with "class", "interface", "property", or "method" keywords in the query.
