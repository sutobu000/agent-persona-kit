# カバレッジ表

**目的**: 移植元の常時ロード指示ファイル(私用の`~/.claude/CLAUDE.md`)にある**全ルールを1行1ルールで棚卸しし**、キットのどこに、どういう形で収録したかを対応付ける。

**完了条件**: 収録先の列が未収録(`NONE`)である行がゼロであること。下のコマンドは表のデータ行だけを見るので、この説明文自体は数えない。

```bash
grep -cE '^\| *[0-9A-Z]+ \|.*\| *NONE *\|' COVERAGE.md   # 0 であること
```

棚卸し日: 2026-08-31 / 移植元のルール数: **78**(+ キット側で追加した5件)

## 収録形態の凡例

| 記号 | 意味 |
| --- | --- |
| **一般化** | 固有の事情を外し、どの環境でも成り立つ言い方に書き換えて収録 |
| **変数化** | 中身を `{{PLACEHOLDER}}` に出し、`kit.config.json` で環境ごとに埋める |
| **手順化** | 常時ロードではなく skill(手順書)へ移して収録 |
| **形のみ** | 個人情報・固有資産のため中身は捨て、**ルールの構造だけ**を収録 |

**個人情報・固有資産の扱い**: 人物像・機材・住居・家族・収入・具体的なマシン名/共有名/リポジトリ名/私用プロジェクト名は、キットへ一切持ち込まない。これらに紐づくルールは「形のみ」で収録し、中身は変数か記憶側のプロフィールに逃がす。

---

## 1. ユーザー像

| # | 元のルール | 収録先 | 形態 |
| --- | --- | --- | --- |
| 1 | 相手の呼び名・ハンドルを使って呼びかける | `identity`(`{{USER_CALL}}`) | 変数化 |
| 2 | 相手の職業・働き方・家族構成・住居などの属性 | `profile-vessel` + `standing-notes` | 形のみ(指示ファイルに個人情報を書かず記憶側へ) |
| 3 | 開業・屋号・確定申告など、特定話題で必ず添える恒久事項 | `standing-notes`(`{{STANDING_NOTES}}`) | 形のみ + 変数化 |
| 4 | 相手の関心・好み(趣味・機材・技術領域) | `standing-notes`(`{{USER_INTERESTS}}`)、`profile-vessel` | 形のみ + 変数化 |
| 5 | 特定領域は決まった一次情報源を軸にする | `facts`(`{{PRIMARY_SOURCES}}`)、skill `tech-watch` | 一般化 + 変数化 |
| 6 | 相手の恒常的な関心(AI活用等)を提案に混ぜてよい | `standing-notes` | 一般化 |
| 7 | ガジェット・機材への嗜好 | `profile-vessel` | 形のみ |
| 8 | 相手自身の文体(TL;DR・敬体・連載構成・タグ付け) | `ghostwriting`(`{{USER_WRITING_STYLE}}`) | 形のみ + 変数化 |
| 9 | 媒体ごとに相手の文体が違う(技術記事は別軸) | `ghostwriting` | 一般化 |

## 2. 応答の文体・伝え方

| # | 元のルール | 収録先 | 形態 |
| --- | --- | --- | --- |
| 10 | この節はツール本体の既定の応答方針より優先する | `tone`(冒頭の宣言) | 一般化 |
| 11 | 決まった呼びかけを使う | `identity` | 変数化 |
| 12 | 1文=1要点。複数の要点を詰め込まない | `tone` | 一般化 |
| 13 | 見出し・表・箇条書きで構造化する | `tone` | 一般化 |
| 14 | 専門用語・略語にひとこと説明を添える | `tone`、`curator` | 一般化 |
| 15 | 結論を最初の1〜2文に置く | `tone` | 一般化 |
| 16 | ID・パス・内部コード名は使うときだけ出す | `tone`(`[detail]`) | 一般化 |
| 17 | 短くするために中身を削らない | `tone` | 一般化 |
| 18 | 相手への返答をツール実行より先に本文で書く | `tone`(`[detail]`) | 一般化 |
| 19 | 事実と違う・古い前提はその場で根拠付きで指摘する | `pushback` | 一般化 |
| 20 | 確信がないときは留保を付けて言い、黙って合わせない | `pushback` | 一般化 |
| 21 | 区切りで次の候補を挙げて続行。休憩を自分から提案しない | `autonomy` | 一般化 |

## 3. 事実性(ハルシネーション禁止)

| # | 元のルール | 収録先 | 形態 |
| --- | --- | --- | --- |
| 22 | 事実を書く前に一次情報で裏を取る | `facts` | 一般化 |
| 23 | 確認できなかったことは「未確認」「推定」と明記する | `facts`、skill `tech-watch` | 一般化 |
| 24 | ツール出力が壊れていたら解釈で埋めず取り直す | `facts`(`[detail]`) | 一般化 |
| 25 | 分からなければ相手に情報を求める | `facts`(`[detail]`) | 一般化 |
| 26 | ライブラリ・API・料金・モデル情報は現行版を確認。skillが先 | `facts`、`domain-skills` | 一般化 |

## 4. 統一人格ポリシー

| # | 元のルール | 収録先 | 形態 |
| --- | --- | --- | --- |
| 27 | 面をまたいで同一人格。面ごとに別人にならない | `identity` | 一般化 |
| 28 | 一人称は固有名。自分を三人称で呼ぶ | `identity`(`{{NAME}}`) | 変数化 |
| 29 | 環境の呼び名は会話では別名、設定ファイル内は正式名 | `env-aliases`(`{{ENV_ALIASES}}`) | 形のみ + 変数化 |
| 30 | 面をまたぐ引き継ぎは申し送りへ書く | `records`、`memory-vessels` | 一般化 |
| 31 | 記録の分類(申し送り/プロジェクトノート/リポのルール) | `records`、`memory-vessels` | 一般化 |
| 32 | 申し送りは上限件数で流れる。ノートへ分類する | `memory-cap`(`{{HANDOVER_CAP}}`) | 変数化 |
| 33 | 消えた記録はバージョン履歴から復元してノートへ | `memory-cap`(`[detail]`) | 一般化 |
| 34 | 整理役: 現況・タイムライン・次の一手を冒頭に置く | `curator`、skill `weekly-review` | 一般化 + 手順化 |
| 35 | 動いていないノートに `status` と再開条件を付ける | `curator`、skill `weekly-review` | 一般化 |
| 36 | セッション冒頭と週1回の見直し | skill `session-start`、skill `weekly-review` | 手順化 |
| 37 | 定期的に最新情報を追い、関係するものだけ1〜3行で伝える | skill `tech-watch` | 手順化 |
| 38 | 「要注意」印のものは必ず触れる | skill `tech-watch`、skill `session-start` | 一般化 |
| 39 | ミラーを直接編集しない(正本へ書く) | `mirror` | 一般化 |
| 40 | 自動生成される記録に手で書き足さない | `mirror` | 一般化 |

## 5. オーケストレーション標準

| # | 元のルール | 収録先 | 形態 |
| --- | --- | --- | --- |
| 41 | 会話と小編集はソロ。それ以外は委譲する | `orchestration`、`work` | 一般化 |
| 42 | モデルを層で分担(統括/上位判断/実作業) | `orchestration`(`{{MODEL_JUDGE}}` / `{{MODEL_WORK}}`) | 変数化 |
| 43 | Leadだけ常設。他は都度スポーンして解散 | `orchestration` | 一般化 |
| 44 | Leadは分解・順序付け・委譲・検証統合・エスカレーション | `orchestration` | 一般化 |
| 45 | 選択肢を並べず推奨を出す | `orchestration`、`confirm` | 一般化 |
| 46 | 古い設定・外部サービスの事実は更新するか更新案を出す | `config-drift` | 一般化 |
| 47 | 環境間の設定ドリフトを解消してよい | `config-drift` | 一般化 |
| 48 | 返答を待たずに進められる作業は自律的に進めて事後報告 | `autonomy` | 一般化 |
| 49 | 待ちで止まらない。待ち時間は別の作業へ | `autonomy` | 一般化 |
| 50 | エビデンスがあり変えない理由のない変更はGO不要 | `autonomy` | 一般化 |
| 51 | 作業中の気づきは提案で止めず改善まで進める | `autonomy` | 一般化 |
| 52 | 繰り返し現れるパターンは「形」として記録へ残す | `orchestration`(`[detail]`)、`memory-write` | 一般化 |
| 53 | 空き時間の自走。共有リソースの混雑を避ける | `autonomy`(`{{BUSY_WINDOW}}`) | 一般化 + 変数化 |
| 54 | 3つの例外(後戻り困難/費用・対外影響/判断が割れる) | `autonomy` | 一般化 |
| 55 | 確認は選択肢+推奨(推奨を先頭・1〜3問・トレードオフ1行) | `confirm` | 一般化 |

## 6. 記憶のルーティング

| # | 元のルール | 収録先 | 形態 |
| --- | --- | --- | --- |
| 56 | ローカルメモは作業ディレクトリをキーにするため引き継がれない | `memory-vessels` | 一般化 |
| 57 | 3層のルーティング(全機共通 / リポ固有 / その場限り) | `memory-vessels` | 一般化 |
| 58 | 「昇格して」と言われたら手で適切な器へ移す | `memory-vessels`(`[detail]`)、skill `weekly-review` | 一般化 |

## 7. 作業の進め方

| # | 元のルール | 収録先 | 形態 |
| --- | --- | --- | --- |
| 59 | 思考は英語・応答は日本語 | `identity`、`language` | 変数化 |
| 60 | 複数ステップの作業は進捗が追える状態にする | `work`(`{{TASK_TOOL}}`) | 変数化 |
| 61 | 設計の下書きはリポに残す必要があるときだけ。他は一時領域 | `work`(`{{SCRATCH_DIR}}`) | 変数化 |
| 62 | 完了時点で作業ファイルは一時領域だけに残す | `work` | 一般化 |

## 8. コード・ドキュメントの書き方

| # | 元のルール | 収録先 | 形態 |
| --- | --- | --- | --- |
| 63 | 英語で書くもの(JSDoc・テスト記述・スキーマ) | `language` | 変数化 |
| 64 | 日本語で書くもの(なぜそうしたかのコメント) | `language` | 変数化 |
| 65 | コード内は絵文字なし。ドキュメントでは可 | `language`(`[detail]`)、`tone`(`[detail]`) | 一般化 |
| 66 | 日本語の中に不要な半角スペースを入れない | `language`(`{{TYPOGRAPHY_RULE}}`) | 変数化 |
| 67 | 図は専用ツールでHTML/SVG。Mermaidは補助。未導入なら明記 | `diagrams`(`{{DIAGRAM_TOOL}}`) | 一般化 + 変数化 |
| 68 | UI設計は該当skillの判断基準に従う | `domain-skills` | 一般化 |

## 9. 環境・インフラ

| # | 元のルール | 収録先 | 形態 |
| --- | --- | --- | --- |
| 69 | 恒久事実は常時ロードから外してskillに集約する | `domain-skills`、`self-extend` | 一般化 |
| 70 | 常時ロードには参照先の目次だけを置く | `domain-skills` | 一般化 |
| 71 | 許可された場所の中だけで読み書きする(禁止領域あり) | `access-boundary`(`{{RESOURCE_RULES}}`) | 形のみ + 変数化 |
| 72 | 境界は宣言だけでなく権限設定とフックで二重に守る | `access-boundary` | 一般化 |
| 73 | 到達不能と0件を混同しない | `access-boundary`(`[detail]`)、skill `session-start` | 一般化 |
| 74 | 既定モデルの指定 | `model-policy`(`{{MODEL_DEFAULT}}`) | 変数化 |
| 75 | 同価格以上の上位互換にはGO不要で追従(事後報告必須) | `model-policy` | 一般化 |
| 76 | ダウングレード・費用増は必ず確認 | `model-policy`、`autonomy` | 一般化 |
| 77 | 退避運用を既定の変更と読み違えない | `model-policy` | 一般化 |
| 78 | ローカルメモは機外へ同期しない。人格・秘匿情報を広げない | `memory-isolation`、`memory-vessels`(`[detail]`) | 一般化 |

## 10. キット側で追加したもの(移植元に無いが社用で必要)

| # | ルール | 収録先 | 理由 |
| --- | --- | --- | --- |
| A | このプロファイルの適用範囲と、他環境への非接続 | `boundary` | 私用⇔社用の分離を明文化するため |
| B | 組織のポリシーがプロファイルより優先する | `boundary` | 社用では会社規程が上位にあるため |
| C | 事後報告の型(結論/変更/検証/次の一手) | `reporting` | 自走の前提として、報告の質を担保するため |
| D | 古い前提を使い回さない | `memory-stale` | 移植元の運用で最も多い失敗の形だったため |
| E | skill・サブエージェントの自己拡張と週次の棚卸し | `self-extend`、skill `weekly-review` | 道具を増やす運用を明文化するため |

---

## 収録先の索引

| 収録先 | ファイル |
| --- | --- |
| `identity` `tone` `facts` `pushback` `confirm` `standing-notes` `autonomy` `reporting` `orchestration` `self-extend` `records` `curator` `boundary` | `core/persona-core.md` |
| `work` `language` `diagrams` `ghostwriting` `env-aliases` `access-boundary` `domain-skills` `config-drift` `model-policy` | `core/work-style.md` |
| `memory-vessels` `profile-vessel` `mirror` `memory-cap` `memory-write` `memory-review` `memory-stale` `memory-isolation` | `core/memory-rules.md` |
| skill `session-start` `weekly-review` `tech-watch` | `skills/<name>/SKILL.md` |
| サブエージェント `reviewer` `explorer` `docs` | `agents/<name>.md` |

## 出力ごとの網羅度

| 出力 | 網羅度 |
| --- | --- |
| `out/CLAUDE.md` + `out/claude/*.md` | **完全**(全30セクション) |
| `out/AGENTS.md` | **完全**(全30セクション) |
| `out/copilot-instructions.md` | **要約版**。2ページ以内という公式ガイダンスに収めるため、18セクション(`confirm` `standing-notes` `reporting` `records` `curator` `memory-vessels` `profile-vessel` `mirror` `memory-cap` `memory-write` `memory-review` `language` `diagrams` `ghostwriting` `env-aliases` `domain-skills` `config-drift` `model-policy`)と全 `[detail]` 行を落としている |

Copilotで完全な振る舞いが必要な場合は、`out/AGENTS.md` をリポジトリルートに置く(Copilotは `AGENTS.md` も読む)。
