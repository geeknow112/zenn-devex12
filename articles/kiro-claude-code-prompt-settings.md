---
title: "Kiro/Claude Codeを賢く動かすプロンプト設定術"
emoji: "🧠"
type: "tech"
topics: ["kiro", "claudecode", "ai", "プロンプト", "vscode"]
published: false
---

## はじめに

AIコーディングアシスタント「Kiro」や「Claude Code」を使っていて、こんな経験はありませんか？

- 毎回同じことを説明している
- プロジェクトのルールを守ってくれない
- 余計な質問が多くて作業が進まない

これらは**設定ファイルとプロンプト**で解決できます。

本記事では、Kiro/Claude Codeを「賢いアシスタント」にするための設定術を紹介します。

## 設定ファイルの種類

### Kiroの場合

```
.kiro/
├── steering/          # AIへの指示ファイル
│   ├── user-preferences.md
│   └── media-update.md
└── settings/
    └── mcp.json       # MCP設定
```

### Claude Codeの場合

```
.claude/
├── settings.json      # プロジェクト設定
└── CLAUDE.md          # プロジェクト固有の指示
```

## Steering（Kiro）の書き方

Steeringは、AIに対する**永続的な指示**を書くファイルです。

### 基本形式

```markdown
---
inclusion: auto   # 常に読み込む
---

# プロジェクトルール

- コミットメッセージは日本語で書く
- テストは必ず書く
```

### inclusion の種類

| 値 | 動作 |
|---|---|
| `auto` | 常に読み込む |
| `manual` | `#ファイル名` で明示的に呼び出し |
| `fileMatch` | 特定ファイルを開いたときのみ |

### 実用例：ユーザー設定

```markdown
---
inclusion: auto
---

# ユーザー設定

## コミュニケーション
- 敬語を使用する
- 簡潔に回答する
- 質問せず推論して進める

## 作業ルール
- メモファイルへの追記は確認してから行う
```

### 実用例：作業別の指示（manual）

```markdown
---
inclusion: manual
---

# メディア更新作業

「メディア更新」と言われたら以下の手順で作業開始。

## 対象リポジトリ
- Zenn: zenn-devex12
- Qiita: qiita-devex12

## 作業の流れ
1. memo/ フォルダで前回の進捗を確認
2. 質問せず、メモの情報で判断して進める
```

チャットで `#media-update` と入力すると、この指示が読み込まれます。

## CLAUDE.md（Claude Code）の書き方

Claude Codeでは、プロジェクトルートに `CLAUDE.md` を置きます。

```markdown
# プロジェクト概要

このリポジトリはXXXのためのYYYです。

## 技術スタック
- 言語: TypeScript
- フレームワーク: Next.js
- DB: PostgreSQL

## コーディングルール
- 関数は50行以内
- any禁止
- テストは__tests__/に配置

## よく使うコマンド
- `npm run dev` - 開発サーバー起動
- `npm run test` - テスト実行
- `npm run lint` - Lint実行
```

## プロンプトのコツ

### 1. 「質問するな」を明記する

```markdown
- 不明点があっても質問せず、推論して進める
- 判断に迷ったら、より安全な選択肢を取る
```

AIは丁寧に質問してきますが、作業効率が落ちます。

### 2. 具体的なファイルパスを書く

❌ 悪い例
```markdown
設定ファイルを確認してから作業する
```

✅ 良い例
```markdown
作業前に `memo/memo.YYYYMMDD.md` を確認する
リポジトリは `c:\Users\xxx\git_repo\` 配下
```

### 3. 作業フローを明記する

```markdown
## Zenn記事投稿フロー
1. ブランチ作成: `git checkout -b article/xxx`
2. 記事作成: `articles/xxx.md`
3. コミット → push → PR作成
4. マージ後、自動デプロイ
```

手順が明確だと、AIが迷わず実行できます。

## MCP設定（高度な連携）

MCPを使うと、外部ツールとの連携が可能になります。

```json
{
  "mcpServers": {
    "github": {
      "command": "gh",
      "args": ["mcp", "server"],
      "disabled": false
    }
  }
}
```

GitHub CLIと連携すれば、PR作成やIssue操作をAIが直接実行できます。

## 設定ファイルの管理

### .gitignoreに入れるもの

```gitignore
# 個人設定（APIキーなど）
.kiro/settings/mcp.json
.claude/settings.json
```

### リポジトリにコミットするもの

```
# チーム共通のルール
.kiro/steering/coding-rules.md
CLAUDE.md
```

## まとめ

| 設定 | 用途 |
|---|---|
| steering (auto) | 常に適用するルール |
| steering (manual) | 作業別の指示 |
| CLAUDE.md | プロジェクト概要・ルール |
| MCP | 外部ツール連携 |

これらを整備しておくと、AIが「毎回説明しなくても動ける」アシスタントになります。

**一度設定すれば、あとは楽。**

ぜひ試してみてください。
