# Playwright の baseURL をハードコードしていたバグを環境変数対応に修正した

## 対象読者

- Playwright を使った E2E テストを CI に組み込んでいるエンジニア
- ステージング環境など複数の環境に対して同じテストスイートを実行したいエンジニア

---

## 問題の背景

`frontend/playwright.config.ts` において、Playwright の `baseURL` が次のようにハードコードされていた。

```ts
use: {
  baseURL: "http://localhost:3000",
  trace: "on-first-retry",
},
```

この状態では、`baseURL` を変更する手段が存在しない。ローカル開発中は問題が起きないが、CI でステージング環境などの別オリジンに向けて E2E テストを実行したい場合、設定ファイルを直接書き換えるしかなかった。

---

## 問題がなぜ困るか

Playwright はテスト内で `page.goto("/")` のような相対パスを使うことができ、その際に `baseURL` が補完される。`baseURL` がコードに固定されていると以下の問題が起きる。

- **CI で向き先を変えられない**  
  ステージング URL を環境変数でインジェクトしても、設定ファイルが読まない。
- **環境ごとにリポジトリ変更が必要になる**  
  環境に応じた設定変更をコミットする運用は、レビューコストが増え事故の温床になる。
- **他の設定項目との一貫性がない**  
  同じ `playwright.config.ts` 内で `forbidOnly`・`retries`・`workers` はすでに `process.env.CI` を参照している。`baseURL` だけがハードコードになっており、設定の一貫性が壊れていた。

---

## 修正内容

変更は 1 行のみ。`process.env.PLAYWRIGHT_BASE_URL` が設定されていればそれを使い、未設定の場合はローカル開発用のデフォルト値にフォールバックする。

```diff
- baseURL: "http://localhost:3000",
+ baseURL: process.env.PLAYWRIGHT_BASE_URL ?? "http://localhost:3000",
```

`??`（Nullish Coalescing）演算子を使っているため、環境変数が空文字列 `""` の場合はフォールバックせずそのまま空文字列が渡る点には注意が必要だが、URL として無効な値は Playwright 実行時にエラーになるため検知はできる。

---

## 修正後の設定ファイル全体

```ts
import { defineConfig, devices } from "@playwright/test";

export default defineConfig({
  testDir: "./e2e",
  fullyParallel: true,
  forbidOnly: !!process.env.CI,
  retries: process.env.CI ? 2 : 0,
  workers: process.env.CI ? 1 : undefined,
  reporter: "html",
  use: {
    baseURL: process.env.PLAYWRIGHT_BASE_URL ?? "http://localhost:3000",
    trace: "on-first-retry",
  },
  projects: [
    {
      name: "chromium",
      use: { ...devices["Desktop Chrome"] },
    },
    {
      name: "webkit",
      use: { ...devices["Desktop Safari"] },
    },
  ],
  webServer: {
    command: "bun run dev",
    url: "http://localhost:3000",
    reuseExistingServer: !process.env.CI,
  },
});
```

`webServer.url` は引き続き `"http://localhost:3000"` のままになっている。これはローカル開発時にサーバーの起動確認先として使われる URL であり、`baseURL` とは別の役割を持つ。CI でステージング環境に向ける場合は `webServer` ブロック自体を無効化するか、別途考慮が必要な点として残っている。

---

## 検証

修正後に `bun run typecheck` を実行し、TypeScript の型エラーがないことを確認した。`process.env.PLAYWRIGHT_BASE_URL` は `string | undefined` 型であり、`??` による `string` フォールバックで `baseURL` が `string` 型に確定するため、型検査も問題なく通過する。

---

## まとめ

| 項目 | 修正前 | 修正後 |
|------|--------|--------|
| `baseURL` の値 | `"http://localhost:3000"` (固定) | `process.env.PLAYWRIGHT_BASE_URL ?? "http://localhost:3000"` |
| CI での向き先変更 | 不可 | 環境変数 `PLAYWRIGHT_BASE_URL` で指定可能 |
| ローカル開発 | 変わらず動作する | デフォルト値へのフォールバックで変わらず動作する |

E2E テスト設定における `baseURL` のハードコードは見落としやすいが、CI でのマルチ環境テストを妨げる典型的な問題のひとつだ。今回のように環境変数とデフォルト値の組み合わせで対応することで、ローカル開発への影響をゼロに保ちながら柔軟性を確保できる。
