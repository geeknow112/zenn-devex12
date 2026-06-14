---
title: "Go + Cobraで業務自動化CLIツールを作る"
emoji: "🐍"
type: "tech"
topics: ["go", "cobra", "cli", "automation"]
published: false
---

## はじめに

業務で繰り返し行う作業を自動化したいと思ったことはありませんか？

今回は Go と Cobra を使って、記事投稿を自動化する CLI ツールを作成した過程を紹介します。

## なぜ Go + Cobra なのか

### Go を選んだ理由

- **シングルバイナリ**: ビルドすると1つの実行ファイルになる
- **クロスプラットフォーム**: Windows/Mac/Linux 向けに簡単にビルド可能
- **高速**: コンパイル言語なので実行が速い

### Cobra を選んだ理由

Cobra は Go 製の CLI フレームワークで、以下のツールで採用されています：

- kubectl（Kubernetes CLI）
- hugo（静的サイトジェネレーター）
- gh（GitHub CLI）

サブコマンド構造を簡単に作れるのが魅力です。

## プロジェクト構成

```
biz-tools/
├── main.go          # エントリーポイント
├── go.mod           # 依存関係
├── config.yaml      # 設定ファイル（gitignore）
├── cmd/
│   ├── root.go      # ルートコマンド
│   └── media.go     # mediaサブコマンド
└── README.md
```

## 実装

### 1. プロジェクト初期化

```bash
mkdir biz-tools && cd biz-tools
go mod init github.com/yourname/biz-tools
go get github.com/spf13/cobra
```

### 2. エントリーポイント（main.go）

```go
package main

import "github.com/yourname/biz-tools/cmd"

func main() {
    cmd.Execute()
}
```

### 3. ルートコマンド（cmd/root.go）

```go
package cmd

import (
    "fmt"
    "os"
    "github.com/spf13/cobra"
)

var rootCmd = &cobra.Command{
    Use:   "biz-tools",
    Short: "Business automation CLI tool",
    Long:  `biz-tools is a CLI tool for automating business workflows.`,
}

func Execute() {
    if err := rootCmd.Execute(); err != nil {
        fmt.Fprintln(os.Stderr, err)
        os.Exit(1)
    }
}
```

### 4. サブコマンド（cmd/media.go）

```go
package cmd

import (
    "fmt"
    "github.com/spf13/cobra"
)

var mediaCmd = &cobra.Command{
    Use:   "media",
    Short: "Media publishing commands",
}

var mediaDraftCmd = &cobra.Command{
    Use:   "draft [file]",
    Short: "Create a draft and PR on GitHub",
    Args:  cobra.MinimumNArgs(1),
    RunE: func(cmd *cobra.Command, args []string) error {
        platform, _ := cmd.Flags().GetString("platform")
        file := args[0]
        fmt.Printf("Creating draft for %s on %s\n", file, platform)
        // 実際の処理を実装
        return nil
    },
}

func init() {
    rootCmd.AddCommand(mediaCmd)
    mediaCmd.AddCommand(mediaDraftCmd)
    mediaDraftCmd.Flags().StringP("platform", "p", "zenn", "Target platform")
}
```

### 5. ビルドと実行

```bash
# ビルド
go build -o biz-tools

# 実行
./biz-tools media draft article.md -p zenn
```

## 設定ファイルで柔軟に

プラットフォームごとのリポジトリパスを `config.yaml` で管理：

```yaml
platforms:
  zenn:
    repo: /path/to/zenn-repo
  qiita:
    repo: /path/to/qiita-repo
```

これを読み込んで、指定されたリポジトリに自動でPRを作成します。

## まとめ

Go + Cobra を使えば、構造化された CLI ツールを簡単に作成できます。

今回作成したツールでは：
1. 記事ファイルを指定
2. 対象リポジトリにブランチ作成
3. コミット & プッシュ
4. GitHub PR 自動作成

という流れを1コマンドで実行できるようになりました。

業務自動化の第一歩として、ぜひ試してみてください。

## 参考

- [Cobra GitHub](https://github.com/spf13/cobra)
- [Go 公式ドキュメント](https://go.dev/doc/)
