---
title: "【完全版】GitHub ActionsでX(Twitter)自動投稿を構築する - 設計から実装まで"
emoji: "🐦"
type: "tech"
topics: ["github", "githubactions", "twitter", "x", "automation"]
published: false
---

## はじめに

「ブログ更新したらXにも自動で告知したい」
「定期的にツイートを予約投稿したい」
「でも外部サービスに月額払いたくない」

この記事では、**GitHub + GitHub Actions + X API**だけで自動ツイート投稿システムを構築します。完全無料で、コードはすべてGitHubで管理できます。

### この記事で作るもの

```
┌─────────────────────────────────────────────────────────┐
│  GitHub リポジトリ                                        │
│  ├── tweets/                                            │
│  │   ├── 2026-07-14-new-article.json  ← ツイート予約    │
│  │   └── 2026-07-15-morning.json                        │
│  └── .github/workflows/                                 │
│      └── tweet.yml  ← GitHub Actions                    │
└─────────────────────────────────────────────────────────┘
          │
          │ push or schedule
          ▼
┌─────────────────────────────────────────────────────────┐
│  GitHub Actions                                         │
│  1. tweets/フォルダから投稿対象を取得                    │
│  2. X API v2でツイート投稿                               │
│  3. 投稿済みファイルをposted/に移動                      │
└─────────────────────────────────────────────────────────┘
          │
          ▼
┌─────────────────────────────────────────────────────────┐
│  X (Twitter)                                            │
│  ツイートが投稿される                                    │
└─────────────────────────────────────────────────────────┘
```

### なぜこの構成なのか

| 要件 | 解決策 |
|------|--------|
| 無料で使いたい | GitHub Actions無料枠（月2000分） |
| コードで管理したい | GitHubリポジトリで全履歴が残る |
| 予約投稿したい | cronスケジュール or push時に実行 |
| チーム運用したい | PRでレビュー後にマージ→投稿 |

---

## 1. X Developer Portalでアプリを作成

### 1-1. Developer Portalにアクセス

https://developer.x.com/en/portal/dashboard

Xアカウントでログインし、Developer Portalにアクセスします。

### 1-2. プロジェクトとアプリの作成

1. 「Projects & Apps」→「+ Create Project」
2. プロジェクト名を入力（例：`auto-tweet`）
3. Use caseは「Making a bot」を選択
4. プロジェクト説明を入力
5. アプリ名を入力（例：`auto-tweet-bot`）

### 1-3. 認証情報の取得

「Keys and Tokens」タブで以下を取得：

| 種類 | 用途 |
|------|------|
| API Key | アプリ識別 |
| API Key Secret | アプリ認証 |
| Access Token | ユーザー認証 |
| Access Token Secret | ユーザー認証 |

:::message alert
これらの値は**絶対にコードに直接書かない**でください。GitHub Secretsで管理します。
:::

### 1-4. App permissionsの設定

「User authentication settings」→「Edit」で以下を設定：

- **App permissions**: Read and write（読み書き）
- **Type of App**: Web App, Automated App or Bot
- **Callback URI**: `https://example.com`（使わないがダミーで必要）
- **Website URL**: 自分のサイトかGitHubプロフィール

設定後、Access Tokenを**再生成**してください（権限変更後は再生成が必要）。

---

## 2. GitHub Secretsの設定

### 2-1. リポジトリのSecretsに登録

リポジトリの「Settings」→「Secrets and variables」→「Actions」→「New repository secret」

以下の4つを登録：

| Name | Value |
|------|-------|
| `X_API_KEY` | API Key |
| `X_API_KEY_SECRET` | API Key Secret |
| `X_ACCESS_TOKEN` | Access Token |
| `X_ACCESS_TOKEN_SECRET` | Access Token Secret |

---

## 3. ツイートデータの形式

### 3-1. JSONファイル形式

`tweets/`フォルダにJSONファイルを作成します。

```json:tweets/2026-07-14-new-article.json
{
  "text": "新しい記事を公開しました！\n\nGitHub ActionsでX自動投稿を構築する方法を解説しています。\n\n完全無料で予約投稿もできます。\n\nhttps://zenn.dev/devex12/articles/x-auto-tweet-github-actions",
  "scheduled_at": "2026-07-14T09:00:00+09:00"
}
```

### 3-2. フィールド説明

| フィールド | 必須 | 説明 |
|-----------|------|------|
| `text` | ✅ | ツイート本文（280文字以内） |
| `scheduled_at` | - | 予約日時（ISO 8601形式）。省略時は即時投稿 |
| `media_ids` | - | 添付メディアのID（事前アップロード必要） |
| `reply_to` | - | リプライ先のツイートID |

### 3-3. 複数ツイート（スレッド）

```json:tweets/2026-07-15-thread.json
{
  "thread": [
    {
      "text": "【スレッド】GitHub Actionsの便利な使い方を紹介します。\n\n1/5"
    },
    {
      "text": "① 自動テスト\n\npushするたびにテストが走る。CIの基本。\n\n2/5"
    },
    {
      "text": "② 自動デプロイ\n\nmainにマージしたら本番環境に自動デプロイ。\n\n3/5"
    }
  ],
  "scheduled_at": "2026-07-15T12:00:00+09:00"
}
```

---

## 4. GitHub Actionsワークフロー

### 4-1. ワークフローファイル

```yaml:.github/workflows/tweet.yml
name: Auto Tweet

on:
  push:
    paths:
      - 'tweets/*.json'
  schedule:
    # 毎日9時と18時（JST）に実行
    - cron: '0 0,9 * * *'  # UTC 0:00, 9:00 = JST 9:00, 18:00
  workflow_dispatch:  # 手動実行用

jobs:
  tweet:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          token: ${{ secrets.GITHUB_TOKEN }}

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'

      - name: Install dependencies
        run: npm install twitter-api-v2

      - name: Post tweets
        env:
          X_API_KEY: ${{ secrets.X_API_KEY }}
          X_API_KEY_SECRET: ${{ secrets.X_API_KEY_SECRET }}
          X_ACCESS_TOKEN: ${{ secrets.X_ACCESS_TOKEN }}
          X_ACCESS_TOKEN_SECRET: ${{ secrets.X_ACCESS_TOKEN_SECRET }}
        run: node scripts/tweet.js

      - name: Commit posted files
        run: |
          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"
          git add -A
          git diff --staged --quiet || git commit -m "chore: move posted tweets"
          git push
```

### 4-2. 投稿スクリプト

```javascript:scripts/tweet.js
const { TwitterApi } = require('twitter-api-v2');
const fs = require('fs');
const path = require('path');

// X APIクライアント初期化
const client = new TwitterApi({
  appKey: process.env.X_API_KEY,
  appSecret: process.env.X_API_KEY_SECRET,
  accessToken: process.env.X_ACCESS_TOKEN,
  accessSecret: process.env.X_ACCESS_TOKEN_SECRET,
});

const tweetsDir = './tweets';
const postedDir = './tweets/posted';

async function main() {
  // posted/フォルダがなければ作成
  if (!fs.existsSync(postedDir)) {
    fs.mkdirSync(postedDir, { recursive: true });
  }

  // tweets/フォルダのJSONファイルを取得
  const files = fs.readdirSync(tweetsDir)
    .filter(f => f.endsWith('.json') && !f.startsWith('.'));

  const now = new Date();

  for (const file of files) {
    const filePath = path.join(tweetsDir, file);
    const stat = fs.statSync(filePath);
    
    // ディレクトリはスキップ
    if (stat.isDirectory()) continue;

    const data = JSON.parse(fs.readFileSync(filePath, 'utf-8'));

    // scheduled_atがある場合、時刻チェック
    if (data.scheduled_at) {
      const scheduledTime = new Date(data.scheduled_at);
      if (scheduledTime > now) {
        console.log(`⏰ Skipped (scheduled for later): ${file}`);
        continue;
      }
    }

    try {
      // スレッドの場合
      if (data.thread && Array.isArray(data.thread)) {
        let lastTweetId = null;
        for (const tweet of data.thread) {
          const result = await client.v2.tweet({
            text: tweet.text,
            ...(lastTweetId && { reply: { in_reply_to_tweet_id: lastTweetId } }),
          });
          lastTweetId = result.data.id;
          console.log(`✅ Posted thread part: ${result.data.id}`);
        }
      } else {
        // 単一ツイート
        const result = await client.v2.tweet({ text: data.text });
        console.log(`✅ Posted: ${result.data.id} - ${file}`);
      }

      // 投稿済みファイルを移動
      const postedPath = path.join(postedDir, file);
      fs.renameSync(filePath, postedPath);
      console.log(`📁 Moved to posted/: ${file}`);

    } catch (error) {
      console.error(`❌ Failed: ${file}`, error.message);
      // エラーでも続行（他のツイートを投稿）
    }
  }
}

main().catch(console.error);
```

---

## 5. 実際の運用フロー

### 5-1. 即時投稿

```bash
# 1. ツイートファイルを作成
echo '{"text": "テスト投稿です"}' > tweets/test.json

# 2. コミット & プッシュ
git add tweets/test.json
git commit -m "tweet: テスト投稿"
git push
```

GitHub Actionsが自動実行され、ツイートが投稿されます。

### 5-2. 予約投稿

```bash
# scheduled_atを指定
cat > tweets/2026-07-15-morning.json << 'EOF'
{
  "text": "おはようございます！\n\n今日も1日頑張りましょう。",
  "scheduled_at": "2026-07-15T07:00:00+09:00"
}
EOF

git add tweets/
git commit -m "tweet: 7/15朝の予約投稿"
git push
```

cronスケジュール（毎日9時）で実行され、`scheduled_at`の時刻を過ぎていれば投稿されます。

### 5-3. PRベースの運用（チーム向け）

```bash
# 1. ブランチ作成
git checkout -b tweet/campaign-announce

# 2. ツイート作成
echo '{"text": "キャンペーン開始！..."}' > tweets/campaign.json

# 3. PR作成
git add tweets/
git commit -m "tweet: キャンペーン告知"
git push -u origin tweet/campaign-announce
gh pr create --title "キャンペーン告知ツイート" --body "レビューお願いします"
```

レビュー後にマージすると自動投稿されます。

---

## 6. 応用: biz-toolsへの統合

CLIから簡単にツイート予約できるようにします。

### 6-1. コマンド追加

```go:cmd/tweet.go
package cmd

import (
	"encoding/json"
	"fmt"
	"os"
	"path/filepath"
	"time"

	"github.com/spf13/cobra"
)

type Tweet struct {
	Text        string `json:"text"`
	ScheduledAt string `json:"scheduled_at,omitempty"`
}

var tweetCmd = &cobra.Command{
	Use:   "tweet",
	Short: "X(Twitter)ツイート管理",
}

var tweetAddCmd = &cobra.Command{
	Use:   "add [text]",
	Short: "ツイートを予約追加",
	Args:  cobra.ExactArgs(1),
	RunE: func(cmd *cobra.Command, args []string) error {
		text := args[0]
		scheduledAt, _ := cmd.Flags().GetString("at")

		tweet := Tweet{Text: text}
		if scheduledAt != "" {
			tweet.ScheduledAt = scheduledAt
		}

		// ファイル名生成
		filename := fmt.Sprintf("%s.json", time.Now().Format("2006-01-02-150405"))
		filepath := filepath.Join("tweets", filename)

		// JSON書き出し
		data, _ := json.MarshalIndent(tweet, "", "  ")
		if err := os.MkdirAll("tweets", 0755); err != nil {
			return err
		}
		if err := os.WriteFile(filepath, data, 0644); err != nil {
			return err
		}

		fmt.Printf("✅ Created: %s\n", filepath)
		return nil
	},
}

func init() {
	tweetAddCmd.Flags().StringP("at", "a", "", "予約日時 (例: 2026-07-15T09:00:00+09:00)")
	tweetCmd.AddCommand(tweetAddCmd)
	rootCmd.AddCommand(tweetCmd)
}
```

### 6-2. 使い方

```bash
# 即時投稿用
biz-tools tweet add "新しい記事を公開しました！ https://..."

# 予約投稿
biz-tools tweet add "おはようございます！" --at "2026-07-15T07:00:00+09:00"
```

---

## 7. トラブルシューティング

### 7-1. 403 Forbidden

**原因**: App permissionsが「Read only」のまま

**解決**: Developer Portalで「Read and write」に変更し、Access Tokenを**再生成**

### 7-2. 429 Too Many Requests

**原因**: レート制限に到達

**解決**: 
- 無料プラン: 1日50ツイートまで
- 投稿間隔を空ける（スクリプトにsleep追加）

### 7-3. 重複ツイートエラー

**原因**: 同じ内容を短時間で投稿

**解決**: ツイート内容を少し変える or 時間を空ける

---

## まとめ

GitHub + GitHub Actions + X APIで、完全無料の自動ツイートシステムを構築しました。

**メリット:**
- 月額費用ゼロ
- コードで管理、履歴が残る
- PRでレビューしてから投稿
- cronで定期投稿

**この構成が向いている人:**
- エンジニア（コード管理に慣れている）
- 個人開発者（コスト削減したい）
- チーム運用（レビューフロー入れたい）

ぜひ試してみてください。

---

**関連記事:**
- [GitHub Actionsで自動デプロイを構築する](https://zenn.dev/devex12/articles/...)
- [biz-toolsで業務自動化](https://zenn.dev/devex12/articles/...)
