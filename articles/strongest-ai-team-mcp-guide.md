---
title: "MCPで自律型AIチームを作る実装ログ【要点版】"
emoji: "🤖"
type: "tech"
topics: ["ai", "mcp", "multiagent", "typescript", "bun"]
published: true
---

## はじめに: なぜ「AIチーム」なのか？

「AIにコードを書かせてみたけど、結局自分で直した方が早かった」
「チャットボットと話しているうちに、文脈が溢れて前の指示を忘れてしまった」

生成AIを使って開発をしていると、誰もが一度はこの壁にぶつかります。
私たち「auto-income」プロジェクトも、最初はそうでした。

しかし、ある日気づいたのです。
**「1人の天才AIにすべてを任せるのではなく、役割を持った凡人AIたちをチームとして働かせればいいのではないか？」**

これが、私たちが「AIエージェントチーム」を構築し、現在進行形で収益化実験を行っている理由です。
本記事では、机上の空論ではなく、実際に私たちのシステム（TypeScript + Bun）で稼働しているコードをベースに、**「あなたのPC上に、あなただけの専属AI開発チームを構築する方法」** を解説します。

キーワードは **MCP (Model Context Protocol)**。
2024年にAnthropic社が提唱したこの新標準プロトコルこそが、AIエージェントを「チャットボット」から「頼れる同僚」へと進化させる鍵です。

### この記事で得られるもの
- **自律型AIチームの設計思想**: なぜPM・実装・レビューに分けるのか？
- **MCPの実践的活用**: ツールやメモリをエージェント間で共有する具体的なコード
- **最小構成のMCPサーバー実装**: Bun + TypeScriptで動くサンプルコード

:::message
本記事は要点版です。品質保証手法「納得テスト」の実装コード、エージェント間連携の実践コード、失敗事例と対策を含む**完全版**は[noteで公開中](https://note.com/agent_workshop/n/n9217c45f117e?utm_source=zenn&utm_medium=article&utm_campaign=mcp_guide)です。
:::

さあ、あなたのPCに「最強のチーム」を迎え入れましょう。

---

## 第1章: AIエージェントは「1人」より「チーム」が強い

### 単体エージェントの限界
ChatGPTやClaudeのWeb UIを使っているとき、こんな経験はありませんか？
「要件定義をして、設計をして、コードを書いて、テストもして」と一度に頼むと、後半のタスクがおざなりになる現象。

これは、LLM（大規模言語モデル）の **「注意機構（Attention）」の限界** です。
一度に処理できるコンテキスト（文脈）には限りがあり、タスクが複雑になればなるほど、モデルの注意力は散漫になります。

### 役割分担（Role-based）のメリット
そこで有効なのが、**「役割分担」** です。
人間社会でも、1人で企画・営業・開発・経理をこなすのは困難ですが、チームなら可能です。AIも同じです。

私たちのチーム構成は以下の通りです。

1.  **PMエージェント（ゆい）**: 全体の進行管理、タスクの分解、優先順位付け。
2.  **実装エージェント（れい）**: 具体的なコードの記述、リファクタリング。
3.  **レビューエージェント（みゆ）**: コードの品質チェック、バグ発見、設計の妥当性評価。
4.  **リサーチエージェント（ひな）**: 外部情報の収集、トレンド調査。

このように役割を分けることで、各エージェントは「自分の仕事」に集中でき、結果として全体のパフォーマンスが向上します。
特に重要なのが **「相互レビュー」** です。実装エージェントが書いたコードを、別の視点（レビューエージェント）がチェックすることで、単純なミスやハルシネーション（嘘の出力）を劇的に減らすことができます。

---

## 第2章: MCPでつなぐ脳と手足

### MCP (Model Context Protocol) とは？
「MCP」という言葉を聞いたことはありますか？
これは、**AIモデル（脳）と、外部のデータやツール（手足）をつなぐための標準規格** です。

これまで、AIに外部ツールを使わせるには、LangChainや各社の独自Function Callingを駆使して、複雑なグルーミングコードを書く必要がありました。
しかし、MCPを使えば、**「MCPサーバー」として定義したツールやリソースを、あらゆるMCP対応クライアント（Claude Desktopアプリや、自作のエージェント）から統一的に利用できる** ようになります。

### なぜHTTP APIじゃダメなのか？
「普通のREST APIでいいじゃん」と思うかもしれません。
しかし、MCPにはAI特化の利点があります。

1.  **プロンプト情報の自動提供**: サーバー側から「このツールはどう使うべきか」という説明（プロンプト）を動的に提供できます。
2.  **リソースの抽象化**: ファイルの中身、DBのレコード、APIのレスポンスなどを「リソース」として統一的に扱えます。
3.  **ステートフルな接続**: JSON-RPCベースの双方向通信により、セッションを通じた対話的なやり取りが可能です。

### サーバーとクライアントの関係
私たちのシステムでは、以下のような構成でMCPを活用しています。

- **MCPサーバー**: ファイル操作、SQLiteデータベース操作、Web検索、ブラウザ操作などの「機能」を提供。
- **MCPクライアント**: 各AIエージェント（ゆい、れい、みゆ...）。必要に応じてサーバーに接続し、道具を使う。

例えば、「日次レポートを作成する」というタスクの場合：
1. PMエージェントが「データベース閲覧ツール」を使って売上データを取得。
2. 実装エージェントが「ファイル操作ツール」を使ってMarkdownファイルを作成。
3. レビューエージェントが「ファイル読み込みツール」を使って内容を確認。

このように、MCPサーバーを介してエージェントたちが「同じ道具」「同じデータ」を共有することで、スムーズな連携が可能になります。

---

## 第3章: 実装の第一歩 〜最小構成のMCPサーバー〜

では、実際にコードを書いてみましょう。
ここでは、TypeScriptとBunを使って、**「簡単な計算機能を提供するMCPサーバー」** を作ります。

### プロジェクトのセットアップ
まず、プロジェクトを作成し、必要なライブラリをインストールします。
私たちは `bun` を推奨しますが、`npm` でも同様に可能です。

```bash
mkdir mcp-demo
cd mcp-demo
bun init
bun add @modelcontextprotocol/sdk zod
```

### サーバーの実装 (server.ts)
`@modelcontextprotocol/sdk` を使えば、驚くほど簡単にサーバーが書けます。

```typescript
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import { z } from "zod";

// サーバーインスタンスの作成
const server = new McpServer({
  name: "calc-server",
  version: "1.0.0",
});

// ツールの登録: 加算
server.tool(
  "add",
  "2つの数値を足し算します",
  {
    a: z.number().describe("1つ目の数値"),
    b: z.number().describe("2つ目の数値"),
  },
  async ({ a, b }) => {
    return {
      content: [{ type: "text", text: String(a + b) }],
    };
  }
);

// サーバーの起動 (標準入出力を使用)
const transport = new StdioServerTransport();
await server.connect(transport);

console.error("Calculator MCP Server running on stdio");
```

### クライアントからの利用 (client.ts)
次に、このサーバーを利用するクライアント（エージェント側）のコードです。
ここではシンプルに、ツールを呼び出す部分だけを実装します。

```typescript
import { Client } from "@modelcontextprotocol/sdk/client/index.js";
import { StdioClientTransport } from "@modelcontextprotocol/sdk/client/stdio.js";

// サーバープロセスをサブプロセスとして起動
const transport = new StdioClientTransport({
  command: "bun",
  args: ["run", "server.ts"],
});

const client = new Client(
  { name: "example-client", version: "1.0.0" },
  { capabilities: {} }
);

await client.connect(transport);

// ツール一覧の取得
const tools = await client.listTools();
console.log("Available tools:", tools.tools.map(t => t.name));

// ツールの実行
const result = await client.callTool({
  name: "add",
  arguments: { a: 10, b: 20 },
});

console.log("Result:", result.content[0].text); // "30"
```

### 実行してみる
```bash
bun run client.ts
```
これだけで、**「クライアントがサーバーのツールを認識し、引数を渡して実行し、結果を受け取る」** という一連の流れが完了しました。

実際の開発では、このクライアント部分に LLM（OpenAI APIやAnthropic API）を組み込み、**「LLMがツールを選んで実行する」** 仕組みを作ります。
それこそが、AIエージェントの「手足」となるのです。

---

## この先の内容

ここまでで、MCPの基本概念と最小構成のサーバー実装を体験できました。

実際のプロジェクトでは、ここからさらに以下の課題に取り組む必要があります：

- **品質保証**: AIの出力を定量評価する仕組み（私たちは「納得テスト」と呼んでいます）
- **エージェント間連携**: PM→実装→レビューの連携コード
- **失敗対策**: 無限ループ、ハルシネーション連鎖、コンテキスト溢れへの対処法

これらの実装コード・失敗事例・対策を含む**完全版ガイド（全6章）** は、noteで公開しています。

:::message alert
**完全版はこちら**
[【ソースコード付】個人PCに「最強のAI開発チーム」を雇う方法：MCPとマルチエージェント完全実装ガイド](https://note.com/agent_workshop/n/n9217c45f117e?utm_source=zenn&utm_medium=article&utm_campaign=mcp_guide)

- 第4章: 品質保証「納得テスト」の全貌と実装コード
- 第5章: MCPでエージェント同士を会話させる実践コード
- 第6章: 失敗博物館 — 踏み抜いた3つの落とし穴と対策
:::
