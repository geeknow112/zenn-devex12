---
title: "【2026年版】無料で使えるAIコーディングツール比較 - Claude Code, Gemini CLI, Kiro"
emoji: "🤖"
type: "tech"
topics: ["AI", "Claude", "Gemini", "Kiro", "開発ツール"]
published: true
---

## はじめに

2026年、AIコーディングツールは百花繚乱の時代を迎えています。本記事では、実際に試した3つのターミナルベースAIコーディングツールを比較します。

- **Claude Code** - Anthropic製
- **Gemini CLI** - Google製
- **Kiro** - AWS製

結論から言うと、**無料で安定して使いたいならGemini CLI一択**です。

## 各ツールの概要

### Claude Code

Anthropicが提供するターミナルベースのAIコーディングアシスタント。

```bash
npm install -g @anthropic-ai/claude-code
claude
```

**特徴：**
- Claude Opus 4.8 / Sonnet 4.6 などの最新モデルが使える
- ファイル操作、シェルコマンド実行が可能
- エージェント的な動作（自律的にタスクを完了）

**料金：**
- Anthropic APIキーが必要（**有料**）
- 従量課金制

### Gemini CLI

Google公式のターミナルAIツール。**無料枠が充実**しています。

```bash
npm install -g @google/gemini-cli
gemini
```

**特徴：**
- Gemini 3モデル（100万トークンコンテキスト）
- Googleアカウントでログインするだけで使える
- MCP（Model Context Protocol）対応
- Google検索連携

**料金：**
- **無料枠：60リクエスト/分、1,000リクエスト/日**
- APIキー不要（Googleアカウントのみ）

### Kiro

AWSが提供するAI IDE。デスクトップアプリケーション。

**特徴：**
- Claude Opus 4.5搭載
- VS Codeライクなインターフェース
- GitHub連携、MCP対応
- Spec（仕様駆動開発）機能

**料金：**
- 無料で利用可能（AWS Builderアカウント）

## 比較表

| 項目 | Claude Code | Gemini CLI | Kiro |
|------|-------------|------------|------|
| 形態 | CLI | CLI | IDE |
| 無料利用 | ❌ | ✅ | ✅ |
| セットアップ | 難（APIキー必要） | 易（Googleログイン） | 易 |
| モデル | Claude系 | Gemini 3 | Claude Opus 4.5 |
| コンテキスト | 200K | 1M | 200K |
| 日本語 | ○ | ○ | ○ |
| ファイル操作 | ○ | ○ | ○ |
| シェル実行 | ○ | ○ | ○ |
| MCP対応 | ○ | ○ | ○ |

## Claude CodeをGeminiで無料で使う試み（失敗談）

「Claude Codeの操作感でGeminiを使えないか？」と思い、以下を試しました。

### 方法1: 環境変数でGemini APIを指定

```bash
setx ANTHROPIC_API_KEY "your-gemini-api-key"
setx ANTHROPIC_BASE_URL "https://generativelanguage.googleapis.com/v1beta/openai"
setx ANTHROPIC_MODEL "gemini-2.0-flash"
```

**結果：❌ 失敗**

```
There's an issue with the selected model (gemini-2.0-flash). 
It may not exist or you may not have access to it.
```

Gemini APIとAnthropic APIの互換性がなく、モデル指定が通りませんでした。

### 方法2: gemini-for-claude-code プロキシ

[gemini-for-claude-code](https://github.com/coffeegrind123/gemini-for-claude-code)というプロキシサーバーを使う方法。

```bash
git clone https://github.com/coffeegrind123/gemini-for-claude-code.git
cd gemini-for-claude-code
python -m venv .venv
.venv\Scripts\activate
pip install -r requirements.txt

set GEMINI_API_KEY=your-api-key
python server.py
```

別ターミナルで：

```bash
set ANTHROPIC_BASE_URL=http://localhost:8082
claude
```

**結果：❌ 部分的に動くが不安定**

サーバーは起動し、接続もできましたが：

```
API Error: 422 {"detail":[{"type":"literal_error",...}]}
```

Claude Code v2.1.167 とプロキシの互換性が崩れていました。このリポジトリは2026年2月にアーカイブ済みで、最新版Claude Codeには対応していません。

### 結論

**Claude CodeをGeminiで動かすのは現時点では非現実的**です。

公式サポートではないため、バージョンアップのたびに壊れるリスクがあります。

## おすすめの選択

### 無料でターミナルAIを使いたい → Gemini CLI

```bash
npm install -g @google/gemini-cli
gemini
```

Googleアカウントでログインするだけ。1日1,000リクエストまで無料。

### IDEで使いたい → Kiro

[Kiro公式サイト](https://kiro.dev/)からダウンロード。Claude Opus 4.5が無料で使えます。

### Claude Codeの本家を使いたい → APIキー取得

[Anthropic Console](https://console.anthropic.com/)でAPIキーを取得して課金。

## Gemini CLIの使い方

### インストール

```bash
npm install -g @google/gemini-cli
```

### 起動

```bash
gemini
```

初回はGoogleアカウントでログインを求められます。

### 基本操作

```bash
# カレントディレクトリで起動
gemini

# 特定のモデルを指定
gemini -m gemini-2.5-flash

# 非インタラクティブモード（スクリプト向け）
gemini -p "このコードベースのアーキテクチャを説明して"
```

### 便利なコマンド

| コマンド | 説明 |
|----------|------|
| `/help` | ヘルプ表示 |
| `/chat` | 新しい会話を開始 |
| `/model` | モデル切り替え |
| `/bug` | バグ報告 |

## まとめ

| ツール | こんな人におすすめ |
|--------|-------------------|
| **Gemini CLI** | 無料でCLI AIツールを使いたい |
| **Kiro** | IDE統合が欲しい、AWS環境で開発している |
| **Claude Code** | Claudeモデルにこだわりがある、課金OK |

個人的には、**Gemini CLI + Kiro** の組み合わせが最強だと思います。どちらも無料で、用途に応じて使い分けられます。

## 参考リンク

- [Gemini CLI GitHub](https://github.com/google-gemini/gemini-cli)
- [Claude Code](https://www.anthropic.com/claude-code)
- [Kiro](https://kiro.dev/)
- [gemini-for-claude-code（アーカイブ済み）](https://github.com/coffeegrind123/gemini-for-claude-code)
