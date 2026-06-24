---
title: "【実践】Kiro/Claude Codeの「信頼済みコマンド」で調査作業の承認をゼロにする"
emoji: "🔓"
type: "tech"
topics: ["kiro", "claudecode", "ai", "自動化", "開発効率化"]
published: false
---

## この記事の位置づけ

前回の記事では「完全オート」にするためのSteering + Hooks設定を紹介しました。

https://zenn.dev/geeknow112/articles/kiro-full-autopilot-setup

今回はその補足として、**「信頼済みコマンド（Trusted Commands）」設定**を解説します。

これにより、`grep`、`ls`、`aws logs`などの調査系コマンドが承認なしで実行されるようになります。

## 問題：調査フェーズの承認地獄

AIエージェントにコード調査を依頼すると、こうなりがちです。

```
AI: grep -r "auth" src/ を実行してもいいですか？
私: OK
AI: ls -la src/auth/ を実行してもいいですか？
私: OK
AI: cat src/auth/login.ts を実行してもいいですか？
私: OK
AI: aws logs filter-log-events --log-group-name /app/prod を実行してもいいですか？
私: OK
（延々と続く）
```

調査系コマンドは**読み取り専用で破壊的ではありません**。にもかかわらず、毎回承認を求められます。

Steering設定だけでは、この問題は解決しきれません。**コマンド実行の承認はSteering管轄外**だからです。

## 解決策：信頼済みコマンドを設定する

### Kiroの場合

Settings → **Kiro Agent: Trusted Commands** で設定できます。

文字列の前方一致でマッチするため、`grep`と設定すれば`grep -r "auth" src/`も許可されます。

**設定レベル：**
| スコープ | 適用範囲 |
|---------|---------|
| User | 全ワークスペースで有効 |
| Workspace | 特定プロジェクトのみ |

### Claude Codeの場合

設定ファイル `settings.json` に記述します。

**設定ファイルの配置場所：**
| スコープ | ファイル | Git管理 |
|---------|---------|---------|
| グローバル | `~/.claude/settings.json` | No |
| プロジェクト | `.claude/settings.json` | 可 |
| ローカル | `.claude/settings.local.json` | No（gitignore） |

## 設定例：調査系コマンド許可リスト

### Claude Code用 settings.json

```json:~/.claude/settings.json
{
  "permissions": {
    "allow": [
      "Bash(grep:*)",
      "Bash(rg:*)",
      "Bash(ls:*)",
      "Bash(cat:*)",
      "Bash(head:*)",
      "Bash(tail:*)",
      "Bash(find:*)",
      "Bash(wc:*)",
      "Bash(du:*)",
      "Bash(which:*)",
      "Bash(ps:*)",
      "Bash(diff:*)",
      "Bash(sort:*)",
      "Bash(aws * describe-*)",
      "Bash(aws * get-*)",
      "Bash(aws * list-*)",
      "Bash(aws logs:*)",
      "Bash(git status:*)",
      "Bash(git log:*)",
      "Bash(git diff:*)",
      "Bash(git show:*)",
      "Bash(git branch:*)",
      "Bash(gh pr list:*)",
      "Bash(gh pr view:*)",
      "Bash(gh pr diff:*)",
      "Bash(gh run:*)"
    ],
    "deny": [
      "Bash(rm -rf *)",
      "Bash(git push -f *)",
      "Bash(git reset --hard *)",
      "Bash(DROP *)",
      "Bash(DELETE FROM *)"
    ]
  }
}
```

### Kiro用 Trusted Commands

Settings → Kiro Agent: Trusted Commands に以下を1行ずつ追加：

```
grep
rg
ls
cat
head
tail
find
wc
du
which
ps
diff
sort
aws
git status
git log
git diff
git show
git branch
gh pr list
gh pr view
gh pr diff
gh run
```

:::message
Kiroは前方一致なので、`aws`だけ設定すれば全AWSコマンドが許可されます。
読み取り専用に絞りたい場合は`aws`の代わりに`aws ec2 describe`等を個別設定してください。
:::

## 許可するコマンドの選定基準

### 読み取り専用コマンド（推奨）

| カテゴリ | コマンド | 理由 |
|---------|---------|------|
| ファイル検索 | `grep`, `rg`, `find` | 読み取りのみ |
| ファイル表示 | `cat`, `head`, `tail`, `ls` | 読み取りのみ |
| 統計・確認 | `wc`, `du`, `ps`, `which` | システム状態の参照 |
| AWS調査 | `describe-*`, `get-*`, `list-*` | 読み取りAPI |
| Git調査 | `status`, `log`, `diff`, `show` | ローカル参照のみ |
| GitHub CLI | `pr list`, `pr view`, `pr diff` | 読み取りAPI |

### 明示的に拒否すべきコマンド（必須）

```json
"deny": [
  "Bash(rm -rf *)",
  "Bash(git push -f *)",
  "Bash(git reset --hard *)",
  "Bash(DROP *)",
  "Bash(DELETE FROM *)"
]
```

**これらは破壊的な操作なので、明示的に拒否リストに入れておきます。**

## ワイルドカードの使い方

Claude Codeでは `:*` がワイルドカードとして機能します。

| パターン | マッチ例 |
|---------|---------|
| `Bash(grep:*)` | `grep -r "auth" src/` |
| `Bash(aws * describe-*)` | `aws ec2 describe-instances` |
| `Bash(git log:*)` | `git log --oneline -10` |

:::message alert
**危険：広すぎるパターンは避ける**

`Bash(git *)` のようなパターンは、`git push -f` や `git reset --hard` も許可してしまいます。必ず読み取り系のサブコマンドを明示してください。
:::

## 実際の効果

### Before（設定前）

```
私: 認証周りのコードを調査して

AI: grep -r "auth" src/ を実行してもいいですか？
私: OK
AI: ls src/auth/ を実行してもいいですか？
私: OK
AI: cat src/auth/login.ts を実行してもいいですか？
私: OK
AI: aws logs get-log-events ... を実行してもいいですか？
私: OK
... (20回以上の承認)

所要時間：30分（大半は承認待ち）
```

### After（設定後）

```
私: 認証周りのコードを調査して

AI: （自動でgrep, ls, cat, aws logsを実行）
AI: 調査完了しました。レポートです：
    ...

所要時間：5分
```

**承認回数：20回 → 0回**

## SteeringとTrusted Commandsの組み合わせ

| 設定 | 役割 |
|------|------|
| Steering | 「質問するな」「推論で進めろ」という方針を指示 |
| Trusted Commands | 調査系コマンドの実行承認をスキップ |

**両方設定することで、真の「完全オート調査」が実現します。**

```md:investigate-repo.md（Steering）
## 基本方針
- **質問しない**。調査対象を自分で判断する
- **網羅的に調べる**。関連しそうなファイルはすべて確認する
```

```json:settings.json（Trusted Commands）
{
  "permissions": {
    "allow": ["Bash(grep:*)", "Bash(ls:*)", "Bash(cat:*)"]
  }
}
```

## 設定後の確認方法

### Claude Code

```bash
# 現在の設定を確認
claude /permissions
```

### Kiro

Settings → Kiro Agent: Trusted Commands で一覧を確認

## トラブルシューティング

### 「設定したのに承認が出る」場合

**原因1：パターンが合っていない**

```json
// NG：スペースが含まれているとマッチしない場合がある
"Bash(aws logs get-log-events:*)"

// OK：ワイルドカードを柔軟に
"Bash(aws logs:*)"
```

**原因2：設定ファイルの場所が違う**

グローバル設定（`~/.claude/settings.json`）とプロジェクト設定（`.claude/settings.json`）を混同していないか確認してください。

### 「許可しすぎて怖い」場合

最小限から始めて、必要に応じて追加していく方針がおすすめです。

```json
// 最小構成
{
  "permissions": {
    "allow": [
      "Bash(grep:*)",
      "Bash(ls:*)",
      "Bash(cat:*)"
    ]
  }
}
```

## まとめ

| 設定 | 効果 |
|------|------|
| 調査系コマンドを許可 | 承認ゼロで調査が進む |
| 破壊的コマンドを拒否 | 安全性を担保 |
| Steeringと組み合わせ | 真の完全オート調査 |

調査タスクの効率が**6倍**になります（30分→5分）。ぜひ設定してみてください。

## 関連記事

https://zenn.dev/geeknow112/articles/kiro-full-autopilot-setup
https://zenn.dev/geeknow112/articles/kiro-steering-real-workflow
