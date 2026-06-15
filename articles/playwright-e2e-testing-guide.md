---
title: "【2026年版】Playwright E2Eテスト入門 - Cypressから乗り換えた理由と実践ガイド"
emoji: "🎭"
type: "tech"
topics: ["playwright", "e2e", "testing", "javascript", "typescript"]
published: false
---

## この記事で得られること

- Playwright の**特徴と他ツールとの比較**
- プロジェクトへの**導入手順**
- 実践的な**テストコードの書き方**
- CI/CD への組み込み方法
- 実際にハマった**トラブルシューティング**

## なぜ Playwright なのか

E2E テストツールは選択肢が多いですが、2026年現在 Playwright を選ぶ理由は明確です。

### 主要ツール比較

| 項目 | Playwright | Cypress | Selenium |
|------|------------|---------|----------|
| ブラウザ対応 | Chromium, Firefox, WebKit | Chromium系のみ | 全ブラウザ |
| 実行速度 | ◎ 高速 | ○ 普通 | △ 遅い |
| 並列実行 | ◎ 標準対応 | △ 有料 | ○ 設定必要 |
| モバイル | ◎ エミュレーション | △ 限定的 | ○ Appium連携 |
| 学習コスト | ○ 中程度 | ◎ 低い | △ 高い |
| メンテナ | Microsoft | Cypress社 | Selenium HQ |

### Cypress から乗り換えた理由

1. **Safari テストが必要になった** - Cypress は WebKit 非対応
2. **並列実行が無料** - Cypress Dashboard は有料
3. **API テストも統合できる** - request context が便利
4. **トレース機能が強力** - デバッグが圧倒的に楽

## 環境構築

### インストール

```bash
# 新規プロジェクト
npm init playwright@latest

# 既存プロジェクトに追加
npm install -D @playwright/test
npx playwright install
```

`npm init playwright@latest` を実行すると対話形式でセットアップが進みます。

```
? Do you want to use TypeScript or JavaScript? TypeScript
? Where to put your end-to-end tests? tests
? Add a GitHub Actions workflow? Yes
? Install Playwright browsers? Yes
```

### 生成されるファイル構成

```
project/
├── tests/
│   └── example.spec.ts    # サンプルテスト
├── playwright.config.ts    # 設定ファイル
├── package.json
└── .github/
    └── workflows/
        └── playwright.yml  # CI設定
```

## 基本的なテストの書き方

### 最初のテスト

```typescript
// tests/login.spec.ts
import { test, expect } from '@playwright/test';

test.describe('ログイン機能', () => {
  test('正常にログインできる', async ({ page }) => {
    // 1. ページにアクセス
    await page.goto('https://example.com/login');

    // 2. フォーム入力
    await page.fill('input[name="email"]', 'test@example.com');
    await page.fill('input[name="password"]', 'password123');

    // 3. ログインボタンをクリック
    await page.click('button[type="submit"]');

    // 4. ダッシュボードに遷移することを確認
    await expect(page).toHaveURL(/.*dashboard/);
    await expect(page.locator('h1')).toContainText('ダッシュボード');
  });

  test('パスワードが間違っているとエラーが表示される', async ({ page }) => {
    await page.goto('https://example.com/login');
    await page.fill('input[name="email"]', 'test@example.com');
    await page.fill('input[name="password"]', 'wrong-password');
    await page.click('button[type="submit"]');

    // エラーメッセージが表示されることを確認
    await expect(page.locator('.error-message')).toBeVisible();
    await expect(page.locator('.error-message')).toContainText('パスワードが正しくありません');
  });
});
```

### Locator の選び方

Playwright では要素の特定方法が複数あります。**優先順位**を意識すると保守性が上がります。

```typescript
// ✅ 推奨: role, label, placeholder
await page.getByRole('button', { name: 'ログイン' });
await page.getByLabel('メールアドレス');
await page.getByPlaceholder('example@mail.com');

// ○ 許容: data-testid
await page.getByTestId('login-button');

// △ 非推奨: CSS セレクタ（変更に弱い）
await page.locator('.btn-primary');
await page.locator('#login-form > button');
```

### Page Object Model

テストが増えてきたら Page Object Model でリファクタリングします。

```typescript
// pages/LoginPage.ts
import { Page, Locator } from '@playwright/test';

export class LoginPage {
  readonly page: Page;
  readonly emailInput: Locator;
  readonly passwordInput: Locator;
  readonly submitButton: Locator;
  readonly errorMessage: Locator;

  constructor(page: Page) {
    this.page = page;
    this.emailInput = page.getByLabel('メールアドレス');
    this.passwordInput = page.getByLabel('パスワード');
    this.submitButton = page.getByRole('button', { name: 'ログイン' });
    this.errorMessage = page.locator('.error-message');
  }

  async goto() {
    await this.page.goto('/login');
  }

  async login(email: string, password: string) {
    await this.emailInput.fill(email);
    await this.passwordInput.fill(password);
    await this.submitButton.click();
  }
}
```

```typescript
// tests/login.spec.ts
import { test, expect } from '@playwright/test';
import { LoginPage } from '../pages/LoginPage';

test('正常にログインできる', async ({ page }) => {
  const loginPage = new LoginPage(page);
  await loginPage.goto();
  await loginPage.login('test@example.com', 'password123');
  
  await expect(page).toHaveURL(/.*dashboard/);
});
```

## 設定ファイルのカスタマイズ

```typescript
// playwright.config.ts
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  testDir: './tests',
  
  // 並列実行
  fullyParallel: true,
  workers: process.env.CI ? 2 : undefined,
  
  // リトライ設定
  retries: process.env.CI ? 2 : 0,
  
  // レポート
  reporter: [
    ['html', { open: 'never' }],
    ['json', { outputFile: 'test-results.json' }],
  ],
  
  use: {
    // ベースURL
    baseURL: 'http://localhost:3000',
    
    // トレース（デバッグ用）
    trace: 'on-first-retry',
    
    // スクリーンショット
    screenshot: 'only-on-failure',
    
    // ビデオ
    video: 'retain-on-failure',
  },

  // ブラウザ設定
  projects: [
    {
      name: 'chromium',
      use: { ...devices['Desktop Chrome'] },
    },
    {
      name: 'firefox',
      use: { ...devices['Desktop Firefox'] },
    },
    {
      name: 'webkit',
      use: { ...devices['Desktop Safari'] },
    },
    // モバイル
    {
      name: 'mobile-chrome',
      use: { ...devices['Pixel 5'] },
    },
  ],

  // 開発サーバー自動起動
  webServer: {
    command: 'npm run dev',
    url: 'http://localhost:3000',
    reuseExistingServer: !process.env.CI,
  },
});
```

## 実行コマンド

```bash
# 全テスト実行
npx playwright test

# 特定ファイルのみ
npx playwright test tests/login.spec.ts

# 特定ブラウザのみ
npx playwright test --project=chromium

# UI モード（デバッグに便利）
npx playwright test --ui

# headed モード（ブラウザ表示）
npx playwright test --headed

# デバッグモード
npx playwright test --debug

# レポート表示
npx playwright show-report
```

## ハマったポイントと解決策

### 1. 要素が見つからない

**症状**: `locator.click: Target closed` や `Timeout`

**原因と解決**:

```typescript
// ❌ NG: 要素が表示される前にクリック
await page.click('.dynamic-button');

// ✅ OK: 要素が表示されるまで待機
await page.locator('.dynamic-button').waitFor({ state: 'visible' });
await page.locator('.dynamic-button').click();

// ✅ さらに良い: expect で待機を兼ねる
await expect(page.locator('.dynamic-button')).toBeVisible();
await page.locator('.dynamic-button').click();
```

### 2. iframe 内の要素操作

```typescript
// iframe を取得
const frame = page.frameLocator('iframe[name="payment"]');

// iframe 内の要素を操作
await frame.locator('input[name="card-number"]').fill('4242424242424242');
```

### 3. ファイルアップロード

```typescript
// 単一ファイル
await page.setInputFiles('input[type="file"]', 'path/to/file.pdf');

// 複数ファイル
await page.setInputFiles('input[type="file"]', [
  'path/to/file1.pdf',
  'path/to/file2.pdf',
]);
```

### 4. 認証状態の保持

毎回ログインするのは非効率。認証状態を保存して再利用します。

```typescript
// playwright.config.ts
export default defineConfig({
  projects: [
    // 認証用（セットアップ）
    { name: 'setup', testMatch: /.*\.setup\.ts/ },
    
    // 認証済み状態でテスト
    {
      name: 'chromium',
      use: { 
        ...devices['Desktop Chrome'],
        storageState: 'playwright/.auth/user.json',
      },
      dependencies: ['setup'],
    },
  ],
});
```

```typescript
// tests/auth.setup.ts
import { test as setup, expect } from '@playwright/test';

const authFile = 'playwright/.auth/user.json';

setup('authenticate', async ({ page }) => {
  await page.goto('/login');
  await page.fill('input[name="email"]', 'test@example.com');
  await page.fill('input[name="password"]', 'password123');
  await page.click('button[type="submit"]');
  
  await expect(page).toHaveURL(/.*dashboard/);
  
  // 認証状態を保存
  await page.context().storageState({ path: authFile });
});
```

## GitHub Actions 連携

```yaml
# .github/workflows/playwright.yml
name: Playwright Tests

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          
      - name: Install dependencies
        run: npm ci
        
      - name: Install Playwright Browsers
        run: npx playwright install --with-deps
        
      - name: Run Playwright tests
        run: npx playwright test
        
      - uses: actions/upload-artifact@v4
        if: always()
        with:
          name: playwright-report
          path: playwright-report/
          retention-days: 30
```

## まとめ

| ポイント | 内容 |
|----------|------|
| ツール選定 | Safari対応、並列実行無料、トレース機能で Playwright |
| Locator | role, label 優先。CSS セレクタは最終手段 |
| 設計 | Page Object Model で保守性向上 |
| デバッグ | `--ui` モードとトレース機能を活用 |
| CI | 認証状態を保存してテスト効率化 |

E2E テストは導入コストがかかりますが、一度整備すればリグレッション防止に絶大な効果があります。

Playwright なら学習コストも低く、公式ドキュメントも充実しているのでぜひ試してみてください。

## 参考リンク

- [Playwright 公式ドキュメント](https://playwright.dev/)
- [Playwright GitHub](https://github.com/microsoft/playwright)
- [Best Practices](https://playwright.dev/docs/best-practices)
