# クロスツール連携(Claude Code ↔ Codex)

**Claude Code を Lead、Codex を作業員として使う**ときの実務レシピ。すべて 2026-08-31 に一次情報で確認した仕様に基づく。確認できなかったものは「未確認」と明記する。

同じスコープ(社用なら社用)の中での連携だけを扱う。**私用と社用をまたぐ接続はしない**(README「データ分離の原則」)。

---

## パターン① Claude=Lead が `codex exec` を作業員として呼ぶ

いちばん素直で、依存が少ない。Codex を**1回きりの非対話プロセス**として呼び、結果を Lead が検証する。

### `codex exec` の実仕様(確認済み)

| 項目 | 内容 |
| --- | --- |
| 呼び出し | `codex exec "プロンプト"` — プロンプトは**引数1個**。`codex exec -` で標準入力から全文を読む |
| 出力の分離 | **進捗は stderr、最終メッセージだけが stdout**。スクリプトから拾うのはこれ |
| モデル指定 | `--model, -m <name>` |
| サンドボックス | `--sandbox, -s` = `read-only` \| `workspace-write` \| `danger-full-access` |
| 作業ディレクトリ | `--cd, -C <path>` |
| 機械可読出力 | `--json`(JSONL。`thread.started` / `turn.started` / `item.*` / `turn.completed`。トークン使用量を含む) |
| 最終メッセージの保存 | `--output-last-message, -o <path>` |
| 構造化出力 | `--output-schema <path>` |
| 継続 | `codex exec resume --last "追加の指示"` / `codex exec resume <SESSION_ID>` |
| 画像入力 | `--image, -i path[,path...]` |
| その他 | `--ephemeral`(セッションを残さない)、`--skip-git-repo-check` |
| 承認の全バイパス | `--dangerously-bypass-approvals-and-sandbox`(別名 `--yolo`)。**隔離環境でだけ使う** |
| `--full-auto` | 互換のため残っている**非推奨**フラグ。現行は `--sandbox workspace-write` を使う |
| **終了コード** | **未確認**。公式ドキュメントに終了コードの規約が書かれていない |

出典: <https://learn.chatgpt.com/docs/non-interactive-mode> / <https://learn.chatgpt.com/docs/developer-commands>

### 委譲の書き方(テンプレート)

サブエージェントに渡すときと同じ3点セット(**成果物の形 / 完了条件 / 触ってよい範囲**)を、そのままプロンプトとフラグに割り当てる。

```bash
codex exec \
  --cd /path/to/repo \
  --sandbox workspace-write \
  --json \
  --output-last-message /tmp/codex-result.md \
  "以下を実施してください。

  【成果物】src/parser.ts のテストを全て通す修正。
  【完了条件】npm test が exit 0。テストの期待値は変更しない。
  【触ってよい範囲】src/parser.ts のみ。他のファイルは読むだけ。
  【やらないこと】リファクタリング、依存の追加、git commit。

  最後に、変更した行と理由を箇条書きで報告してください。"
```

- **範囲の制限は、プロンプトだけに頼らず `--sandbox` と `--cd` で機構的にも掛ける。**人の指示だけを防波堤にしない(`access-boundary` の考え方と同じ)。
- `--json` と `--output-last-message` は併用する。前者で経過とトークン量、後者で最終報告を確実に取る。

### 成果の検証(Lead 側)

**Codex の報告をそのまま結論にしない。** 返ってきた文章は主張であって、事実ではない。

1. **完了条件を Lead 自身が実行して確かめる**(上の例なら `npm test` を Lead が回す)。
2. **差分を見る**(`git diff`)。触ってよい範囲の外が変わっていないか確認する。
3. 終了コードの規約が未確認なので、**成否の判定は終了コードに頼らない。** `--output-last-message` の中身と、①②の実測で判定する。
4. 想定外の変更があれば `git checkout` で戻し、指示を絞って再実行する。

## パターン② MCP でつなぐ

### Codex を MCP サーバーとして公開できるか

**できる。ただし公式に非推奨。**

- `codex mcp-server` が stdio 上の MCP サーバーとして起動し、`codex`(セッション開始。`prompt` / `approval-policy` / `sandbox` / `config` を受ける)と `codex-reply`(`threadId` で継続)の2ツールを公開する。
- **このコマンドは公式ドキュメントで DEPRECATED と明記**され、代わりに「Codex app server」または「**Claude Code 用の Codex プラグイン**」が案内されている。新規に組むならそちらを先に検討する。
- 別の MCP クライアントへ登録するときは stdio サーバーとして `command: "codex"`, `args: ["mcp-server"]` の形になるが、**Claude Code 向けの具体的な登録JSONは公式に記載がなく未確認。**
- 動作確認だけなら `npx @modelcontextprotocol/inspector codex mcp-server`。
- `codex app-server` は**別物**で非推奨ではないが、公式に「MCPサーバーの代替ではない」と明記された独自のJSON-RPCインターフェース。

出典: <https://learn.chatgpt.com/docs/mcp-server> / <https://learn.chatgpt.com/docs/app-server>

**推奨**: 非推奨コマンドに寄りかからず、**パターン①(`codex exec`)を既定にする。** MCP で繋ぐ必要が出たら、まず Codex プラグイン for Claude Code の現況を確認する。

### 逆向き: Codex から MCP サーバーを使う(こちらは非推奨ではない)

Codex は MCP **クライアント**としては現役。`~/.codex/config.toml`(またはプロジェクトの `.codex/config.toml`)に `[mcp_servers.<name>]` を書く。

- **stdio**: `command`(必須)、`args`、`env`、`env_vars`、`cwd`
- **リモート(Streamable HTTP)**: `command` の代わりに `url`。`auth`(`oauth` / `chatgpt`)、`bearer_token_env_var`、`http_headers`、`env_http_headers`
- 共通: `startup_timeout_sec`(既定10秒)、`tool_timeout_sec`(既定60秒)、`enabled`、`required`、`enabled_tools` / `disabled_tools`
- CLI から: `codex mcp add <name> -- <command>` / `codex mcp list` / `codex mcp login <name>`

出典: <https://learn.chatgpt.com/docs/extend/mcp> / <https://learn.chatgpt.com/docs/config-file/config-reference>

つまり、**記憶MCPサーバーを Claude Code と Codex の両方から使う構成は成立する**(両者ともMCPクライアントとして remote HTTP をサポートする)。Copilot だけは OAuth のリモートMCPが非対応なので外れる(README参照)。

## パターン③ 直接つながない疎結合

いちばん壊れにくい。ツール同士を直結せず、**中間の成果物で受け渡す。**

| 受け渡し先 | 使いどころ | 注意 |
| --- | --- | --- |
| **ファイル** | 同じマシン・同じリポジトリでの分担 | 書き込む範囲を事前に決め、`git diff` で検証する |
| **ブランチ / PR** | レビューを挟みたいとき。Codex に実装、Claude にレビューをさせる等 | PR本文に「成果物・完了条件・触ってよい範囲」を書いておくと、そのまま検証の基準になる |
| **共有の記憶MCP** | 面をまたいだ引き継ぎ(申し送り・ノート) | **同一スコープ内なら、記憶サーバーの共有は分離原則に反しない** |

### 記憶サーバーの共有について

**分離原則が禁じているのは「私用の記憶」と「社用の記憶」をつなぐこと**であって、同じスコープ内で複数のツールが同じ記憶サーバーを読み書きすることではない。

- 社用の Claude Code と社用の Codex が、社用の記憶MCPサーバーを共有する → **問題ない。むしろ推奨。**同一人格を面をまたいで保つための本来の使い方。
- 私用の記憶MCPを社用の環境に登録する → **禁止。**
- 判断基準は「そのデータが漏れて困る相手が、向こう側にいるか」。

---

## 社用PC初日のチェックリスト

**入れる前に確認する。**入れてから聞くと戻せない。

### 1. 会社ポリシー(最優先・人に聞く)

- [ ] 外部AIコーディングツールの利用可否。**どのツールが承認済みか。**
- [ ] ソースコードを外部サービスへ送信してよいか。除外すべきリポジトリはあるか。
- [ ] MCPサーバー(特に外部ホストのリモートMCP)の利用可否。
- [ ] 記憶データの保存先として、外部のGitホスティングやクラウドを使ってよいか。
- [ ] 生成物の扱い(著作権・監査ログの要否)。

**ここが通らなければ以降は不要。**キットを入れず、ポリシーの範囲内でできることに絞る。

### 2. Codex の実仕様(手を動かして確認)

- [ ] `codex --version` — インストールされているか、版はいくつか。
- [ ] `codex exec --help` — 上の表のフラグが**この版に実在するか**。ドキュメントと版がずれることがある。
- [ ] `codex exec --sandbox read-only "この作業ディレクトリの構成を1行で説明して"` — 最小の疎通確認。
- [ ] `--json` の出力が期待どおりか。`--output-last-message` でファイルが作られるか。
- [ ] **終了コードの挙動を自分で測る**(公式に規約がないため)。わざと失敗させて `echo $?` を見る。
- [ ] `~/.codex/config.toml` が既に組織配布されていないか。上書きしてよいか。

### 3. MCP 対応の確認

- [ ] Claude Code から社用の記憶MCPへ接続できるか(`claude mcp list`)。
- [ ] Codex の `[mcp_servers]` 設定が組織ポリシーで許可されているか。
- [ ] Copilot を使うなら、**OAuthのリモートMCPは非対応**である点が要件とぶつからないか。
- [ ] `codex mcp-server` を使う計画なら、**非推奨である点**を踏まえて代替(Codexプラグイン for Claude Code)を先に調べる。

### 4. キットの配置

- [ ] `kit.config.json` を社用の値で埋める(特に `SCOPE_LABEL` `RESOURCE_RULES` `MODEL_DEFAULT`)。
- [ ] `node build/generate.mjs` を実行し、警告が出ないことを確認する。
- [ ] 生成物を各ツールの置き場所へ配置する(README「社用に建てる手順」)。
- [ ] **私用の記憶MCPが登録されていないことを確認する。**

---

## 未確認事項

| 項目 | 状況 |
| --- | --- |
| `codex exec` の終了コードの規約 | 公式ドキュメントに記載なし。**実機で測る**こと |
| `codex mcp-server` の Claude Code 向け登録JSONの正確な書式 | 公式に記載なし |
| 「Codex プラグイン for Claude Code」の詳細仕様 | 非推奨案内の中で言及されているのみ。**未調査** |
| Codex サブエージェントの同時実行数・入れ子の深さの既定値 | 公式ページに数値の記載なし |
