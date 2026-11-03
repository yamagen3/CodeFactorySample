# コードファクトリー設計 v0

状態: 設計の合意まで完了。Grok Bot のクラウド環境での動作は未検証(2026-11-03 時点)。

## 1. 構成

```text
あなた
  | 依頼 / 仕様の承認 / 停止時の判断
  v
Grok Bot (クラウド環境)   全体管理、受入れテストの承認、第二ゲート、マージ
  | 工程ごとに DSH を起動
  v
DSH + factory プラグイン  指示役(モデルなし)、子プロセスの管理、進捗の記録
  ├ Agent SDK → CC        仕様、受入れテスト、第一ゲート
  └ ACP → OpenCode (隔離)  実装

agmsg             進捗の連絡と詰まりの検知(参考情報)
GitHub 公開リポジトリ  成果物と判定の正本
```

## 2. 決定事項

| # | 論点 | 決定 |
|---|---|---|
| Q1, Q8 | 実行場所 | すべて Grok Bot のクラウド環境。このPCの DSH 設定は流用しない |
| Q2, Q14 | 最初の対象 | 新規の CSV→Markdown 表 CLI |
| Q3 | 第二ゲート | Grok。通れば自動マージ |
| Q4 | 差し戻し | 全ゲート合格のときだけマージ。実装は最大 3 周で、超えたら停止して通知 |
| Q5 | 権限 | OpenCode を隔離し、持たせるのは自分のモデル用キーだけ |
| Q6 | 進捗管理 | agmsg を使う。デッドロックを防ぐ程度の参考情報で、判定の根拠にしない |
| Q7 | 起動の流れ | Grok が全体を管理し、DSH が Agent SDK 経由で CC を、ACP 経由で OpenCode を抱える。DSH のモデルは使わず、factory プラグインが指示を出す(11 章) |
| Q9 | Git リモート | GitHub Free の公開リポジトリ |
| Q10 | 仕様の承認 | あなたが承認する |
| Q11 | CC の認証 | サブスクのトークン(`CLAUDE_CODE_OAUTH_TOKEN`) |
| Q12 | 実行単位 | 工程ごとに DSH を 1 回起動 |
| Q13 | 第一ゲート | CC が受入れテストを先に書き、Grok の承認で固定。実行は GitHub Actions、通ったら CC が差分を仕様と照合 |

## 3. 流れ

1. あなたが Grok Bot に依頼する。
2. Grok が DSH を起動し、DSH が CC を動かす。CC は `factory/<id>` ブランチに仕様と受入れテストを作り、未解決の質問を返す。
3. Grok が質問をあなたに渡す。あなたが回答して仕様を承認する。
4. Grok が受入れテストと仕様の対応を確認し、`spec/<id>` タグを打って固定する。
5. 周回を始める。
   1. Grok が DSH を起動し、DSH が OpenCode に実装させる。
   2. Grok が成果をパッチとして取り出し、クリーンな clone に適用して push し、PR を作る。
   3. GitHub Actions が `guard` と `ci` を実行する。
   4. Grok が DSH を起動し、DSH が CC を動かす。CC は差分を仕様と照合し、`gate/claude` を記録する。
   5. Grok が独立にレビューし、`gate/grok` を記録する。
6. 全チェックが成功なら Grok がマージする。不合格なら指摘を添えて 5 に戻る。3 周で合格しなければ停止して通知する。

CC は毎回新しいセッションで、あなたに質問できない(DSH が `AskUserQuestion` を無効にする)。工程間の引き継ぎはすべて Git 上のファイルで行う。

## 4. main の必須チェック

| チェック | 記録者 | 内容 |
|---|---|---|
| `guard` | GitHub Actions(`pull_request_target`) | `spec/`、`tests/acceptance/`、`.github/` が `spec/<id>` タグから変わっていたら不合格 |
| `ci` | GitHub Actions | 固定された受入れテストと通常のテストを実行 |
| `gate/claude` | CC | 差分と仕様の照合 |
| `gate/grok` | Grok | 独立レビュー |

- チェックはコミットごとに記録するので、新しいコミットでは取り直しになる。
- 管理者も迂回できない設定にする。保護の設定はあなたがクラウド外で行い、管理権限のトークンはクラウドに置かない。
- `guard` は PR 内のコードを実行しない。比較対象のワークフローは `main` 側の定義を使うので、PR で書き換えられない。

## 5. 認証情報

| 持ち主 | 持つもの | 持たないもの |
|---|---|---|
| OpenCode | 専用のモデルキー | Git トークン、CC と Grok の認証情報 |
| CC | `CLAUDE_CODE_OAUTH_TOKEN`、`statuses` のみの GitHub トークン | push やマージの権限 |
| DSH | 子プロセスに渡す値だけ | 自分用のモデルキー、GitHub の管理権限 |
| Grok | 対象リポジトリ限定の GitHub トークン(contents、pull requests、statuses) | Administration 権限 |

- モデルキーは専用にして上限額を設定する。リポジトリには入れない。
- DSH は子プロセスの環境から認証情報を除くので、CC と OpenCode の `env` に必要なものだけ明示する。
- CC の環境に `ANTHROPIC_API_KEY` を入れない(トークンより優先される)。
- OpenCode の作業領域には push 権限のある `.git` を置かない。`.git/hooks` 経由で進行管理側の権限を使われるのを防ぐため。

## 6. 公開リポジトリの運用

- ファクトリーへの入力は、あなたが Grok Bot に渡した依頼だけ。他人の Issue や PR コメントはエージェントに読ませない。
- フォークからの PR は処理しない。Issue は無効にする。
- シークレットスキャンとプッシュ保護を有効にする。
- 仕様、受入れレポート、ログはすべて公開される前提で書く。

## 7. 既定値(未合意。変更可)

| 項目 | 既定値 |
|---|---|
| 停止の判定 | 工程開始から 30 分で factory プラグインが子プロセスを止める。DSH には実行時間の上限が見当たらないため。Grok は 1 回だけやり直し、再度止まれば通知 |
| 周回数の記録 | PR のラベル(`round-1` など)。Grok が更新する |
| agmsg に書く役 | Grok と factory プラグイン(工程の開始と終了)。CC と OpenCode には書かせない |
| 通知先 | Grok Bot のチャット |
| 題材の言語 | Python 3.12、標準ライブラリのみ |

## 8. 未決定

- **factory プラグインの置き場所:** 題材と同じリポジトリなら `guard` の対象に加え、分けるなら GitHub トークンの範囲も分ける。OpenCode に書き換えさせないことが条件。
- **隔離の方式:** コンテナか別ユーザーか。環境調査の結果で決める。
- **権限の許可リスト:** CC の `dontAsk` と、DSH の OpenCode 向け `permission: reject` のままでは、必要な操作も拒否される。工程ごとに許可する範囲を検証で決める。

## 9. 受け入れたリスク

- **Grok への権限集中:** 全体管理、受入れテストの承認、第二ゲート、マージを Grok が担う。Grok は技術的には `gate/claude` も書ける。
- **サブスクのトークン:** 日常利用と枠を共有し、外部クラウドに置く。規約や課金の扱いは変更される可能性がある。枠の上限到達は停止として扱う。
- **公開リポジトリ:** 成果物とログがすべて公開される。
- **DSH の内部 API:** factory プラグインは 0.2.1-rc.5 の内部 API に依存する。バージョンを固定し、壊れたら Agent SDK の直接呼び出しに切り替える。
- **agmsg:** 送信者を検証しないので、判定には使わない。
- **Grok Bot の情報:** クラウド環境の仕様は第三者の記事に基づき、xAI 公式では未確認。

## 10. 検証計画

P0 から順に進める。P0〜P3 のどれかが通らなければ、Q8 の実行場所に戻って決め直す。

| 段階 | 確認すること | 合格条件 |
|---|---|---|
| P0 | クラウド環境の調査(付録 A) | 必要なツール、通信先、永続化、隔離の手段が揃う |
| P1 | DSH、factory プラグイン、OpenCode、agmsg の導入 | モデルのキーなしで DSH とプラグインが起動し、`agmsg` で送受信できる |
| P2 | DSH に抱えられた CC | `env` で渡した `CLAUDE_CODE_OAUTH_TOKEN` で認証され、短い応答が返る。bare モードでは読まれないため、実際の呼び出しで確認する |
| P3 | DSH に抱えられた OpenCode の隔離 | ACP 経由で実装できる。かつ環境変数にもファイルにも Git トークンと CC トークンが見えない |
| P4 | GitHub の保護 | チェック未成功の PR は Grok のトークンでマージできない |
| P5 | 全体通し | 下の 4 シナリオがすべて期待どおり |

### P5 のシナリオ(ファクトリー v0 の完了条件)

1. **正常系:** CSV→Markdown 表 CLI が依頼から `main` まで通る。
2. **テスト改ざん:** 実装側で受入れテストを変更すると `guard` が失敗し、マージできない。
3. **3 周停止:** 通らない受入れテストを与えると 3 周で停止し、通知が届く。
4. **停止検知:** 実行中の工程を止めると Grok が検知し、1 回やり直してから通知する。

## 11. モデルなしで DSH に CC を抱えさせる

`dsh headless` や Web 画面などの既製の入口は、DSH のモデルが指示役になる。キーがなければ `MISSING_CREDENTIAL` で終了する。そこで指示役を factory プラグインに置き換える。

```js
// headless-runner を無効にし、CC の部品とこのプラグインを --patch で読み込む
const { agent } = await ctx.agents.create({ sessionId, meta: { cwd }, agentOptions }); // 形だけの親
const run = await ctx.subagents.startActivation({
  provider: 'claude-code', label, delivery: 'caller', signal,
  request: { cwd, prompt: [{ type: 'text', text: task }], parent: agent },
});
const result = await run.result; // stopReason と CC の出力
await run.dispose();
```

- **確認済み(2026-11-03、このPC):** モデルのキーなしで CC が完了し、出力をそのまま受け取れた。DSH は 0.2.1-rc.5、CC はログイン済みの環境で動かした。
- **未確認:** クラウド環境、`CLAUDE_CODE_OAUTH_TOKEN`、OpenCode を同じ方法で呼ぶこと。それぞれ P1〜P3 で確認する。
- **実装時の注意:**
  - `agentOptions` にはモデルの設定値を渡す。テストでは `deepseek-official/deepseek-flash` を指定した。キーなしでも動き、このモデルは呼ばれない。
  - CC の部品はパッチからパッケージ名で解決できなかったので、ファイルのパスで指定した。
  - 既製の処理を止めるので、終了コードはプラグインが `stopReason` から決める。
  - 子の思考やツール操作は親に渡らないので、途中経過は分からない。進捗は工程の開始と終了だけ記録する。

## 付録 A: Grok Bot に渡す環境調査プロンプト

```text
コードファクトリーを動かせるか調べたいので、このPCではなく、あなたのクラウド環境を調査してください。
調査だけを行い、インストール、設定変更、ファイル作成はしないでください。
環境変数や認証情報の値は出力しないでください。
分からない項目は推測せず「不明」と書き、結果は項目番号つきの表で返してください。

1. OS、アーキテクチャ、CPU、メモリ、ディスクの空き容量
2. node、npm、git、python3、bash、sqlite3、docker、podman、tmux、sudo の有無とバージョン
3. 現在のユーザー、sudo の可否、別ユーザーを作成できる権限の有無(作成はしない)
4. コンテナを起動できそうか(起動はせず、ランタイムとソケットの有無だけ確認)
5. ファイルはセッションをまたいで残るか、再起動で消える場所はどこか
6. バックグラウンドプロセスを動かし続けられる時間と、その制限
7. 定期実行の仕組みの有無と最短間隔
8. github.com、api.anthropic.com、api.openai.com、registry.npmjs.org に HTTPS で届くか
9. 認証情報を安全に保管する仕組みの有無と、プロセスへの渡し方
10. 1 タスクの最大実行時間
```

## 参考

- [DeepSeek Harness](https://github.com/deepseek-ai/deepseek-harness)
- [fujibee/agmsg](https://github.com/fujibee/agmsg)
- [About protected branches - GitHub Docs](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-protected-branches/about-protected-branches)
- [Authentication - Claude Code Docs](https://code.claude.com/docs/en/authentication)
- [Use the Claude Agent SDK with your Claude plan](https://support.claude.com/en/articles/15036540-use-the-claude-agent-sdk-with-your-claude-plan)
- [Cua Docs: Grok Bot](https://cua.ai/docs/use-cua-with/grok-bot)(第三者情報)
- [Vellum: Official Grok Bot Breakdown](https://www.vellum.ai/blog/official-grok-bot-breakdown)(第三者情報)
