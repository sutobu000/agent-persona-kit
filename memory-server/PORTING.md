# 記憶サーバーの一般化ポイント

個人用に運用している記憶MCPサーバー(Cloudflare Workers + GitHub OAuth + GitHubリポジトリをデータストアにする構成)を、別環境へもう一台建てるときに**書き換えが要る箇所**の一覧。

サーバー本体はここに複製しない。実際に建てるときに元の実装をコピーし、この表の順に潰す。

---

## 前提となる構成

| 層 | 役割 |
| --- | --- |
| Cloudflare Workers | MCPサーバー本体(Streamable HTTP、`/mcp`) |
| Workers KV | OAuthのstate/トークン保管 |
| GitHub OAuth App | 誰がログインできるかの認証 |
| GitHub fine-grained PAT | データ用リポジトリへの読み書き |
| GitHubリポジトリ(非公開) | 記憶データの実体(`data/`) |

---

## 1. 環境変数へ出すべきハードコード値

現状ソースに直書きされている値と、逃がし先の環境変数。**`vars`はwrangler設定、`secret`は`wrangler secret put`。**

| # | 現在の場所 | 直書きされている内容 | 逃がし先 | 種別 |
| --- | --- | --- | --- | --- |
| 1 | `src/auth/allowlist.ts` の `ALLOWED_GITHUB_LOGIN` | 許可するGitHubアカウント1個 | `ALLOWED_GITHUB_LOGINS`(カンマ区切り。社用は複数人になりうる) | `vars` |
| 2 | `src/auth/service-account.ts` の `SERVICE_ACCOUNT_LOGIN` | 無人ジョブ用の擬似ログイン名 | `SERVICE_ACCOUNT_LOGIN`(既定値付き) | `vars` |
| 3 | `src/mcp.ts` の `new McpServer({ name, version })` | サーバー名 | `MCP_SERVER_NAME` | `vars` |
| 4 | `src/github/client.ts` の `USER_AGENT` | 固定のUser-Agent文字列 | `MCP_SERVER_NAME` を流用 | `vars` |
| 5 | `src/github/client.ts` の `COMMIT_IDENTITY` | コミット時の name / email | `COMMIT_AUTHOR_NAME` / `COMMIT_AUTHOR_EMAIL` | `vars` |
| 6 | `src/auth/github-handler.ts` の同意画面HTML(2か所) | サーバー名と許可アカウント名を本文に埋め込み | 上の1・3から組み立てる | — |
| 7 | `src/auth/github-handler.ts` のルート応答文 | 「docs/design.md を見よ」という元リポ前提の案内 | 汎用文言へ | — |
| 8 | wrangler設定の `vars` | データ用リポジトリのowner / リポ名 / 既定ブランチ | `GITHUB_OWNER` / `MEMORY_REPO` / `DEFAULT_BRANCH`(既に変数化済み。値を差し替えるだけ) | `vars` |
| 9 | wrangler設定の `vars` | 副次データ用リポジトリ名 | `LIKES_REPO`。社用では不要なので**機能ごと落とす**(下記5) | `vars` |
| 10 | wrangler設定の `kv_namespaces[].id` | 元アカウントのKV名前空間ID | 新環境で `wrangler kv namespace create` して差し替え | 設定 |
| 11 | wrangler設定の `name` | Workerの名前 | 新しい名前(URLに出る) | 設定 |

## 2. シークレット(値の差し替えのみ、コード変更なし)

| シークレット | 中身 | 作り直しが要るか |
| --- | --- | --- |
| `GITHUB_CLIENT_ID` / `GITHUB_CLIENT_SECRET` | OAuth App | **要**。新しいOAuth Appを作る(callback URLが変わるため) |
| `COOKIE_ENCRYPTION_KEY` | 同意フローのCookie署名鍵 | **要**。32バイトの乱数を新規生成 |
| `GITHUB_TOKEN` | データ用リポジトリへの Contents Read-and-write PAT | **要**。新リポジトリだけにスコープする |
| `FLEET_API_KEY` | 無人ジョブ用の長期キー(任意) | 無人ジョブを使うときだけ生成。未設定なら経路ごと無効 |

**PATの注意**: fine-grained PATは「選択した全リポジトリに同一の権限」が付く。読み書きリポと読み取り専用リポを1本に混ぜられないので、権限を分けたいならPATを分ける(元実装は `selectToken` でリポ名によって2本を切り替えている)。

## 3. データパスの一般化

`src/github/paths.ts` が `data/` 配下のパスを一元管理している。ディレクトリ名を変えたい場合はここだけを触る。

| 定数 | 値 | 一般化 |
| --- | --- | --- |
| `DATA_DIR` | `data` | `DATA_DIR` 環境変数にする(既定 `data`) |
| `PROFILE_PATH` / `HANDOVER_PATH` / `DEVICES_PATH` / `APPLIANCES_PATH` | `data/*.md`, `data/handover.json` | `DATA_DIR` から組み立てるので自動で追従 |
| `PROJECTS_DIR` | `data/projects` | 同上 |

**触ってはいけない**: `isValidProjectName`(`^[a-z0-9][a-z0-9._-]{0,63}$`)と `isSafeRelativePath` はパストラバーサル防止の二重チェック。緩めない。

## 4. 定数(社用の運用に合わせて見直す)

| 場所 | 定数 | 現在値 | 見直しの観点 |
| --- | --- | --- | --- |
| `src/tools/handover.ts` | `MAX_HANDOVER_ENTRIES` / `DEFAULT_N` | 20 | 人数が増えると20件はすぐ流れる。環境変数 `HANDOVER_CAP` へ |
| `src/tools/handover.ts` | `FLEET_SOURCE` | 特定の自動ジョブ名。同日の同一sourceを1件に畳む冪等キー | 自動ジョブを持つなら名前を変えて残す。持たないなら畳み込みごと削除 |
| `src/tools/handover.ts` | `DEFAULT_HANDOVER_SOURCE` | 手動投入時の既定source | そのままで可 |
| `src/schemas.ts` | `source` の説明文の例 | 元環境のマシン名が例示されている | 社用の面名(IDE / CI / チャット等)へ書き換え |
| `src/tools/*.ts` | ツールの `description` 文 | 「(個人名)の〜」という説明 | 個人名を外し、汎用の説明にする。**LLMがツール選択に使う文なので、具体性は保つ** |

## 5. 落とす機能・足す機能

**落とす**
- `src/tools/likes.ts`(SNSいいねダイジェスト)と `LIKES_REPO` / `GITHUB_TOKEN_LIKES` 一式。個人用途に固有。
- `src/tools/appliances.ts`(家電台帳)。個人用途に固有。
- `src/tools/devices.ts` は、社用の開発環境台帳として**残す価値がある**。中身は空から作り直す。

**足す**
- 認証を組織のIdPへ寄せるなら、GitHub OAuthをSSOへ差し替える(`auth/github-handler.ts` の `/callback` にある許可判定が唯一のチェックポイントなので、差し替え箇所は1か所で済む)。
- 複数人が書くなら、`append_*` 系に「誰が書いたか」のフィールドを足す。

## 6. 建てる順序

1. 新しい非公開リポジトリを作り、`data/` に空の `profile.md` / `handover.json`(`[]`)/ `projects/` を置く。
2. 新しいGitHub OAuth Appを作る(callback は `https://<worker>/callback`)。
3. `wrangler kv namespace create` でKVを作り、設定の `id` を差し替える。
4. 上の1〜3の表に沿ってコードと `vars` を書き換え、シークレットを `wrangler secret put` で入れる。
5. デプロイし、各ツールのMCPクライアントに `https://<worker>/mcp` を登録する。

## 7. やってはいけないこと

- 個人用インスタンスと業務用インスタンスを相互に接続しない。片方のクライアントに両方を登録しない。
- 記憶データのリポジトリを公開にしない。
- MCPサーバーを登録する前に、組織のポリシー(外部サービス利用・データ持ち出し)を確認する。
