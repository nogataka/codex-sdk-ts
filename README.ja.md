# codex-sdk-ts

OpenAI Codex エージェント向け軽量 TypeScript SDK。

これは公式 `@openai/codex-sdk` のフォークで、**Codex バイナリをバンドルしていません**。代わりに、システムにインストールされた `codex` コマンドを PATH から検索します。

## 互換性

| 項目 | バージョン |
|------|---------|
| **ベース** | [@openai/codex-sdk](https://github.com/openai/codex/tree/main/sdk/typescript) @ コミット `9327e99b2` |
| **Codex CLI** | v0.98.0 以上（`--json` フラグのサポートが必要） |
| **Node.js** | 18 以上 |

この SDK は上流の OpenAI Codex SDK と定期的に同期されています。更新状況は [upstream sync workflow](.github/workflows/upstream-sync.yml) をご確認ください。

## 関連 SDK

| 言語 | パッケージ | リポジトリ |
|----------|---------|------------|
| **TypeScript** | [`codex-sdk-ts`](https://www.npmjs.com/package/codex-sdk-ts) | [nogataka/codex-sdk-ts](https://github.com/nogataka/codex-sdk-ts) |
| **Python** | [`codex-sdk-py`](https://pypi.org/project/codex-sdk-py/) | [nogataka/codex-sdk-py](https://github.com/nogataka/codex-sdk-py) |

両 SDK は同一の機能と API 設計を持っています。お好みの言語をお選びください。

## 前提条件

この SDK を使用する前に、Codex CLI がインストールされ、システムの PATH で利用可能である必要があります:

```bash
# Codex CLI のインストール（詳細は https://github.com/openai/codex を参照）
npm install -g @openai/codex
```

## インストール

```bash
npm install codex-sdk-ts
```

Node.js 18 以上が必要です。

## クイックスタート

```typescript
import { Codex } from "codex-sdk-ts";

const codex = new Codex();
const thread = codex.startThread();
const turn = await thread.run("テスト失敗を診断して修正案を提案して");

console.log(turn.finalResponse);
console.log(turn.items);
```

同じ `Thread` インスタンスで `run()` を繰り返し呼び出すことで、会話を継続できます。

```typescript
const nextTurn = await thread.run("修正を実装して");
```

### ストリーミングレスポンス

`run()` はターン終了までイベントをバッファリングします。ツール呼び出し、ストリーミングレスポンス、ファイル変更通知などの中間進捗に反応するには、代わりに `runStreamed()` を使用してください。構造化イベントの非同期ジェネレータを返します。

```typescript
const { events } = await thread.runStreamed("テスト失敗を診断して修正案を提案して");

for await (const event of events) {
  switch (event.type) {
    case "item.completed":
      console.log("item", event.item);
      break;
    case "turn.completed":
      console.log("usage", event.usage);
      break;
  }
}
```

### 構造化出力

Codex エージェントは、指定されたスキーマに準拠した JSON レスポンスを生成できます。スキーマは各ターンにプレーンな JSON オブジェクトとして提供できます。

```typescript
const schema = {
  type: "object",
  properties: {
    summary: { type: "string" },
    status: { type: "string", enum: ["ok", "action_required"] },
  },
  required: ["summary", "status"],
  additionalProperties: false,
} as const;

const turn = await thread.run("リポジトリのステータスを要約して", { outputSchema: schema });
console.log(turn.finalResponse);
```

[Zod スキーマ](https://github.com/colinhacks/zod) から [`zod-to-json-schema`](https://www.npmjs.com/package/zod-to-json-schema) パッケージを使用して JSON スキーマを作成し、`target` を `"openAi"` に設定することもできます。

```typescript
const schema = z.object({
  summary: z.string(),
  status: z.enum(["ok", "action_required"]),
});

const turn = await thread.run("リポジトリのステータスを要約して", {
  outputSchema: zodToJsonSchema(schema, { target: "openAi" }),
});
console.log(turn.finalResponse);
```

### 画像の添付

テキストと一緒に画像を含める場合は、構造化入力エントリを提供します。テキストエントリは最終プロンプトに連結され、画像エントリは `--image` 経由で Codex CLI に渡されます。

```typescript
const turn = await thread.run([
  { type: "text", text: "これらのスクリーンショットを説明して" },
  { type: "local_image", path: "./ui.png" },
  { type: "local_image", path: "./diagram.jpg" },
]);
```

### 既存スレッドの再開

スレッドは `~/.codex/sessions` に保存されます。メモリ上の `Thread` オブジェクトを失った場合は、`resumeThread()` で再構築して続行できます。

```typescript
const savedThreadId = process.env.CODEX_THREAD_ID!;
const thread = codex.resumeThread(savedThreadId);
await thread.run("修正を実装して");
```

### 作業ディレクトリの制御

Codex はデフォルトでカレントディレクトリで実行されます。回復不能なエラーを避けるため、Codex は作業ディレクトリが Git リポジトリであることを要求します。スレッド作成時に `skipGitRepoCheck` オプションを渡すことで、Git リポジトリチェックをスキップできます。

```typescript
const thread = codex.startThread({
  workingDirectory: "/path/to/project",
  skipGitRepoCheck: true,
});
```

### Codex CLI 環境の制御

デフォルトでは、Codex CLI は Node.js プロセス環境を継承します。`Codex` クライアントのインスタンス化時にオプションの `env` パラメータを提供することで、CLI が受け取る変数を完全に制御できます。Electron アプリなどのサンドボックス化されたホストに便利です。

```typescript
const codex = new Codex({
  env: {
    PATH: "/usr/local/bin",
  },
});
```

SDK は、提供した環境の上に必要な変数（`OPENAI_BASE_URL` や `CODEX_API_KEY` など）を引き続き注入します。

### `--config` オーバーライドの渡し方

`config` オプションを使用して、追加の Codex CLI 設定オーバーライドを提供します。SDK は JSON オブジェクトを受け取り、ドット区切りのパスにフラット化し、値を TOML リテラルとしてシリアル化してから、繰り返しの `--config key=value` フラグとして渡します。

```typescript
const codex = new Codex({
  config: {
    show_raw_agent_reasoning: true,
    sandbox_workspace_write: { network_access: true },
  },
});
```

スレッドオプションは、グローバルオーバーライドの後に出力されるため、重複する設定に対して優先されます。

### MCP サーバー設定

[Model Context Protocol (MCP)](https://modelcontextprotocol.io/) サーバーを設定して、外部ツールで Codex の機能を拡張します。

```typescript
const codex = new Codex({
  config: {
    mcp_servers: {
      playwright: {
        command: "npx",
        args: ["-y", "@playwright/mcp@latest"]
      },
      filesystem: {
        command: "npx",
        args: ["-y", "@anthropic/mcp-server-filesystem@latest", "/path/to/dir"]
      }
    }
  }
});
```

`mcp_servers` 設定はインライン TOML テーブルとしてシリアル化され、グローバル Codex 設定を変更することなく MCP サーバーを動的に設定できます。

## ライセンス

Apache-2.0
