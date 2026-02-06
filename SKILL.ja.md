---
name: minecraft-server-scriptapi
description: Minecraft Bedrock Script API の @minecraft/server モジュールに関するガイドライン。サーバーサイドのスクリプティング、ワールド/システム API、イベント、エンティティ、コンポーネント、または Minecraft Creator ScriptAPI のモジュールバージョンに関する質問に答える際に使用します。
---

# Minecraft Server ScriptAPI

`@minecraft/server` モジュールに関するリクエストに答える際は、以下の手順に従ってください。

## スコープの確定

- 要求されたタスク（イベント、ワールド/システム、エンティティ、コンポーネント、コマンド、ティックタイミングなど）を特定します。
- ユーザーが API バージョンを指定している場合はそれを優先し、指定がない場合はドキュメントページの最新の安定版（Stable）を優先します。
- ユーザーがベータ版やプレビュー版の機能に言及している場合は、ベータ版を使用し、動作が不安定である可能性を指摘してください。

## 適切なドキュメントの検索

- モジュールのインデックスページから開始し、特定のクラス、インターフェース、またはイベントに移動します。
- 公式の Microsoft Learn ページを信頼できる情報源（Source of Truth）として優先してください。
- メソッドのシグネチャやプロパティの型が不明確な場合は、記憶に頼らずに特定のクラスのページを確認してください。
- 詳細が不足している場合は、Microsoft Docs MCP ツールを使用してページを検索・取得してください。

## 正確なガイダンスの提供

- ドキュメントに記載されているクラス、メソッド、プロパティ、イベント名を正確に引用してください。
- タスクを解決するために必要な最小限のサンプルコード（要求に応じて TypeScript または JavaScript）を含めてください。
- 必要なインポート、特に `@minecraft/server` からの `world`、`system`、および型について明記してください。
- 関連する場合のみ、重要な定数（例：`TicksPerSecond`、`TicksPerDay`）について言及してください。
- `@minecraft/server` 関連の新しい型/クラス/ヘルパーを定義する前に、Microsoft Learn MCP を介して公式の同等機能があるか確認してください。公式の API 型を優先し、ヘルパーは最小限に留めてください。

## 一般的なパターン

### イベント処理
- `world` または `system` イベントの subscribe/unsubscribe を行い、パフォーマンスのためにガードロジックを実装します。

### ワールド操作
- 必要に応じて `world.getDimension(MinecraftDimensionTypes.Overworld)` を使用するか、ターゲットのディメンションを明示的に指定します。

### エンティティとコンポーネントの使用
- エラーを避けるため、アクセス前にコンポーネントの存在を確認してください。
- 生の文字列ではなく、`@minecraft/server` の型付きコンポーネント ID を使用してください：
  - エンティティコンポーネントには `EntityComponentTypes`（例：`entity.getComponent(EntityComponentTypes.Health)`）
  - ブロックコンポーネントには `BlockComponentTypes`
  - アイテムコンポーネントには `ItemComponentTypes`

### スケジューリング
- `system.run`、`system.runTimeout`、または `TicksPerSecond` を使用したティックベースのスケジューリングを使用します。

### 識別子と列挙型
- 利用可能な場合は、生の文字列 ID ではなく **必ず `@minecraft/vanilla-data` の列挙型を使用してください**：
  - ブロックには `MinecraftBlockTypes`
  - エンティティには `MinecraftEntityTypes`
  - アイテムには `MinecraftItemTypes`
  - ディメンションには `MinecraftDimensionTypes`
  - エフェクトには `MinecraftEffectTypes`
  - その他の利用可能な列挙型：`potionEffect`, `potionDelivery`, `feature`, `enchantment`, `cooldownCategory`, `cameraPresets`, `biome`
- **カスタムコマンドと scriptEvent の ID** には名前空間プレフィックスを含める必要があります（例：`example:testCommand`）：
  - プレフィックスはアドオンごとに設定します。アドオン全体で一貫した1つのプレフィックスを使用してください
  - 1つのアドオン内で複数のプレフィックスを混在させることはできません

### コード構造（冗長性の回避）
- ドキュメントで同等のものが存在しないと確認できない限り、公式の `@minecraft/server` 型（例：`Vector3`）を再定義しないでください。

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

## 前提条件の検証

- リクエストがゲームのバージョンや実験的な機能に依存する場合は、明確にするための質問を 1 つ行ってください。
- ユーザーのリクエストが API の制限と競合する場合は、その制約を説明し、サポートされている代替案を提示してください。

## 出力

- インポート文と最小限の構成を含む、簡潔で動作するコードスニペットを優先してください。
- 明らかでない部分のみを説明し、関連する API 名を引用してください。

## バージョニングと互換性

- ユーザーがバージョンを指定しない場合は、最新の安定版の `@minecraft/server` と、それに対応するバージョンの `@minecraft/vanilla-data` を使用してください。
- ユーザーがベータ/プレビュー機能を指定した場合は、ベータ版のドキュメントを対象とし、動作が不安定であることを伝えてください。
- ユーザーの環境でシンボルが見つからない場合は、パッケージバージョンのアップグレード、または最も近い安定した代替手段の使用を提案してください。

## 権限モード (読み取り専用、早期実行、制限付き実行)

- 権限エラー（Permission error）は通常、読み取り専用モードまたは早期実行モードでネイティブのメソッドやプロパティが呼び出されたことを意味します。
- API ドキュメントで以下のような注意事項を確認してください。
  - 「この関数は読み取り専用モードでは呼び出せません（This function can't be called in read-only mode.）」
  - 「このプロパティは読み取り専用モードでは編集できません（This property can't be edited in read-only mode.）」
  - 「この関数は早期実行モードで呼び出すことができます（This function can be called in early-execution mode.）」
  - 「このプロパティは早期実行モードで読み取ることができます（This property can be read in early-execution mode.）」

### 読み取り専用モード (Read-only mode)

- シミュレーションが開始される前、イベントが発火する前、またはスクリプトティックの開始時などは読み取り専用モードになります。
- このモードではワールドの状態を変更することはできません。
- 解決策：制限された呼び出しを `system.run` / `system.runTimeout` / `system.runJob` に移動し、遅延実行する前に必要なイベントデータをキャプチャしておきます。

### 早期実行モード (Early-execution mode)

- ワールドが完全にロードされる前に実行されるため、多くのワールド API が利用できません。
- 解決策：ワールドに依存するコードを `world.afterEvents.worldLoad` に移動するか、`system.run` で遅延させてください。
- イベントの聞き逃しを防ぐため、イベントの購読（subscribe）はルートコンテキストで行ってください。

早期実行モードでも安全な API (ドキュメントより):

- `world.beforeEvents.*.subscribe/unsubscribe`
- `world.afterEvents.*.subscribe/unsubscribe`
- `system.beforeEvents.*.subscribe/unsubscribe`
- `system.afterEvents.*.subscribe/unsubscribe`
- `system.clearJob`, `system.clearRun`, `system.run`, `system.runInterval`, `system.runJob`, `system.runTimeout`, `system.waitTicks`
- `BlockComponentRegistry.registerCustomComponent`
- `ItemComponentRegistry.registerCustomComponent`

## カスタムコマンド

- `CustomCommand` インターフェースを使用してコマンドを定義します：`name`, `description`, `permissionLevel` に加え、`mandatoryParameters`（必須パラメータ）と `optionalParameters`（任意パラメータ）を指定します。
- パラメータには `CustomCommandParameter` を使用し、`name` と `type` を指定します（`CustomCommandParamType` が `Enum` の場合は `enumName` も必要です）。
- 列挙型（Enum）をパラメータで使用する前に、`CustomCommandRegistry.registerEnum(name, values)` で登録してください。
- コマンドは `CustomCommandRegistry.registerCommand(customCommand, callback)` で登録します。コールバックは `CustomCommandOrigin` とそれに続くパラメータを引数として受け取り、`CustomCommandResult` を返します。
- 関連ドキュメント：`CustomCommand`, `CustomCommandParameter`, `CustomCommandRegistry`, `CustomCommandOrigin`, `CustomCommandResult`, `CommandPermissionLevel`

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

## スクリプトイベント（Script events）

- `system.sendScriptEvent(id, message)` を使用してスクリプトイベントを発火させます。`id` は名前空間付きの識別子、`message` はペイロード文字列です。
- イベントの受信は `system.afterEvents.scriptEventReceive.subscribe(callback, options?)` で行います。
- コールバックの引数は `ScriptEventCommandMessageAfterEvent` 型で、`id`, `message`, `sourceType` およびオプションの `sourceEntity`/`sourceBlock`/`initiator` を含みます。
- `ScriptEventMessageFilterOptions.namespaces` を使用して、名前空間による受信イベントのフィルタリングが可能です。

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

## パフォーマンスの安全性

- イベント内での負荷の高い処理については、`system.runJob(generator)` を使用してタスクを複数のティックに分散させ、イベントパイプラインのブロッキングを回避してください。
- 膨大なエンティティやブロックのセットを反復処理する前に、ショートサーキットチェックを行ってください。

## system.run ガイドノート

- `system.run` はコールバックを次のティックに予約します。before-event からの読み取り専用の制限を回避して処理を実行するのに適しています。
- `system.runTimeout` は遅延ティック数を指定できます。`0` を指定すると同じティック内で実行されますが、誤用すると無限ループを引き起こす可能性があります。
- `system.runInterval` は `system.clearRun` で解除されるまで、N ティックごとに処理を繰り返します。
- `system.runJob` は長時間実行されるタスク向けです。細かく yield を行うジェネレーター関数を使用してください。
- ジェネレーターの各反復処理は、ジョブシステムがさまざまなデバイスでスケールできるよう、小さく一貫した負荷に保つようにしてください。

例（重い処理の遅延）:

```ts
world.beforeEvents.playerBreakBlock.subscribe((event) => {
  system.runJob(function* () {
    // ここで重い処理を実行
  });
});
```

## 型ガードと安全性

- 分岐ロジックには `typeId` チェックを使用し、アクセス前に期待するコンポーネントが存在することを確認してください。
- `instanceof` は、対象が `@minecraft/server` のクラスであり、かつドキュメントでそのように規定されている場合にのみ使用してください。

## 最小限のテンプレート

イベントの購読:

```ts
world.afterEvents.playerJoin.subscribe((event) => {
  const player = event.player;
});
```

ティックループ:

```ts
system.runInterval(() => {
  // ティックごとのロジック
}, 1);
```

ディメンションの取得:

```ts
const overworld = world.getDimension(MinecraftDimensionTypes.Overworld);
```

エンティティのスポーン:

```ts
const overworld = world.getDimension(MinecraftDimensionTypes.Overworld);
overworld.spawnEntity(MinecraftEntityTypes.Zombie, { x: 0, y: 80, z: 0 });
```

アイテムの付与:

```ts
const item = new ItemStack(MinecraftItemTypes.Diamond, 1);
player.getComponent("inventory")?.container?.addItem(item);
```

エフェクトの付与:

```ts
player.addEffect(MinecraftEffectTypes.Speed, 200, { amplifier: 1 });
```

コンポーネントの取得（型指定あり）:

```ts
const health = entity.getComponent(EntityComponentTypes.Health);
```

## よくある落とし穴

- **null コンポーネント**: プロパティやメソッドにアクセスする前に、必ずコンポーネントの存在を確認してください。
- **イベント内での重い処理**: `system.runJob` を使用して負荷の高い処理を遅延させ、ブロッキングを回避してください。
- **権限不足**: 読み取り専用モードや早期実行モードで操作が呼び出されていないことを確認してください。
- **生の文字列 ID**: 常に `@minecraft/vanilla-data` と `@minecraft/server` の型付き列挙型を優先してください。
- **一貫性のない名前空間プレフィックス**: アドオン内のすべてのカスタムコマンドとスクリプトイベントには、単一の一貫したプレフィックスを使用してください。

## Microsoft Docs MCP の使用

- MCP エンドポイント参照: https://learn.microsoft.com/api/mcp
- `microsoft_docs_search` を使用し、具体的なクエリで検索します：モジュール名 + クラス + タスク（例："@minecraft/server Player class", "Minecraft Script API BlockComponent", "world events @minecraft/server"）。
- 抜粋が短すぎる場合や完全なシグネチャが必要な場合は、`microsoft_docs_fetch` で取得します。
- 型や定義については、クエリに "class", "interface", "property", "method" などのキーワードを含めて検索してください。
