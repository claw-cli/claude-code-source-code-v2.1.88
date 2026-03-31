# Claude Code（流出ソースコード アーカイブ）

[English](./README.md) | [中文](./README_ZH.md) | [繁體中文](./README_ZH_TW.md) | [조선어](./README_KO.md) | 日本語 | [Español](./README_ES.md) | [Русский](./README_RU.md)

<p align="center">
  <img src="https://img.shields.io/badge/status-archived%20%2B%20rebuilding-blue" />
  <img src="https://img.shields.io/badge/runtime-Bun%20%2F%20Node-black" />
  <img src="https://img.shields.io/badge/language-TypeScript%20→%20Rust-orange" />
  <img src="https://img.shields.io/badge/focus-Code%20Agent%20Systems-green" />
</p>

---

## 🚨 UPDATE → Rust で再構築中

> **流出した Claude Code をため込むだけでなく、より良い harness ツーリングを作ろう。  
コードを集めるな。成果を作れ。**

私たちは現在、**Rust で Claude Code を再構築**しており、**より強力で、より信頼性が高く、本番運用に耐えるコードエージェントシステム**を作ることを目指しています。

👉 Rust 実装と最新の進捗に興味がある方はこちら:  
https://github.com/claw-cli/claw-code-cli

---

## 📦 このリポジトリは何か？

このリポジトリは、npm パッケージに同梱された sourcemap を通じて公開アクセス可能になった **Claude Code v2.1.88 のソースコードのアーカイブスナップショット**です。

これは以下の役割を果たします:

- 📚 実世界の AI エージェントシステムを研究するための**研究アーティファクト**  
- 🔍 大規模 LLM ツーリングアーキテクチャの**参照実装**  
- 🧪 **より良いシステムを再構築するための土台**

---

2026 年 3 月 31 日、Chaofan Shou は異変に気づきました。Claude Code のソースコード全体が、npm にひっそり公開されていたのです。しかも、パッケージに含まれていた sourcemap ファイルの中に隠されていました。

[![流出を知らせたツイート](https://raw.githubusercontent.com/kuberwastaken/claude-code/main/public/leak-tweet.png)](https://raw.githubusercontent.com/kuberwastaken/claude-code/main/public/leak-tweet.png)

このリポジトリはその瞬間を記録しています。公開されてしまったコードのアーカイブであると同時に、この流出がどのように起きたのか、そしてその裏側のシステムについて何が分かるのかをまとめたものでもあります。

では見ていきましょう。

## これはどうやって起きたのか？

ここは本当に一度立ち止まって考えさせられる部分です。

JavaScript や TypeScript のパッケージを npm に公開する際、現代のビルドパイプラインでは通常 **source map ファイル**（`.map`）が生成されます。これらのファイルは、バンドル済み・最小化済みの本番出力と元のソースコードの橋渡しをします。目的は単純で、本番環境で何か壊れたときに、スタックトレースを圧縮された bundle 内の読みにくい位置ではなく、元のソースの正確な行に対応付けられるようにすることです。

ただし重要なのは、**source map には元のソースコードそのものが埋め込まれていることが多い**という点です。参照ではなく、JSON 構造の中に文字列として保存された生のコードそのものです。

典型的な `.map` ファイルは次のような見た目です:

```json
{
  "version": 3,
  "sources": ["../src/main.tsx", "../src/tools/BashTool.ts", "..."],
  "sourcesContent": ["// 各ファイルの元ソース全文", "..."],
  "mappings": "AAAA,SAAS,OAAO..."
}
````

鍵になるのは `sourcesContent` フィールドです。ここには元の各ファイルの完全な内容を含めることができます。コード、コメント、内部定数、プロンプト、すべてです。このファイルが公開されれば、実質的にコードベース全体も一緒に配布されたことになります。

そして npm はそれを問題なく配布します。`npm pack` を実行したり、tarball を調べたり、パッケージ内容を閲覧したりする人は、誰でも直接アクセスできます。

これは新しい種類の問題ではありません。過去にも何度も起きていますし、今後も起きる可能性が高いです。根本原因はたいてい単純で、`*.map` ファイルが `.npmignore` で除外されていないか、あるいは bundler が本番ビルドで source map 生成を無効化するよう設定されていないだけです。

このケースでは、プロジェクトは Bun を使ってビルドされていました。Bun は明示的に無効化しない限りデフォルトで source map を生成するため、この種の露出は見落としやすくなります。

![npm パッケージ内で露出した Claude Code のソースファイル](https://raw.githubusercontent.com/kuberwastaken/claude-code/main/public/claude-files.png)](https://raw.githubusercontent.com/kuberwastaken/claude-code/main/public/claude-files.png)

特に皮肉なのは、“Undercover Mode” という内部システムが存在していたことです。これはコミットや PR 説明のような生成物を通じて機密情報が漏れるのを防ぐために設計されていました。

AI がテキスト中で内部情報をうっかり露出しないようにかなりの努力が払われていたのに、結局コードベース全体がビルド成果物経由で公開されてしまったのです。

---

## Claude の内部はどうなっているのか？

最近ずっと岩の下にいたのでなければ、Claude Code が Anthropic 公式の Claude 用コーディング CLI ツールであり、最も人気のある AI コーディングエージェントだということはご存じでしょう。

外から見ると、洗練されてはいるものの比較的シンプルな CLI に見えます。

中身を見ると、それは **785KB の [`main.tsx`](https://github.com/kuberwastaken/claude-code/blob/main/main.tsx)** エントリポイントに、カスタム React ターミナルレンダラ、40 以上のツール、マルチエージェントオーケストレーションシステム、“dream” と呼ばれるバックグラウンドのメモリ統合エンジンなどが詰め込まれたものです。

前置きはこのくらいにして、半日かけてソースを深掘りして見つけた本当に面白い部分をいくつか紹介します。

---

## BUDDY - ターミナルの中のたまごっち

冗談ではありません。

Claude Code には "Buddy" と呼ばれる完全な **たまごっち風コンパニオンペットシステム** があります。**決定的なガチャシステム**があり、種族レアリティ、色違いバリアント、手続き的に生成されるステータス、そして OpenClaw のように初回孵化時に Claude が書くソウル説明まで備えています。

全体は [`buddy/`](https://github.com/kuberwastaken/claude-code/tree/main/buddy) にあり、`BUDDY` コンパイル時 feature flag の背後に隠されています。

### ガチャシステム

あなたの buddy の種族は **Mulberry32 PRNG** によって決まります。これは高速な 32 ビット疑似乱数生成器で、`userId` のハッシュに `'friend-2026-401'` という salt を加えた値を seed にしています:

```typescript
// Mulberry32 PRNG - ユーザーごとに決定的で再現可能
function mulberry32(seed: number): () => number {
  return function() {
    seed |= 0; seed = seed + 0x6D2B79F5 | 0;
    var t = Math.imul(seed ^ seed >>> 15, 1 | seed);
    t = t + Math.imul(t ^ t >>> 7, 61 | t) ^ t;
    return ((t ^ t >>> 14) >>> 0) / 4294967296;
  }
}
```

同じユーザーは常に同じ buddy を引きます。

### 18 種類の種族（コード上では難読化）

種族名は `String.fromCharCode()` 配列で隠されており、Anthropic はこれらが文字列検索に引っかからないようにしたかったのが明らかです。デコードすると、完全な種族リストは以下の通りです:

| レアリティ | 種族 |
|--------|---------|
| **Common** (60%) | Pebblecrab, Dustbunny, Mossfrog, Twigling, Dewdrop, Puddlefish |
| **Uncommon** (25%) | Cloudferret, Gustowl, Bramblebear, Thornfox |
| **Rare** (10%) | Crystaldrake, Deepstag, Lavapup |
| **Epic** (4%) | Stormwyrm, Voidcat, Aetherling |
| **Legendary** (1%) | Cosmoshale, Nebulynx |

さらに、レアリティとは完全に独立した **1% の色違い確率** もあります。つまり、色違いの Legendary Nebulynx が出る確率は **0.01%** です。すごい。

### ステータス、目、帽子、そしてソウル

各 buddy には手続き的に以下が生成されます:
- **5 つのステータス**: `DEBUGGING`, `PATIENCE`, `CHAOS`, `WISDOM`, `SNARK`（各 0-100）
- **6 種類の目のスタイル** と **8 種類の帽子オプション**（一部はレアリティで制限）
- **“ソウル”**: 前述の通り、初回孵化時に Claude がキャラクターになりきって生成する性格説明

スプライトは **高さ 5 行、幅 12 文字の ASCII アート** として複数フレームで描画されます。待機アニメーション、反応アニメーションがあり、入力プロンプトの隣に座ります。

### Lore

コードでは 2026 年 4 月 1 日から 7 日を**ティーザー期間**として参照しており（たぶんイースター向け？）、本格リリースは 2026 年 5 月にゲートされています。このコンパニオンには次のようなシステムプロンプトがあります:

```
A small {species} named {name} sits beside the user's input box and 
occasionally comments in a speech bubble. You're not {name} - it's a 
separate watcher.
```

つまり単なる飾りではなく、buddy は独自の個性を持ち、名前で呼ばれると返答することもできます。本当にリリースしてほしいです。

---

## KAIROS - “常時オンの Claude”

[`assistant/`](https://github.com/kuberwastaken/claude-code/tree/main/assistant) の中には **KAIROS** という完全なモードがあります。つまり、入力を待たずに動き続ける、永続的で常時稼働の Claude アシスタントです。監視し、記録し、気づいたことに**能動的に**対処します。

これは `PROACTIVE` / `KAIROS` コンパイル時 feature flag の背後にあり、外部ビルドには完全に存在しません。

### 仕組み

KAIROS は **追記専用の日次ログファイル** を維持します。1 日を通して観察、判断、行動を書き込み続けます。定期的に `<tick>` プロンプトを受け取り、能動的に行動するか静かにしているかを決めます。

システムには **15 秒のブロッキング予算** があり、ユーザーのワークフローを 15 秒以上止めるような能動行動は後回しにされます。これは、Claude がうるさくならずに役立とうとしているということです。

### Brief モード

KAIROS が有効なときには、**Brief** という特別な出力モードがあります。これは非常に簡潔な応答で、常駐アシスタントがターミナルを埋め尽くさないようにするためのものです。おしゃべりな友人と、本当に価値があることだけを話すプロのアシスタントの違いだと思ってください。

### 専用ツール

KAIROS は通常の Claude Code が持っていないツールを使えます:

| Tool | 何をするか |
|------|-------------|
| **SendUserFile** | ファイルを直接ユーザーに送る（通知、要約） |
| **PushNotification** | ユーザーのデバイスに push 通知を送る |
| **SubscribePR** | PR アクティビティを購読・監視する |

 ---

## ULTRAPLAN - 30 分間のリモート計画セッション

これはインフラ観点でもかなりワイルドです。

**ULTRAPLAN** は、Claude Code が複雑な計画タスクを **Opus 4.6** が動作する **リモート Cloud Container Runtime（CCR）セッション** にオフロードし、最大 **30 分** 考えさせ、その結果をブラウザから承認できるモードです。

基本的な流れ:

1. Claude Code が深い計画を必要とするタスクを特定する
2. `tengu_ultraplan_model` 設定を介してリモート CCR セッションを起動する
3. ターミナルにはポーリング状態が表示され、**3 秒ごと**に結果を確認する
4. その間、ブラウザベースの UI で計画の進行を見守り、承認または拒否できる
5. 承認されると、特別な sentinel 値 `__ULTRAPLAN_TELEPORT_LOCAL__` によって結果がローカル端末へ“テレポート”される

---

## “Dream” システム - Claude は文字通り夢を見る

これは本当に、この中でも最もクールなものの一つです。

Claude Code には **autoDream**（[`services/autoDream/`](https://github.com/kuberwastaken/claude-code/tree/main/services/autoDream)）と呼ばれるシステムがあります。**fork された subagent** として動作するバックグラウンドのメモリ統合エンジンです。この命名は非常に意図的です。Claude が……夢を見ているのです。

これはとても面白くて、[私も先週 LITMUS で同じようなアイデアを考えていたからです。OpenClaw の subagent に余暇を与えて面白い新論文を探させるというものです](https://github.com/Kuberwastaken/litmus)

### 三段階のトリガー

dream は気分で勝手に走るわけではありません。**三段階のトリガーシステム**があります:

1. **時間ゲート**: 前回の dream から 24 時間経過
2. **セッションゲート**: 前回の dream 以降、少なくとも 5 セッション  
3. **ロックゲート**: 統合ロックを取得する（同時実行の dream を防ぐ）

この 3 つすべてを満たす必要があります。これにより、dream のやり過ぎも不足も防がれます。

### 4 つのフェーズ

実行されると、dream は [`consolidationPrompt.ts`](https://github.com/kuberwastaken/claude-code/blob/main/services/autoDream/consolidationPrompt.ts) のプロンプトに従って 4 つの厳格なフェーズを踏みます:

**Phase 1 - Orient**: メモリディレクトリで `ls` を実行し、`MEMORY.md` を読み、既存のトピックファイルをざっと見て改善箇所を探す。

**Phase 2 - Gather Recent Signal**: 永続化する価値のある新しい情報を見つける。優先順位は daily logs → drifted memories → transcript search。

**Phase 3 - Consolidate**: メモリファイルを書き込みまたは更新する。相対日付を絶対日付に変換する。矛盾した事実を削除する。

**Phase 4 - Prune and Index**: `MEMORY.md` を 200 行未満かつ約 25KB に保つ。古いポインタを削除する。矛盾を解消する。

プロンプトには文字通りこう書かれています:

> *"You are performing a dream - a reflective pass over your memory files. Synthesize what you've learned recently into durable, well-organized memories so that future sessions can orient quickly."*

dream subagent には **読み取り専用 bash** が与えられます。プロジェクトを見ることはできますが、何も変更できません。純粋にメモリ統合のための処理です。

---

## 完全なツールレジストリ - 40 以上のツール

Claude Code のツールシステムは [`tools/`](https://github.com/kuberwastaken/claude-code/tree/main/tools) にあります。完全な一覧は次の通りです:

| Tool | 何をするか |
|------|-------------|
| **AgentTool** | 子エージェント / subagent を生成する |
| **BashTool** / **PowerShellTool** | シェル実行（任意でサンドボックス化） |
| **FileReadTool** / **FileEditTool** / **FileWriteTool** | ファイル操作 |
| **GlobTool** / **GrepTool** | ファイル検索（利用可能ならネイティブ `bfs`/`ugrep` を使用） |
| **WebFetchTool** / **WebSearchTool** / **WebBrowserTool** | Web アクセス |
| **NotebookEditTool** | Jupyter notebook 編集 |
| **SkillTool** | ユーザー定義スキルを呼び出す |
| **REPLTool** | 対話型 VM shell（bare mode） |
| **LSPTool** | Language Server Protocol 通信 |
| **AskUserQuestionTool** | ユーザーに入力を求める |
| **EnterPlanModeTool** / **ExitPlanModeV2Tool** | Plan mode 制御 |
| **BriefTool** | ファイルを claude.ai にアップロード/要約する |
| **SendMessageTool** / **TeamCreateTool** / **TeamDeleteTool** | Agent swarm 管理 |
| **TaskCreateTool** / **TaskGetTool** / **TaskListTool** / **TaskUpdateTool** / **TaskOutputTool** / **TaskStopTool** | バックグラウンドタスク管理 |
| **TodoWriteTool** | todo を書く（legacy） |
| **ListMcpResourcesTool** / **ReadMcpResourceTool** | MCP リソースアクセス |
| **SleepTool** | 非同期遅延 |
| **SnipTool** | 履歴スニペット抽出 |
| **ToolSearchTool** | ツール探索 |
| **ListPeersTool** | peer agents を列挙する（UDS inbox） |
| **MonitorTool** | MCP サーバーを監視する |
| **EnterWorktreeTool** / **ExitWorktreeTool** | Git worktree 管理 |
| **ScheduleCronTool** | cron ジョブをスケジュールする |
| **RemoteTriggerTool** | リモート agents を起動する |
| **WorkflowTool** | workflow スクリプトを実行する |
| **ConfigTool** | 設定を変更する（**internal only**） |
| **TungstenTool** | 高度な機能（**internal only**） |
| **SendUserFile** / **PushNotification** / **SubscribePR** | KAIROS 専用ツール |

ツールは `getAllBaseTools()` を通じて登録され、feature gate、ユーザー種別、環境フラグ、権限拒否ルールによって絞り込まれます。さらに、JSON schema をキャッシュしてプロンプト効率を上げる **tool schema cache**（[`toolSchemaCache.ts`](https://github.com/kuberwastaken/claude-code/blob/main/tools/toolSchemaCache.ts)）もあります。

---

## 権限とセキュリティシステム

[`tools/permissions/`](https://github.com/kuberwastaken/claude-code/tree/main/tools/permissions) にある Claude Code の権限システムは、単なる “allow/deny” よりはるかに高度です:

**Permission Modes**: `default`（対話式プロンプト）、`auto`（transcript classifier による ML ベースの自動承認）、`bypass`（チェックをスキップ）、`yolo`（すべて拒否。名前は皮肉）

**Risk Classification**: すべてのツール操作は **LOW**、**MEDIUM**、**HIGH** リスクに分類されます。さらに、**YOLO classifier** という高速な ML ベースの権限判定システムがあり、自動的に決定します。

**Protected Files**: `.gitconfig`、`.bashrc`、`.zshrc`、`.mcp.json`、`.claude.json` などは自動編集から保護されています。

**Path Traversal Prevention**: URL エンコードされた traversal、Unicode 正規化攻撃、バックスラッシュ注入、大文字小文字を無視したパス操作など、すべて対処されています。

**Permission Explainer**: ユーザーが承認する前に、別の LLM 呼び出しでツールのリスクを説明します。Claude が「このコマンドはあなたの git config を変更します」と言うとき、その説明文自体も Claude によって生成されています。

---

## 隠された Beta Header と未公開 API 機能

[`constants/betas.ts`](https://github.com/kuberwastaken/claude-code/blob/main/constants/betas.ts) には、Claude Code が API と交渉しているすべての beta 機能が明らかになっています:

```typescript
'interleaved-thinking-2025-05-14'      // Extended thinking
'context-1m-2025-08-07'                // 1M token context window
'structured-outputs-2025-12-15'        // Structured output format
'web-search-2025-03-05'                // Web search
'advanced-tool-use-2025-11-20'         // Advanced tool use
'effort-2025-11-24'                    // Effort level control
'task-budgets-2026-03-13'              // Task budget management
'prompt-caching-scope-2026-01-05'      // Prompt cache scoping
'fast-mode-2026-02-01'                 // Fast mode (Penguin)
'redact-thinking-2026-02-12'           // Redacted thinking
'token-efficient-tools-2026-03-28'     // Token-efficient tool schemas
'afk-mode-2026-01-31'                  // AFK mode
'cli-internal-2026-02-09'             // Internal-only (ant)
'advisor-tool-2026-03-01'              // Advisor tool
'summarize-connector-text-2026-03-13'  // Connector text summarization
```

`redact-thinking`、`afk-mode`、`advisor-tool` もまだ公開されていません。

---

## Feature Gating - 内部ビルド vs 外部ビルド

これはコードベースの中でも、アーキテクチャ的に最も興味深い部分のひとつです。

Claude Code は Bun の `bun:bundle` にある `feature()` 関数を使った **コンパイル時 feature flag** を利用しています。bundler はそれらを **定数畳み込み** し、外部ビルドではゲートされた分岐を **デッドコード除去** します。既知の flag 一覧は次の通りです:

| Flag | 制御するもの |
|------|--------------|
| `PROACTIVE` / `KAIROS` | 常時オンのアシスタントモード |
| `KAIROS_BRIEF` | Brief コマンド |
| `BRIDGE_MODE` | claude.ai 経由のリモート操作 |
| `DAEMON` | バックグラウンドデーモンモード |
| `VOICE_MODE` | 音声入力 |
| `WORKFLOW_SCRIPTS` | Workflow 自動化 |
| `COORDINATOR_MODE` | マルチエージェントオーケストレーション |
| `TRANSCRIPT_CLASSIFIER` | AFK mode（ML 自動承認） |
| `BUDDY` | コンパニオンペットシステム |
| `NATIVE_CLIENT_ATTESTATION` | クライアント証明 |
| `HISTORY_SNIP` | 履歴切り抜き |
| `EXPERIMENTAL_SKILL_SEARCH` | スキル探索 |

加えて、`USER_TYPE === 'ant'` は Anthropic 内部機能をゲートします。staging API アクセス（`claude-ai.staging.ant.dev`）、内部 beta header、Undercover mode、`/security-review` コマンド、`ConfigTool`、`TungstenTool`、そして `~/.config/claude/dump-prompts/` へのデバッグプロンプト出力です。

**GrowthBook** は実行時 feature gate を扱っており、かなり攻めたキャッシュ値を使います。`tengu_` プレフィックスの feature flag は fast mode から memory consolidation まであらゆるものを制御しています。多くのチェックでは、メインループを止めないために `getFeatureValue_CACHED_MAY_BE_STALE()` が使われており、feature gate に関しては古いデータでも許容されています。

---

## その他の注目ポイント

### Upstream Proxy
[`upstreamproxy/`](https://github.com/kuberwastaken/claude-code/tree/main/upstreamproxy) ディレクトリには、コンテナ対応の proxy relay があり、**`prctl(PR_SET_DUMPABLE, 0)`** を使って同一 UID によるヒープメモリの ptrace を防いでいます。CCR コンテナ内の `/run/ccr/session_token` からセッショントークンを読み取り、CA 証明書をダウンロードし、ローカルの CONNECT→WebSocket relay を起動します。Anthropic API、GitHub、npmjs.org、pypi.org は明示的に proxy 対象外です。

### Bridge Mode
[`bridge/`](https://github.com/kuberwastaken/claude-code/tree/main/bridge) には、claude.ai と統合するための JWT 認証付き bridge システムがあります。作業モードは `'single-session'` | `'worktree'` | `'same-dir'` をサポートしています。より高いセキュリティ階層向けに trusted device token も含まれています。

### Migration 内のモデルコードネーム
[`migrations/`](https://github.com/kuberwastaken/claude-code/tree/main/migrations) ディレクトリには内部コードネームの履歴が見えます:
- `migrateFennecToOpus` - **"Fennec"**（フェネック）が Opus のコードネームだった
- `migrateSonnet1mToSonnet45` - 1M コンテキスト付き Sonnet が Sonnet 4.5 になった
- `migrateSonnet45ToSonnet46` - Sonnet 4.5 → Sonnet 4.6
- `resetProToOpusDefault` - Pro ユーザーはある時点で Opus デフォルトに戻された

### Attribution Header
すべての API リクエストには以下が含まれます:
```
x-anthropic-billing-header: cc_version={VERSION}.{FINGERPRINT}; 
  cc_entrypoint={ENTRYPOINT}; cch={ATTESTATION_PLACEHOLDER}; cc_workload={WORKLOAD};
```
`NATIVE_CLIENT_ATTESTATION` 機能により、Bun の HTTP スタックが `cch=00000` プレースホルダを計算済みハッシュで上書きできます。つまり Anthropic が、そのリクエストが本物の Claude Code インストールから来たものか確認するためのクライアント真正性チェックです。

### Computer Use - “Chicago”
Claude Code には、内部コードネーム **"Chicago"** の完全な Computer Use 実装が含まれており、`@ant/computer-use-mcp` の上に構築されています。スクリーンショット取得、クリック/キーボード入力、座標変換を提供します。Max/Pro サブスクリプションに限定されており（内部ユーザーには ant bypass あり）、利用できます。

### Pricing
気になる人のために言うと、[`utils/modelCost.ts`](https://github.com/kuberwastaken/claude-code/blob/main/utils/modelCost.ts) にあるすべての価格は [Anthropic の公開価格表](https://docs.anthropic.com/en/docs/about-claude/models) と完全に一致しています。ここにニュース性はありません。

---

## ⚡ TL;DR

- Claude Code は単なる CLI ではなく、**完全なエージェントプラットフォーム**
- 含まれるもの:
  - マルチエージェントオーケストレーション
  - バックグラウンドメモリ（“Dream システム”）
  - ツールエコシステム（40 以上のツール）
  - 能動的アシスタント（KAIROS）
- 多用されているもの:
  - Feature flags
  - 実行時 + コンパイル時ゲーティング
  - システム設計としての Prompt engineering

---

## 🧩 主要なアーキテクチャ洞察

### 1. マルチエージェントオーケストレーション

Claude Code は **Coordinator Mode** で動作し、複数の worker を起動できます:

- 並列リサーチ
- 中央集権的プランニング
- 分散実行
- 検証ループ

> 並列性は第一級のプリミティブとして扱われています。

---

### 2. ツーリングシステム（40 以上のツール）

次を含む豊富なツールエコシステム:

- ファイルシステム操作
- シェル実行
- Web ブラウジング
- タスクスケジューリング
- エージェント間通信

すべてのツールは:
- Schema 主導
- 権限ゲート付き
- 動的に有効化

---

### 3. メモリシステム（“Dream”）

バックグラウンドプロセスが:

- 定期的に実行され
- 知識を統合し
- 古い情報を刈り込み
- 長期記憶を維持する

> これは本質的に **LLM memory compression + reflection** です。

---

### 4. 能動的エージェント（KAIROS）

常時オンのアシスタントが:

- 活動を観察し
- 振る舞いを記録し
- 明示的なプロンプトなしに行動する

これは **agent → system evolution** です。

---

### 5. Feature Gating とビルド戦略

- コンパイル時フラグ（Bun 経由）
- デッドコード除去
- 内部ビルドと外部ビルドの分離

これにより以下が可能になります:
- 隠し機能
- 段階的ロールアウト
- 内部実験

---

## 🧠 私たちが学んだこと

- AI コーディングツールは、単なるアシスタントではなく**オペレーティングシステム**になりつつある
- Prompt engineering = **システムアーキテクチャ**
- メモリ + ツール + オーケストレーション = **本物のエージェント**
- 本番システムは次に大きく依存している:
  - Guardrails
  - Permissions
  - Observability

---

## 🛠 なぜ Rust で再構築するのか？

このコードベースは強力ですが:

- ❌ 複雑で保守が難しい
- ❌ JS ランタイムの制約がある
- ❌ 性能保証が弱い
- ❌ 並行性の推論が難しい

次世代エージェントシステムは次のようであるべきだと私たちは考えています:

- ⚡ より速い（ネイティブ性能）
- 🔒 より安全（メモリ + 実行）
- 🧵 設計レベルで並行性を備える
- 📦 配布に向いている（CLI + インフラ）

---

## 🚀 私たちの方向性

私たちが構築しているのは:

> 単なる CLI ラッパーではなく、**次世代のコードエージェントランタイム**です。

注力分野:

- 決定的なエージェント実行
- より良いツールサンドボックス
- 第一級のマルチエージェントオーケストレーション
- 本物のメモリシステム（prompt hack ではない）
- Bun / npm に優しい配布パイプライン

---

## 📦 ビルドと配布

現在取り組んでいるもの:

- ✅ Bun ベースのビルドパイプライン
- 📦 npm 配布
- ⚡ Bun ネイティブ実行

目標:

> シームレスなインストール、即時実行、ゼロフリクション。

---

## ⚠️ 免責事項

このリポジトリの目的:

- 研究
- 教育
- リバースエンジニアリング上の知見

私たちは元の Claude Code の**所有権を主張しません**。

---

## ⭐ 最後に

このリポジトリはアーカイブとして始まりました。

今では **発射台** です。

> コードを集めるな。  
> 実際に届けられるシステムを作れ。
