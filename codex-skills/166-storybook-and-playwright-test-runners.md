# StorybookのブラウザーテストとPlaywright Testを分けて導入する

## はじめに

StorybookでPlaywrightを使っていても、アプリケーションを開いて操作するPlaywright Testの設定が存在するとは限らない。blog-wordcloudでは、StorybookのVitestプロジェクトがChromiumを起動していた一方、専用のPlaywright設定とE2Eテストはなかった。そこで独立したランナーを追加した。

## 前提・環境

- 検証日: 2026年10月24日、Windows / PowerShell
- Runtime: Bun 1.3.13
- Frontend: React / Vite 8.3.4
- Storybook: 10.6.3、Vitest: 5.0.5
- Playwright / `@playwright/test`: 1.64.1
- 対象コミット: `90f28a9`

## やってみた結果

`app/frontend` で `bun run test:e2e` を実行すると、専用のViteサーバーとChromiumが起動し、カウンターのクリックテスト1件が成功するようになった。既存の `bun run test:storybook:coverage` も4件成功した。

これはランナーの動作確認である。ワードクラウドの表示・画像保存、CIでの専用ランナー実行、VRTは実装していない。

## 実装・検証

### フォルダー名より実体を読む

調査時に存在した `app/frontend/playwright@latest/` は、Playwright設定ではなく別のBun Reactテンプレートだった。独自の `package.json` には `dev`・`start`・`build` があり、Playwright Testの依存や実行コマンドはなかった。利用者の指定で削除し、専用のESLint除外も整理した。削除コミットは `842f241` である。どの操作でこの名前になったかは確認していない。

### ランナーをコマンドと検出範囲で分ける

| 用途 | コマンド | 検出対象 |
| --- | --- | --- |
| Storybook | `bun run test:storybook` | Storybook設定が選ぶstories |
| アプリの操作 | `bun run test:e2e` | `e2e/` のサイズ付きテスト |

`@playwright/test` を開発依存へ追加し、既存の `playwright` と1.64.1で揃えた。実行コマンドは次のとおりである。

```sh
# app/frontend で実行。workspace依存は app で導入済みとする。
bunx --no-install playwright install chromium
bun run test:e2e
```

### サーバーをテスト自身で起動する

[公式のwebServer設定](https://playwright.dev/docs/test-webserver)に沿って、次の構成にした。

```ts
const baseURL = "http://127.0.0.1:4173";

// playwright.config.ts の抜粋
use: { baseURL },
webServer: {
  command: "bun run dev --host 127.0.0.1 --port 4173 --strictPort",
  url: baseURL,
  reuseExistingServer: false,
},
```

テスト本体は `page.goto("/")` と書く。ポートが使用中なら別のポートへ移らず、既存サーバーも再利用しないため、別アプリケーションを試験することを避けられる。

### 型チェックとLintの対象も揃える

`playwright.config.ts` と `e2e/**/*.ts` を `tsconfig.node.json` の対象へ追加した。Playwrightテストには共通の静的解析を維持し、VitestとTesting Library専用のテストルールからは除外した。

最初のテストは `e2e/app.medium.test.ts` に置いた。ローカルのHTTPサーバーとブラウザープロセスを使い、外部サービスへ依存しないことを根拠にMediumとした。ファイル検出はSmall・Medium・Largeの命名に対応する。

## 検証結果

| 検証 | 結果 |
| --- | --- |
| `bun run test:e2e` | Chromium 1件成功 |
| `bun run test:storybook:coverage` | 4件成功 |
| `bun run typecheck` / `bun run lint` / `bun run build` | 成功 |
| 変更した設定・テストのBiomeチェック | 成功 |
| `bun install --frozen-lockfile` | 追加変更なしで成功 |
| Playwright導入差分の独立レビュー | 重大な問題なし |

Storybookのカバレッジは文・分岐・関数・行が各100%だった。この数値は報告されたStorybookの対象範囲に限られ、アプリ全体やPlaywrightテストのカバレッジではない。

## つまずいたところ

実行時には、既存のVite設定の `__dirname` に関する警告と、StorybookのMDXファイルがないという警告が出た。テストは成功しており、この作業ではそれらを修正していない。導入前の全体 `bun run code` で見つかった既存の整形問題も、対象ファイル単体のチェックとは分けて記録した。

## 学んだこと・次の課題

ブラウザーを起動する依存、テストを発見するランナー、型・Lintが見るファイルは別々に確認する。現行画面での操作テストが成功して初めて、設定ファイルが存在するだけの段階を越えられた。

次は実際のワードクラウド画面を実装し、選択したメンバーの画像表示と、保存ファイルの内容を確認する受け入れテストへ進む。

## 参考資料

- [Playwright: Web server](https://playwright.dev/docs/test-webserver)
- [Playwright導入コミット](https://github.com/kishimin/blog-wordcloud/commit/90f28a954ea64460c8d7126c55458c3af2f9de7c)
- [不要なテンプレートの削除](https://github.com/kishimin/blog-wordcloud/commit/842f241)
