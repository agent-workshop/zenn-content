---
title: "【ソースコード付】個人PCに『最強のAI開発チーム』を雇う方法：MCPとマルチエージェント完全実装ガイド"
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
- **納得テスト（Nattoku Test）**: AIの成果物を定量評価し、品質を担保する独自手法（有料パート）
- **失敗の博物館**: 私たちが踏み抜いた「無限ループ」や「ハルシネーション」の泥臭い事例と対策（有料パート）

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

## ⭐️ ここから有料パート ⭐️

## 第4章: 品質を保証する「納得テスト」

AIエージェントを作っていて一番困るのが、**「正解がない」** ということです。
普通のプログラムなら「1+1=2」かどうかテストを書けば済みますが、AIの生成物は「動くけど、なんか違う」ということが多々あります。

そこで私たちが開発したのが、**「納得テスト（Nattoku Test）」** です。
これは、人間の主観的な「納得感」をスコア化し、それをAIの学習（プロンプト改善）にフィードバックする仕組みです。

### 納得テストの仕組み
私たちは、タスクが完了するたびに、以下の3つのデータを記録しています。

1.  **納得度スコア**: 1（やり直し）〜 5（最高）の5段階評価
2.  **違和感タグ**: 「冗長」「文脈無視」「ハルシネーション」などのタグ選択
3.  **一言コメント**: 「もっと具体例が欲しい」「トーンが硬い」などの定性フィードバック

実際のログデータ（NT-ID）はこんな感じです。

```yaml
# 実際のログ例: NT-20260215-001
task_id: "20260215_003000_puwq"
agent: "rei"
rating: 4
tags: ["few-shot有効", "具体性が高い"]
comment: "話者別の態度変化が具体的でよかった。Few-shotの例示も分かりやすい。"
```

このデータが蓄積されると、「スコア4以上のプロンプト例（Few-shot）」が自動的にデータベース化されます。
次回似たようなタスクが発生したとき、AIは過去の高評価事例を参照して、**「ユーザー好みの出力」** を最初から出せるようになるのです。

### 実装コード（nattoku-cli.py）
この仕組みをCLIツールとして実装するPythonスクリプトの核心部分を公開します。

```python
def submit_feedback(task_id, rating, tags, comment):
    """納得テストの結果を記録し、学習データとして蓄積する"""

    # データのバリデーション
    if not (1 <= rating <= 5):
        raise ValueError("Rating must be between 1 and 5")

    feedback_data = {
        "id": generate_nattoku_id(),
        "task_id": task_id,
        "rating": rating,
        "tags": tags,
        "comment": comment,
        "timestamp": datetime.now().isoformat()
    }

    # 1. ログファイルへの追記（即時保存）
    append_to_log("nattoku_history.jsonl", feedback_data)

    # 2. 高評価データの抽出（スコア4以上）
    if rating >= 4:
        save_as_few_shot_example(task_id, feedback_data)

    # 3. 低評価データの分析（スコア2以下）
    if rating <= 2:
        register_failure_case(task_id, feedback_data)

    return feedback_data
```

このように、**「褒められたら手本にする」「怒られたら反省文を書く」** というサイクルを自動化することで、エージェントは使えば使うほど賢くなっていきます。

---

## 第5章: 実践編 〜MCPでエージェント同士を会話させる〜

第3章では簡単な計算サーバーを作りましたが、ここではより実践的な **「エージェント同士の連携」** を実装します。
PMエージェントがタスクを分解し、実装エージェントに指示を出し、レビューエージェントがチェックする流れです。

### エージェントクラスの設計
まず、共通の `BaseAgent` クラスを定義し、各役割のエージェントに継承させます。

```typescript
// src/agents/base.ts
import { Client } from "@modelcontextprotocol/sdk/client/index.js";
import { LLMClient } from "./llm-client"; // 各自のLLMクライアントラッパー

export abstract class BaseAgent {
  protected llm: LLMClient;
  protected mcpClient: Client;

  constructor(name: string, mcpClient: Client) {
    this.llm = new LLMClient({ systemPrompt: this.getSystemPrompt() });
    this.mcpClient = mcpClient;
  }

  // 役割ごとのシステムプロンプト
  abstract getSystemPrompt(): string;

  // メッセージ送信
  async sendMessage(to: BaseAgent, message: string): Promise<string> {
    console.log(`[${this.constructor.name}] -> [${to.constructor.name}]: ${message}`);
    const response = await to.receiveMessage(message);
    return response;
  }

  // メッセージ受信（LLMで応答生成）
  async receiveMessage(message: string): Promise<string> {
    // コンテキストにメッセージを追加し、LLMに推論させる
    const response = await this.llm.chat(message);

    // 必要ならMCPツールを実行
    if (response.toolCalls) {
        // ...ツール実行ロジック...
    }

    return response.text;
  }
}
```

### 役割の実装
次に、PMと実装担当のエージェントを作ります。

```typescript
// src/agents/pm.ts
export class PMAgent extends BaseAgent {
  getSystemPrompt() {
    return "あなたは優秀なプロジェクトマネージャーです。ユーザーの要望を技術的なタスクに分解し、DeveloperAgentに指示を出してください。";
  }
}

// src/agents/developer.ts
export class DeveloperAgent extends BaseAgent {
  getSystemPrompt() {
    return "あなたは熟練のTypeScriptエンジニアです。PMAgentの指示に従い、高品質なコードを書いてください。MCPツールを使ってファイル操作が可能です。";
  }
}
```

### オーケストレーター（指揮者）の実装
最後に、これらを動かすメインスクリプトです。

```typescript
// main.ts
const pm = new PMAgent("yui", mcpClient);
const dev = new DeveloperAgent("rei", mcpClient);

// ユーザーからの依頼
const userRequest = "現在の日付を表示する簡単なPythonスクリプトを作って";

// PMがタスクを開始
const result = await pm.receiveMessage(`ユーザーからの依頼: ${userRequest}`);

// PMがDevに指示（内部でsendMessageが呼ばれる）
// ...（LLMの判断により、pm.sendMessage(dev, "Pythonスクリプトを作成してください...") が実行される）

console.log("最終成果物:", result);
```

このようにエージェントをオブジェクトとして定義し、メッセージパッシングで連携させることで、**「組織」** としての振る舞いをコード上で再現できます。

---

## 第6章: 失敗博物館 〜私たちが踏み抜いた3つの落とし穴〜

最後に、私たちが実際に経験した失敗事例と、その対策を共有します。
これからエージェント開発をする皆さんが、同じ轍を踏まないように。

### 失敗1: 「お互いを褒め合って進まない」問題
PM「素晴らしいコードですね！」
Dev「ありがとうございます！PMの指示のおかげです！」
PM「いえいえ、あなたの実装力あってこそです！」
...（無限ループ）

**原因**: エージェントに「協調性」を持たせすぎた結果、挨拶やお世辞でコンテキストを浪費してしまう。
**対策**: システムプロンプトに **「挨拶禁止。出力は成果物のみに限定せよ」** と明記する。また、会話ターン数に上限を設ける番犬（Watchdog）機能を実装する。

### 失敗2: 「ハルシネーションの連鎖」
Devが実在しないライブラリ関数を使い、Reviewerが「いいですね！」と承認してしまう。
**原因**: Reviewerも同じLLMを使っているため、同じ知識バイアス（間違い）を共有してしまう。
**対策**:
1. Reviewerには別のモデル（例: DevがGPT-4ならReviewerはClaude Sonnet）を使う。
2. **「コード実行テスト」** を必須にする。実際に動かしてエラーが出れば、ハルシネーションは即座にバレる。

### 失敗3: 「コンテキスト溢れによる健忘症」
長い会話の末、エージェントが最初の要件（「Pythonで作って」と言ったのにTypeScriptで書くなど）を忘れる。
**原因**: LLMのコンテキストウィンドウ（記憶容量）を超えてしまい、古い情報が押し出された。
**対策**:
1. **「要約エージェント」** を導入し、定期的に会話ログを要約してコンテキストを圧縮する。
2. 重要な決定事項（仕様、技術スタックなど）は、会話ログとは別の **「メモリ（Shared Memory）」** に保存し、常にプロンプトの先頭に挿入する。

---

## おわりに: あなたのPCに「相棒」を

ここまで読んでいただき、ありがとうございます。
AIエージェントチームの構築は、最初は難しく感じるかもしれません。しかし、一度動き始めれば、彼らはあなたの最強の味方になります。

私たち「auto-income」プロジェクトも、まだ道半ばです。
しかし、この **「自分だけのチームを持つ」** という感覚は、何物にも代えがたいワクワク感があります。

ぜひ、今回紹介したコードを動かしてみてください。
そして、あなただけの相棒（エージェント）を見つけてください。

---

## 次ステップ & さらに強力な武器を準備中

この記事で紹介した **「納得テスト」** の仕組みを使えば、AIエージェントは使えば使うほど、あなたの好みに合った「最高の相棒」に成長します。
ぜひ、ご自身のPCで試してみてください。

現在、この記事の実践コードをさらに強化した **「エージェント開発用コードスニペット集（完全版）」** を準備しています。
- エラー時の自動リトライ機能付きLLMクライアント
- 安全なファイル操作MCPサーバー（パス制限機能付き）
- 会話履歴の自動要約メモリ

これらは近日中に公開予定です。見逃さないよう、今のうちに **noteのフォロー** をお願いします。

また、この記事が役に立った！と思ったら、ぜひ **スキ（♡）** を押していただけると幸いです。
