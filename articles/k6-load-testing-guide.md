---
title: "なぜ今k6なのか？JMeterからの移行を考えるべき理由"
emoji: "⚡"
type: "tech"
topics: ["k6", "loadtesting", "javascript", "performance", "jmeter"]
published: true
---

## 負荷テストツールの選定、どうしてますか？

チームで負荷テストを導入しようとしたとき、真っ先に候補に上がるのはJMeterでしょう。実績があり、情報も豊富。でも実際に運用してみると、こんな課題に直面しませんか？

- テストシナリオがXMLで可読性が低い
- GUIで作ったはずなのに、結局XMLを手で直す羽目に
- バージョン管理が辛い（差分が追えない）
- CI/CDへの組み込みが面倒

**k6**はこれらの課題を解決するために設計されたモダンな負荷テストツールです。Grafana Labsが開発しており、**JavaScriptでテストを記述**できるのが最大の特徴。

この記事では、JMeterとの比較を交えながら、k6を選ぶべき理由を掘り下げます。

## k6のアーキテクチャと設計思想

k6は「開発者が負荷テストを書く」ことを前提に設計されています。

### Code as Configuration

```javascript
export const options = {
  stages: [
    { duration: '1m', target: 50 },
    { duration: '3m', target: 50 },
    { duration: '1m', target: 0 },
  ],
  thresholds: {
    http_req_duration: ['p(95)<500'],
  },
};
```

テスト設定がコードなので：
- **レビュー可能** - PRで「この負荷設定で大丈夫？」と議論できる
- **再現可能** - 同じスクリプトを実行すれば同じテストができる
- **差分追跡** - 「前回から何を変えたか」がgit diffで分かる

### Go製のランタイム

k6本体はGoで実装されており、JavaScriptはgoja（Go製のJSエンジン）で実行されます。

これにより：
- **軽量** - JMeterと比較してメモリ消費が1/10以下
- **高性能** - 1台のマシンで数千VUを生成可能
- **シングルバイナリ** - 依存関係なしでどこでも実行可能

## JMeterとの比較：何が違うのか

## インストール

```bash
# Mac
brew install k6

# Windows
choco install k6

# Docker
docker run --rm -i grafana/k6 run - <script.js
```

## 基本的なテストスクリプト

### 最小構成

```javascript
// script.js
import http from 'k6/http';
import { check, sleep } from 'k6';

export default function () {
  const res = http.get('https://test.k6.io');
  
  check(res, {
    'status is 200': (r) => r.status === 200,
    'response time < 500ms': (r) => r.timings.duration < 500,
  });
  
  sleep(1);
}
```

実行:

```bash
k6 run script.js
```

### 仮想ユーザー数と実行時間の指定

```javascript
import http from 'k6/http';
import { sleep } from 'k6';

export const options = {
  vus: 10,        // 仮想ユーザー数
  duration: '30s', // 実行時間
};

export default function () {
  http.get('https://test.k6.io');
  sleep(1);
}
```

コマンドラインで上書きも可能:

```bash
k6 run --vus 50 --duration 1m script.js
```

## 負荷パターン（Stages）

実際の負荷テストでは、徐々に負荷を上げて（ランプアップ）、維持し、徐々に下げる（ランプダウン）パターンが一般的です。

```javascript
import http from 'k6/http';
import { sleep } from 'k6';

export const options = {
  stages: [
    { duration: '1m', target: 50 },   // 1分かけて50ユーザーまで増加
    { duration: '3m', target: 50 },   // 3分間50ユーザーを維持
    { duration: '1m', target: 100 },  // 1分かけて100ユーザーまで増加
    { duration: '3m', target: 100 },  // 3分間100ユーザーを維持
    { duration: '2m', target: 0 },    // 2分かけて0ユーザーまで減少
  ],
};

export default function () {
  http.get('https://api.example.com/endpoint');
  sleep(1);
}
```

## テストタイプ別のシナリオ

### スモークテスト（動作確認）

```javascript
export const options = {
  vus: 1,
  duration: '1m',
  thresholds: {
    http_req_failed: ['rate<0.01'],  // エラー率1%未満
  },
};
```

### 負荷テスト（Load Test）

```javascript
export const options = {
  stages: [
    { duration: '5m', target: 100 },
    { duration: '10m', target: 100 },
    { duration: '5m', target: 0 },
  ],
  thresholds: {
    http_req_duration: ['p(95)<500'],  // 95%のリクエストが500ms未満
  },
};
```

### ストレステスト（限界テスト）

```javascript
export const options = {
  stages: [
    { duration: '2m', target: 100 },
    { duration: '5m', target: 100 },
    { duration: '2m', target: 200 },
    { duration: '5m', target: 200 },
    { duration: '2m', target: 300 },
    { duration: '5m', target: 300 },
    { duration: '2m', target: 0 },
  ],
};
```

### スパイクテスト（急激な負荷）

```javascript
export const options = {
  stages: [
    { duration: '10s', target: 100 },  // 急激に100ユーザー
    { duration: '1m', target: 100 },
    { duration: '10s', target: 1000 }, // さらに急激に1000ユーザー
    { duration: '3m', target: 1000 },
    { duration: '10s', target: 100 },
    { duration: '1m', target: 100 },
    { duration: '10s', target: 0 },
  ],
};
```

## 実践的なスクリプト例

### APIテスト（POST + 認証）

```javascript
import http from 'k6/http';
import { check, sleep } from 'k6';

const BASE_URL = 'https://api.example.com';

export const options = {
  vus: 20,
  duration: '5m',
  thresholds: {
    http_req_failed: ['rate<0.01'],
    http_req_duration: ['p(95)<1000'],
  },
};

// セットアップ（1回だけ実行）
export function setup() {
  const loginRes = http.post(`${BASE_URL}/auth/login`, JSON.stringify({
    email: 'test@example.com',
    password: 'password123',
  }), {
    headers: { 'Content-Type': 'application/json' },
  });
  
  return { token: loginRes.json('access_token') };
}

// メインテスト（繰り返し実行）
export default function (data) {
  const headers = {
    'Content-Type': 'application/json',
    'Authorization': `Bearer ${data.token}`,
  };
  
  // ユーザー一覧取得
  const usersRes = http.get(`${BASE_URL}/users`, { headers });
  check(usersRes, {
    'users status 200': (r) => r.status === 200,
  });
  
  // ユーザー作成
  const createRes = http.post(`${BASE_URL}/users`, JSON.stringify({
    name: `User_${Date.now()}`,
    email: `user_${Date.now()}@example.com`,
  }), { headers });
  check(createRes, {
    'create status 201': (r) => r.status === 201,
  });
  
  sleep(1);
}
```

### 複数エンドポイントのシナリオ

```javascript
import http from 'k6/http';
import { check, sleep, group } from 'k6';

const BASE_URL = 'https://ecommerce.example.com';

export const options = {
  vus: 50,
  duration: '10m',
};

export default function () {
  // トップページ閲覧
  group('Browse Homepage', () => {
    const res = http.get(`${BASE_URL}/`);
    check(res, { 'homepage loaded': (r) => r.status === 200 });
    sleep(2);
  });
  
  // 商品検索
  group('Search Products', () => {
    const res = http.get(`${BASE_URL}/search?q=laptop`);
    check(res, { 'search results': (r) => r.status === 200 });
    sleep(1);
  });
  
  // 商品詳細
  group('View Product', () => {
    const res = http.get(`${BASE_URL}/products/123`);
    check(res, { 'product loaded': (r) => r.status === 200 });
    sleep(3);
  });
  
  // カートに追加
  group('Add to Cart', () => {
    const res = http.post(`${BASE_URL}/cart`, JSON.stringify({
      productId: 123,
      quantity: 1,
    }), {
      headers: { 'Content-Type': 'application/json' },
    });
    check(res, { 'added to cart': (r) => r.status === 200 });
    sleep(1);
  });
}
```

## Thresholds（合否基準）

テストの合否を自動判定できます。CI/CDに組み込む際に必須。

```javascript
export const options = {
  thresholds: {
    // 全リクエスト
    http_req_duration: ['p(95)<500', 'p(99)<1000'],
    http_req_failed: ['rate<0.01'],
    
    // 特定のタグ
    'http_req_duration{name:login}': ['p(95)<1000'],
    'http_req_duration{name:search}': ['p(95)<300'],
    
    // カスタムメトリクス
    'checks': ['rate>0.99'],
  },
};
```

Threshold失敗時は終了コード1を返すので、CIで失敗として扱えます。

```bash
k6 run script.js || echo "Performance test failed!"
```

## 結果の出力

### JSON出力

```bash
k6 run --out json=results.json script.js
```

### CSV出力

```bash
k6 run --out csv=results.csv script.js
```

### InfluxDB + Grafana連携

```bash
k6 run --out influxdb=http://localhost:8086/k6 script.js
```

Grafanaでリアルタイムに可視化できます。

## k6 vs JMeter vs Locust

| 項目 | k6 | JMeter | Locust |
|---|---|---|---|
| 言語 | JavaScript | XML/GUI | Python |
| 学習コスト | 低 | 高 | 中 |
| リソース消費 | 低 | 高 | 中 |
| CI/CD統合 | 容易 | 複雑 | 容易 |
| プロトコル | HTTP, WS, gRPC | 豊富 | HTTP |
| 分散実行 | k6 Cloud | 手動構築 | 組み込み |

### JMeterを選ぶべきケース

- JDBC、JMS、LDAPなど非HTTPプロトコルが必要
- 既存のJMeterスクリプト資産がある
- 非エンジニアがGUIでテストを作成する必要がある

### k6を選ぶべきケース

- 開発チームがJavaScript/TypeScriptに慣れている
- CI/CDパイプラインに負荷テストを組み込みたい
- テストコードをレビュー・管理したい
- 軽量に素早くテストを回したい

## k6の弱点も知っておく

万能ではありません：

- **非HTTPプロトコルは弱い** - JDBCやJMSは非対応（拡張で対応可能だが手間）
- **GUIがない** - コードが書けないメンバーには辛い
- **分散実行は有料** - k6 Cloudを使うか、自前で構築が必要

## 議論：チームへの導入をどう進めるか

k6を導入する際のハードルは「JMeterに慣れたメンバーの抵抗」かもしれません。

実際に導入した経験から、以下のアプローチが有効でした：

1. **小さく始める** - まずはスモークテストから
2. **既存JMeterと併用** - いきなり全移行しない
3. **CI統合のメリットを見せる** - 「PR毎に自動で負荷テストが走る」のインパクト

皆さんのチームではどうでしょうか？JMeterからの移行経験がある方、コメントで教えてください。

## まとめ

k6は「負荷テストもコードで管理する」という現代的なアプローチのツールです。

- JavaScriptで書ける → 開発者の学習コスト低
- コードベース → レビュー・バージョン管理可能
- CLI駆動 → CI/CD統合が容易
- 軽量 → 1台で数千VU生成可能

JMeterの「XML地獄」「差分が追えない」「CIに組み込みづらい」といった課題を感じているなら、k6は有力な選択肢です。

## 参考リンク

- [k6 公式ドキュメント](https://k6.io/docs/)
- [k6 GitHub](https://github.com/grafana/k6)
- [k6 Examples](https://github.com/grafana/k6/tree/master/examples)
- [JMeterからk6への移行ガイド](https://k6.io/docs/testing-guides/migrating-from-jmeter/)
