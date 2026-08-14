---
title: "「隔離環境の構築はこれから」の5日後、実際に選んだのはEC2ではなくRoutineだった"
emoji: "🌙"
type: "tech"
topics: ["claudecode", "claude", "ai", "automation", "cron"]
published: false
---

## この記事で得られること

- 「夜間バッチを完全無人で回すには `--dangerously-skip-permissions` ＋ 隔離環境（EC2+tmux）が必要そう」と結論づけた前回の続き。5日後に実際に導入したのは、EC2ではなく Claude Code Remote の **Routine（スケジュールトリガー）** だったという方針転換の記録
- Routine の実体（`allowed_tools` / `sources` / `cron_expression`）を実際の設定値で確認し、「`git push` と `PR作成` はさせるが、`mainへの直接コミット` と `PRの自動マージ` は絶対にさせない」というガードレールが、ツール権限ではなく**プロンプト内の明示的な禁止指示**だけで担保されていた、という設計上の事実
- cronはUTC入力必須で、JSTで「毎晩深夜2時」のつもりが日付をまたいでズレる罠
- 外部APIを直叩きする設計（Zennのトレンド取得）が、実行環境のネットワークポリシーで丸ごとブロックされた実録（エラーメッセージ付き）

## 背景：前回、隔離環境の話は「保留」にしていた

以前、Zenn/Qiitaのトレンドを調べて記事を書き、コミット・pushまでを夜間に無人で回そうとしたら、承認リスト（`permissions.allow`）に登録していない `git push` や `gh pr create` の手前で毎回止まる、という事象を検証しました。結論として、許可リスト方式（コマンドの事前ホワイトリスト化）と、`--dangerously-skip-permissions` / `--permission-mode bypassPermissions`（チェックそのものを丸ごとスキップ）は別レイヤーの話であり、後者は公式ヘルプに **"Recommended only for sandboxes with no internet access."** と明記されている以上、普段使いのマシンでそのまま使うのは危険だと判断しました。

そこで代替案として挙げていたのが、「使い捨てできる隔離環境（ネットワークやIAM権限を絞ったEC2インスタンスなど）を用意し、その中だけで `bypassPermissions` を使う」という構成でした。ただし記事の最後には「この構成はまだ実装していません」と書いて宿題にしていました。

5日後の今、実際に導入したのは EC2+tmux ではありませんでした。

## 実際に選んだもの：Claude Code Remote の Routine

Claude Code には、プロンプト・対象リポジトリ・connector（Gmail/Slack/Notion等）をひとまとめにして、スケジュールやAPIコール、GitHubイベントをトリガーに自動実行する **Routine** という仕組みがあります。公式ドキュメントには次のように説明されています。

> Routines execute on Anthropic-managed cloud infrastructure, so they keep working when your laptop is closed.

つまり「隔離環境をどう自前で作るか」という前回の問い自体が、Routineを使う前提では発生しません。管理された分離実行環境がプラットフォーム側の機能として最初から用意されているためです。EC2+tmuxの構想は、自分でスケジューラ・ネットワーク境界・プロセス生存監視まで面倒を見る前提でしたが、Routineはそのレイヤーを丸ごと肩代わりします。

今回組んだ記事下書き生成用Routineの実際の設定は次の通りでした。

- `cron_expression`: `0 17 * * *`（UTC 17:00 = JST 翌2:05稼働。実際の初回起動もJST換算で日付をまたいでいた）
- `job_config.ccr.session_context.allowed_tools`: 11個（`Bash` / `Read` / `Write` / `Edit` / `Glob` / `Grep` / `WebFetch` / Gmail検索系2つ / Slackチャンネル検索・送信2つ）
- `job_config.ccr.session_context.sources`: 対象2リポジトリの `git_repository` URL

```mermaid
flowchart TD
    A[cron: 0 17 * * * UTC] --> B[Anthropic管理の分離コンテナでセッション起動]
    B --> C[sourcesで指定したリポジトリをclone]
    C --> D[Gmail検索でQiita/Zennニュースレターを取得]
    D --> E[トレンド分析・自分の実体験との交点を探索]
    E --> F{良い切り口があるか}
    F -->|Yes| G[ブランチ作成 → 下書き.md作成]
    F -->|No| H[何もせずSlackに理由を報告]
    G --> I[git commit → git push → gh pr create]
    I --> J[Slackに結果を投稿]
    I -.->|禁止: プロンプトの明示的禁止のみで担保| K((mainへの直接コミット/自動マージ))
```

## つまずき①：`allowed_tools` に `Bash` があると、境界線はツールではなくプロンプトにしか引けない

`allowed_tools` の中身を実際に見て気づいたのは、`Bash` が許可されている以上、`git push` はもちろん `gh pr create` によるPR作成も理論上そのまま実行できてしまうという点でした。当初は「push はさせるがPR作成は防ぎたい」という要件で設計していましたが、`Bash` にリポジトリ操作を任せる構成である以上、pushとPR作成をツールの許可/不許可という粒度で分離することはできません（どちらも同じシェルコマンド経由で実行可能なため）。

結局、要件自体を「push・PR作成はさせるが、mainへの直接コミットとPRの自動マージは絶対にさせない」という形に見直しました。この境界線も、ツール側で物理的に塞げるものではなく、Routineのプロンプト本文に次の一文を明記することでしか担保できません。

- 「mainブランチへの直接コミット・pushは絶対禁止。変更は必ずPR経由」
- 「PRは作成するが、自分でマージは絶対にしない」

つまり今回のガードレールは、`Bash` を丸ごと許可した時点で、原理的には `git push origin main` やPRの自動マージすら実行可能な状態にあり、それを止めているのは最終的にはモデルが指示に従うことへの信頼だけ、というものでした。前回の記事で書いた「許可リスト方式とフルバイパスは別レイヤー」という話の延長線上にある、もう一段細かい話でした。

## つまずき②：cronはUTC必須。JSTの「深夜」は日付をまたぐ

Routine作成時のcron指定はUTC評価です。「日本時間で深夜2時に動かしたい」つもりで `0 17 * * *`（UTC 17:00）と指定しましたが、これはJSTでは**日付が変わった直後の翌1〜2時台**になります。実際、`created_at` はUTC 16:02（JST換算だと2026-08-14 01:02）で、体感の「2026-08-13の夜」とはズレていました。日次バッチをJSTの特定時刻で動かしたい場合、UTC変換時に日付フィールドまでズレていないか確認する必要があります。

## つまずき③：外部APIの直叩きは、実行環境のネットワークポリシー次第で丸ごと失敗する

トレンド取得のフローでは「Zennからのニュースレターが0件なら `https://zenn.dev/api/articles?order=daily&count=30` を直接取得する」という設計にしていました。実際に今回Zennのニュースレターは0件だったため、このAPIを叩きに行きましたが、結果は次の通りでした。

```
EGRESS_BLOCKED: Access to zenn.dev is blocked by the network egress proxy.
```

`curl` で直接叩いても同様に、プロキシのCONNECTトンネル自体が403で拒否されました。Qiitaはメールニュースレター経由のためこの制約を受けませんでしたが、Zennだけは実行環境のネットワークポリシー次第で情報源が丸ごと消える、という非対称なリスクが可視化されました。ニュースレターという一次経路と、API直叩きという二次経路の両方を用意していたにもかかわらず、二次経路側が環境要因で機能しない、という組み合わせは設計時には想定していませんでした。

## 得られた知見・まとめ

- 夜間バッチの無人化に、EC2+tmuxのような自前の隔離環境は必須ではなかった。Claude Code Remoteの管理された分離実行環境（Routine）がその役割を代替した
- ただし「危険な操作をさせない」設計は、`Bash` のような強い権限を渡した時点でツール側では境界を引けなくなる。今回の構成は、pushとPR作成は許容しつつmainへの直接コミットと自動マージだけをプロンプトの明示的な禁止文言で止める、という指示への信頼に強く依存したものになっている
- cronはUTC基準。JSTでの意図した時刻とズレていないか、作成直後に`created_at`/`next_run_at`で必ず確認する
- 外部サイトのAPIを直叩きする設計は、実行環境のネットワークポリシーに依存する。ニュースレターやRSSなど、経路をなるべく分散させておくと安全

## よくある疑問

**Q. `bypassPermissions` を使わずに、`Bash` を丸ごと許可して本当に安全なのか？**
A. 「壊れても被害範囲が閉じている」ことが前提のEC2案と違い、今回は「対象リポジトリ2つへのgit操作以外に実害のある操作をさせない」ことを、connectorのスコープ（GmailはOAuth経由の読み取り検索のみ、SlackはPublicチャンネル検索・送信のみ）とプロンプトの明示的な禁止で担保しています。完全な技術的強制ではなく、リスクを許容範囲まで下げる設計だという理解が正確です。

**Q. Routineは複数同時に動かせるのか？**
A. 動かせます。今回の環境では、この記事執筆用のRoutineとは別に、用途の異なる日次・時次のRoutineが同一環境内で並行稼働していました。それぞれ許可ツールとconnectorのスコープを別々に絞っており、1つのRoutineの権限が別のRoutineに波及しない設計になっていました。

**Q. 前回のEC2+tmux案は無駄だったのか？**
A. 無駄ではなく、今回の結論を出す前提知識になりました。「フルバイパスは隔離環境とセットが前提」という理解があったからこそ、Routineが提供する分離実行環境がその代替になり得ると判断できました。

## 参考リンク

- [Automate work with routines - Claude Code Docs](https://code.claude.com/docs/en/routines)
- [Run prompts on a schedule - Claude Code Docs](https://code.claude.com/docs/en/scheduled-tasks)
- [Claude Code on the web - Overview](https://code.claude.com/docs/en/claude-code-on-the-web)

※本記事は個人のClaude Code Remote環境で確認した内容です。Routineの挙動は執筆時点でリサーチプレビュー段階であり、将来変わる可能性があります。
