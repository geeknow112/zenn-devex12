---
title: "【Go】Webサイトのセキュリティ簡易診断ツールを作る - SSL・ヘッダー・CMS検出"
emoji: "🔍"
type: "tech"
topics: ["go", "security", "cli", "ssl", "web"]
published: false
---

## この記事で得られること

- Go で **Webサイトのセキュリティ簡易診断ツール** を作る方法
- SSL証明書、HTTPヘッダー、CMS検出などの **公開情報のみ** でのチェック手法
- Cobra を使った **CLIツール設計**
- text / markdown / json / html の **複数フォーマット出力**

## 背景：なぜセキュリティ診断ツールが必要か

Webサイトを運営していると、以下のような問題を見落としがちです：

- SSL証明書の期限切れ
- セキュリティヘッダーの未設定
- CMSのバージョン露出
- サーバー情報の露出

これらは **公開情報のみ** で確認できます。

## 完成形

```bash
# 基本的な使い方
$ biz-tools scan https://example.com

# Markdown形式で出力
$ biz-tools scan https://example.com -o markdown

# HTMLレポートとしてファイル保存
$ biz-tools scan https://example.com -o html -f report.html
```

出力例：

```
=== セキュリティ診断結果 ===
対象URL: https://example.com
診断日時: 2026-07-03 22:25:02
総合リスク: Medium (スコア: 4)

--- SSL/TLS ---
  有効: はい
  有効期限: 2026-10-25 (残り114日)
  発行者: Let's Encrypt
  プロトコル: TLS 1.3

--- 検出された問題 ---
1. [Medium] HSTSヘッダーなし
   → HSTSヘッダーを追加してHTTPS接続を強制してください
```

## 実装

### プロジェクト構成

```
biz-tools/
├── main.go
├── cmd/
│   ├── root.go
│   └── scan.go    ← 今回作るファイル
├── go.mod
└── go.sum
```

### 1. データ構造の定義

まず、診断結果を格納する構造体を定義します。

```go
package cmd

import (
	"crypto/tls"
	"encoding/json"
	"fmt"
	"io"
	"net"
	"net/http"
	"os"
	"regexp"
	"strings"
	"time"

	"github.com/spf13/cobra"
)

// ScanResult holds all scan results
type ScanResult struct {
	URL           string
	ScanTime      time.Time
	SSL           SSLResult
	Headers       HeadersResult
	CMS           CMSResult
	ServerInfo    ServerInfoResult
	DNS           DNSResult
	OverallRisk   string
	RiskScore     int
	Findings      []Finding
}

type SSLResult struct {
	Enabled       bool
	ValidFrom     time.Time
	ValidUntil    time.Time
	DaysRemaining int
	Issuer        string
	Protocol      string
	Risk          string
}

type HeadersResult struct {
	HSTS              bool
	XFrameOptions     string
	XContentType      bool
	CSP               bool
	XSSProtection     bool
	MissingHeaders    []string
	Risk              string
}

type CMSResult struct {
	Detected      bool
	Name          string
	Version       string
	VersionExposed bool
	Risk          string
}

type Finding struct {
	Severity    string // Critical, High, Medium, Low, Info
	Category    string
	Title       string
	Description string
	Remediation string
}
```

### 2. SSL証明書チェック

`crypto/tls` を使って SSL 証明書の有効性と残り日数をチェックします。

```go
func checkSSL(url string) SSLResult {
	result := SSLResult{Risk: "Info"}
	
	if strings.HasPrefix(url, "http://") {
		result.Enabled = false
		result.Risk = "Critical"
		return result
	}
	
	host := getHost(url)
	conn, err := tls.Dial("tcp", host+":443", &tls.Config{
		InsecureSkipVerify: false,
	})
	if err != nil {
		// 証明書に問題がある場合も情報を取得
		conn, err = tls.Dial("tcp", host+":443", &tls.Config{
			InsecureSkipVerify: true,
		})
		if err != nil {
			result.Enabled = false
			result.Risk = "Critical"
			return result
		}
		result.Risk = "High"
	}
	defer conn.Close()
	
	result.Enabled = true
	certs := conn.ConnectionState().PeerCertificates
	if len(certs) > 0 {
		cert := certs[0]
		result.ValidFrom = cert.NotBefore
		result.ValidUntil = cert.NotAfter
		result.DaysRemaining = int(time.Until(cert.NotAfter).Hours() / 24)
		result.Issuer = cert.Issuer.CommonName
		
		// TLSバージョンをチェック
		switch conn.ConnectionState().Version {
		case tls.VersionTLS13:
			result.Protocol = "TLS 1.3"
		case tls.VersionTLS12:
			result.Protocol = "TLS 1.2"
		case tls.VersionTLS11:
			result.Protocol = "TLS 1.1"
			result.Risk = "Medium"
		case tls.VersionTLS10:
			result.Protocol = "TLS 1.0"
			result.Risk = "High"
		}
		
		// 残り日数でリスク判定
		if result.DaysRemaining < 0 {
			result.Risk = "Critical"
		} else if result.DaysRemaining < 14 {
			result.Risk = "High"
		} else if result.DaysRemaining < 30 {
			result.Risk = "Medium"
		}
	}
	
	return result
}
```

### 3. HTTPセキュリティヘッダーチェック

重要なセキュリティヘッダーの有無をチェックします。

```go
func checkHeaders(url string) (HeadersResult, ServerInfoResult) {
	headers := HeadersResult{Risk: "Info"}
	server := ServerInfoResult{Risk: "Info"}
	
	client := &http.Client{
		Timeout: 10 * time.Second,
		CheckRedirect: func(req *http.Request, via []*http.Request) error {
			return http.ErrUseLastResponse
		},
	}
	
	resp, err := client.Get(url)
	if err != nil {
		headers.Risk = "Unknown"
		return headers, server
	}
	defer resp.Body.Close()
	
	// セキュリティヘッダーをチェック
	if resp.Header.Get("Strict-Transport-Security") != "" {
		headers.HSTS = true
	} else {
		headers.MissingHeaders = append(headers.MissingHeaders, "Strict-Transport-Security")
	}
	
	headers.XFrameOptions = resp.Header.Get("X-Frame-Options")
	if headers.XFrameOptions == "" {
		headers.MissingHeaders = append(headers.MissingHeaders, "X-Frame-Options")
	}
	
	if resp.Header.Get("X-Content-Type-Options") == "nosniff" {
		headers.XContentType = true
	} else {
		headers.MissingHeaders = append(headers.MissingHeaders, "X-Content-Type-Options")
	}
	
	if resp.Header.Get("Content-Security-Policy") != "" {
		headers.CSP = true
	} else {
		headers.MissingHeaders = append(headers.MissingHeaders, "Content-Security-Policy")
	}
	
	// リスク判定
	missingCount := len(headers.MissingHeaders)
	if missingCount >= 4 {
		headers.Risk = "High"
	} else if missingCount >= 2 {
		headers.Risk = "Medium"
	} else if missingCount >= 1 {
		headers.Risk = "Low"
	}
	
	// サーバー情報の露出チェック
	server.Server = resp.Header.Get("Server")
	server.XPoweredBy = resp.Header.Get("X-Powered-By")
	if server.Server != "" || server.XPoweredBy != "" {
		server.Exposed = true
		server.Risk = "Low"
	}
	
	return headers, server
}
```

### 4. CMS検出

HTMLのパターンマッチングでWordPressなどのCMSを検出します。

```go
func detectCMS(url string) CMSResult {
	result := CMSResult{Risk: "Info"}
	
	client := &http.Client{Timeout: 10 * time.Second}
	resp, err := client.Get(url)
	if err != nil {
		return result
	}
	defer resp.Body.Close()
	
	body, err := io.ReadAll(io.LimitReader(resp.Body, 1024*100))
	if err != nil {
		return result
	}
	bodyStr := string(body)
	
	// WordPress検出
	wpPatterns := []string{"wp-content", "wp-includes", "wp-json", "wordpress"}
	for _, pattern := range wpPatterns {
		if strings.Contains(strings.ToLower(bodyStr), pattern) {
			result.Detected = true
			result.Name = "WordPress"
			break
		}
	}
	
	// バージョン露出チェック
	if result.Name == "WordPress" {
		genRegex := regexp.MustCompile(`<meta name="generator" content="WordPress\s*([\d.]*)"`)
		matches := genRegex.FindStringSubmatch(bodyStr)
		if len(matches) > 1 && matches[1] != "" {
			result.Version = matches[1]
			result.VersionExposed = true
			result.Risk = "Medium"
		}
	}
	
	return result
}
```

### 5. DNS（SPF/DMARC）チェック

メールセキュリティ設定をDNSレコードから確認します。

```go
func checkDNS(url string) DNSResult {
	result := DNSResult{Risk: "Info"}
	host := getHost(url)
	
	// ドメイン抽出（サブドメイン除去）
	parts := strings.Split(host, ".")
	domain := host
	if len(parts) > 2 {
		domain = strings.Join(parts[len(parts)-2:], ".")
	}
	
	// MXレコードチェック
	mx, err := net.LookupMX(domain)
	if err == nil && len(mx) > 0 {
		result.HasMX = true
	}
	
	// SPFレコードチェック
	txt, err := net.LookupTXT(domain)
	if err == nil {
		for _, t := range txt {
			if strings.HasPrefix(t, "v=spf1") {
				result.HasSPF = true
				break
			}
		}
	}
	
	// DMARCレコードチェック
	dmarc, err := net.LookupTXT("_dmarc." + domain)
	if err == nil {
		for _, d := range dmarc {
			if strings.HasPrefix(d, "v=DMARC1") {
				result.HasDMARC = true
				break
			}
		}
	}
	
	// リスク判定
	if result.HasMX && !result.HasSPF && !result.HasDMARC {
		result.Risk = "High"
	} else if result.HasMX && (!result.HasSPF || !result.HasDMARC) {
		result.Risk = "Medium"
	}
	
	return result
}
```

### 6. 出力フォーマット

text / markdown / json / html の4形式に対応します。

```go
func formatOutput(r *ScanResult, format string) string {
	switch format {
	case "json":
		return formatJSON(r)
	case "markdown":
		return formatMarkdown(r)
	case "html":
		return formatHTML(r)
	default:
		return formatText(r)
	}
}
```

HTML出力では、CSSを埋め込んで見やすいレポートを生成します。

## 使い方

### インストール

```bash
go install github.com/geeknow112/biz-tools@latest
```

### 実行例

```bash
# 基本診断
biz-tools scan https://example.com

# Markdownでファイル出力
biz-tools scan https://example.com -o markdown -f report.md

# HTMLレポート作成
biz-tools scan https://example.com -o html -f report.html
```

## チェック項目一覧

| 項目 | 説明 | リスク判定 |
|------|------|----------|
| SSL証明書 | 有効性、残り日数、TLSバージョン | Critical〜Info |
| HSTS | HTTP Strict Transport Security | Medium |
| X-Frame-Options | クリックジャッキング対策 | Medium |
| CSP | Content Security Policy | Low |
| CMS検出 | WordPress等のバージョン露出 | Medium |
| サーバー情報 | Server/X-Powered-By ヘッダー | Low |
| SPF/DMARC | メールなりすまし対策 | High〜Medium |

## 注意事項

- このツールは **公開情報のみ** を使用した簡易診断です
- 本格的な脆弱性診断には **許可を得た上での** 専門ツールを使用してください
- 他者のサイトをスキャンする場合は **許可を取得** してください

## まとめ

Go + Cobra で実用的なセキュリティ簡易診断ツールを作りました。

- **標準ライブラリ** のみで SSL/TLS、DNS チェックが可能
- **複数の出力形式** に対応し、レポート作成が簡単
- 自社サイトの定期チェックや営業ツールとして活用可能

ソースコードは GitHub で公開しています：
https://github.com/geeknow112/biz-tools

## 関連記事

- [Go + Cobraで業務自動化CLIツールを作る](https://zenn.dev/devex12/articles/go-cobra-cli-automation)
