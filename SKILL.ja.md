---
name: minecraft-server-scriptapi
description: Minecraft Bedrock Script API の @minecraft/server モジュールに関するガイドライン。サーバーサイドのスクリプティング、ワールド/システム API、イベント、エンティティ、コンポーネント、または Minecraft Creator ScriptAPI のモジュールバージョンに関する質問に答える際に使用します。
---

# Minecraft Server ScriptAPI

## ワークフロー

1. **スコープ**: タスクを特定（イベント、エンティティ、コンポーネント、コマンド、タイミング）。ベータ/プレビュー版の要求がない限り、最新の安定版をデフォルトとする。
2. **ドキュメント**: モジュールインデックスから特定のクラス/インターフェース/イベントに移動。Microsoft Learn を真実の情報源（Source of Truth）として使用。詳細が不足している場合は MCP で検索。
3. **出力**: 正確な API 名を引用。必要なインポートを含む最小限の動作例を提供。カスタムヘルパーを作成する前に、公式の同等物が存在しないことを確認。
   - `CustomCommandParamType` のような列挙型については、利用可能かどうかを常に Microsoft Learn で確認してください。
   - `@minecraft/vanilla-data` は Microsoft Learn に掲載されていないため、これらの列挙型については MCP 検索をスキップしてください。

## 一般的なパターン

### イベント
- `world`/`system` イベントの subscribe/unsubscribe。パフォーマンスのためガードロジックを実装。

### ディメンション
- `world.getDimension(MinecraftDimensionTypes.<Dimension>)` を使用し、タスクに適したディメンションを選択してください。

### コンポーネント
- アクセス前に存在を確認。
- 型付き ID を使用：`EntityComponentTypes`、`BlockComponentTypes`、`ItemComponentTypes`。

### スケジューリング
- `system.run`、`system.runTimeout`、`system.runInterval`、`system.runJob` を使用。

### 識別子
- **必ず `@minecraft/vanilla-data` の列挙型を使用**：`MinecraftBlockTypes`、`MinecraftEntityTypes`、`MinecraftItemTypes`、`MinecraftDimensionTypes`、`MinecraftEffectTypes`、`potionEffect`、`potionDelivery`、`feature`、`enchantment`、`cooldownCategory`、`cameraPresets`、`biome`。
- **カスタム ID**：名前空間プレフィックスを含める必要がある（例：`example:cmd`）。アドオンごとに一貫したプレフィックスを1つ使用。

例：

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

## 権限モード

### 読み取り専用
- シミュレーション/イベント/ティック開始前。ワールドの変更不可。
- 修正：`system.run`/`runTimeout`/`runJob` に遅延。

### ドキュメント検証
- メソッド/プロパティを使用する際は、常に Microsoft Learn で読み取り専用セーフまたは早期実行セーフかを確認してください。
- API リファレンスの明示的な注意書き（read-only / early-execution）を確認し、厳密に従ってください。
- アロー関数コールバックのみの場合、以下のような制限実行（restricted-execution）の注意書きも確認してください：
  - "This closure is called with restricted-execution privilege."
  - "This function can't be called in restricted-execution mode."
- 制限実行の注意書きが適用される場合、アロー関数の本体に読み取り専用または早期実行の違反がないかを確認し、必要に応じて `system.run` で遅延してください。

例（読み取り専用の遅延）：

```ts
world.beforeEvents.playerInteractWithBlock.subscribe((event) => {
  const player = event.player;
  system.run(() => {
    player.runCommand("say ok");
  });
});
```

### 早期実行
- ワールドロード前。多くの API が利用不可。
- 修正：`world.afterEvents.worldLoad` または `system.run` に遅延。
- イベントの聞き逃しを避けるため、ルートで subscribe。

**早期実行で安全**：
- イベント購読（`world`/`system` の `beforeEvents`/`afterEvents`）
- `system.clearJob`、`clearRun`、`run`、`runInterval`、`runJob`、`runTimeout`、`waitTicks`
- `BlockComponentRegistry.registerCustomComponent`、`ItemComponentRegistry.registerCustomComponent`

## カスタムコマンド

- インターフェース：`CustomCommand`（`name`、`description`、`permissionLevel`、`mandatoryParameters`、`optionalParameters`）。
- パラメータ：`CustomCommandParameter`（`name`、`type`、オプションで `enumName`）。
- Enum 登録：`CustomCommandRegistry.registerEnum(name, values)`。
- コマンド登録：`CustomCommandRegistry.registerCommand(customCommand, callback)`。
- コールバック：`(origin, ...args) => CustomCommandResult`。

例：

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
        {  type: CustomCommandParamType.String, name: "msg", },
        {  type: CustomCommandParamType.Enum, name: "example:mode"},
      ],
      optionalParameters: [
        { type: CustomCommandParamType.Boolean, name: "silent" },
        { type: CustomCommandParamType.Integer, name: "count" },
      ],
    },
    (origin, msg, mode, silent, count) => {
      const msgValue = String(msg ?? "ok");
      const modeValue = String(mode ?? "off");
      const silentValue = Boolean(silent ?? false);
      const countValue = Number(count ?? 1);

      return {
        status: CustomCommandStatus.Success,
        message: silentValue
          ? undefined
          : `[${origin.sourceType}] ${msgValue} mode=${modeValue} count=${countValue}`,
      };
    }
  );
});
```

## スクリプトイベント

- 送信：`system.sendScriptEvent(id, message)`（名前空間付き ID、ペイロード文字列）。
- 受信：`system.afterEvents.scriptEventReceive.subscribe(callback, options?)`。
- イベント：`ScriptEventCommandMessageAfterEvent`（`id`、`message`、`sourceType`、オプションで `sourceEntity`/`sourceBlock`/`initiator`）。
- フィルター：`ScriptEventMessageFilterOptions.namespaces`。

例：

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

## パフォーマンス

- イベント内での負荷の高い処理：`system.runJob(generator)` を使用してティック間に分散。
- 大規模なセットを反復する前にショートサーキットチェック。

## system.run のバリエーション

- `run`：次のティック。
- `runTimeout(cb, ticks)`：N ティック遅延。`0` は誤用するとタイトループを引き起こす可能性。
- `runInterval(cb, ticks)`：`clearRun` まで N ティックごとに繰り返し。
- `runJob(generator)`：長時間実行される処理。反復処理を小さく保つ。

## 型安全性

- `typeId` チェックとコンポーネント存在確認を使用。
- `instanceof` はドキュメント化された `@minecraft/server` クラスのみに使用。



## 最小限のテンプレート

イベントの購読：

```ts
world.afterEvents.playerJoin.subscribe((event) => {
  const player = event.player;
});
```

ティックループ：

```ts
system.runInterval(() => {
  // ティックごとのロジック
}, 1);
```

ディメンションの取得：

```ts
const overworld = world.getDimension(MinecraftDimensionTypes.Overworld);
```

エンティティのスポーン：

```ts
const overworld = world.getDimension(MinecraftDimensionTypes.Overworld);
overworld.spawnEntity(MinecraftEntityTypes.Zombie, { x: 0, y: 80, z: 0 });
```

アイテムの付与：

```ts
const item = new ItemStack(MinecraftItemTypes.Diamond, 1);
player.getComponent("inventory")?.container?.addItem(item);
```

エフェクトの付与：

```ts
player.addEffect(MinecraftEffectTypes.Speed, 200, { amplifier: 1 });
```

コンポーネントの取得（型指定あり）：

```ts
const health = entity.getComponent(EntityComponentTypes.Health);
```

## よくある落とし穴

- **null コンポーネント**：アクセス前に存在を確認。
- **重いイベント**：`system.runJob` を使用。
- **権限**：読み取り専用/早期実行の違反を回避。
- **生の ID**：型付き列挙型を使用。
- **名前空間**：アドオンごとに一貫したプレフィックスを1つ使用。

## MCP ツール

- エンドポイント：https://learn.microsoft.com/api/mcp
- 検索：`microsoft_docs_search`（モジュール + クラス + タスク）。
- 取得：`microsoft_docs_fetch`（完全なシグネチャ/詳細）。
