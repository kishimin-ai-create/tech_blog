# Playwright の webServer タイムアウトエラーを CI とローカルで条件分岐して修正した

> 公開日: 2026-04-28

---

## エラー概要

`@smoke` タグが付いた Playwright の E2E テストを実行すると、次のエラーが出てテストが開始されなかった。

```
Error: Timed out waiting 60000ms from config.webServer.
```

Playwright の `webServer` オプションはテスト開始前にローカルサーバーを起動する機能を持つ。
このエラーは、指定した URL (`http://localhost:4173`) が制限時間内に応答しなかったことを意味する。

問題が発生していた `playwright.config.ts` の設定は次のとおりだった。

```ts
// 修正前
webServer: {
  command: 'npm run build && npm run preview',
  url: 'http://localhost:4173',
  reuseExistingServer: !process.env.CI,
},
```

---

## 原因

### デフォルトタイムアウト 60 秒を `tsc + vite build` が超過する

`npm run build` の中身は `tsc -b && vite build` だ。TypeScript の型チェックとバンドルを両方行うため、
環境によっては 60 秒を超えることがある。Playwright の `webServer.timeout` のデフォルト値は **60,000 ms**
であり、ビルドが終わらないまま制限時間を迎えると `vite preview` が起動する前に Playwright が中断してしまう。

### CI では `dist/` が既に存在するのに再ビルドしていた

CI の nightly ワークフロー (`ci-nightly.yml`) では、Playwright を実行する前に `npm run build` ステップが明示的に用意されている。

```yaml
- name: Build app
  run: npm run build

- name: Run Playwright (full suite, all browsers)
  run: npx playwright test --grep-invert "@visual"
```

つまり CI では `dist/` がすでに存在する状態で Playwright が起動するにもかかわらず、
`webServer.command` が再度 `npm run build` を実行していた。これが二重ビルドを引き起こし、
タイムアウトをさらに誘発しやすくしていた。

---

## 修正

修正は 3 つのコミットに分けて段階的に行われた。

### ステップ 1 — smoke テストをローカルアプリに向ける（コミット `1786dac`）

既存の smoke テストは外部の `https://playwright.dev/` を開いており、ローカルアプリの検証になっていなかった。
あわせて `webServer.command` に `npm run build` を追加し、`dist/` が存在しなくてもテストが実行できるようにした。

```ts
// e2e/example.spec.ts（修正後）
test('app loads @smoke', async ({ page }) => {
  const response = await page.goto('/');
  expect(response?.status()).toBeLessThan(400);
  await expect(page.locator('body')).toBeVisible();
});
```

### ステップ 2 — タイムアウトを 120 秒に引き上げる（コミット `51d9696`）

`tsc -b + vite build` が 60 秒のデフォルト制限を超えるため、`timeout: 120_000` を追加した。

```ts
webServer: {
  command: 'npm run build && npm run preview',
  url: 'http://localhost:4173',
  reuseExistingServer: !process.env.CI,
  timeout: 120_000,   // 追加
},
```

### ステップ 3 — CI とローカルでコマンドを条件分岐する（コミット `9268a7f`）

CI では `dist/` がワークフローのビルドステップで生成済みなので、`npm run preview` だけで十分だ。
ローカルでは `dist/` が存在しない場合もあるため、`npm run build && npm run preview` を維持する。
`process.env.CI` を使って条件分岐させた。

```ts
// 修正後（最終形）
webServer: {
  command: process.env.CI
    ? 'npm run preview'
    : 'npm run build && npm run preview',
  url: 'http://localhost:4173',
  reuseExistingServer: !process.env.CI,
  timeout: 120_000,
},
```

この設定により、以下のように動作が分かれる。

| 実行環境 | `command` | `reuseExistingServer` | `timeout` |
|---|---|---|---|
| CI | `npm run preview` のみ | `false`（常に新規起動） | 120 秒 |
| ローカル | `npm run build && npm run preview` | `true`（既存サーバーを再利用） | 120 秒 |

---

## まとめ

Playwright の `webServer.timeout` のデフォルト値（60 秒）は、ビルドステップを含むコマンドでは不足することがある。
今回のように `tsc + vite build` を `webServer.command` に含める場合は `timeout` を明示的に引き上げる必要がある。

また、CI では前段のワークフローステップでビルド済みの成果物が存在するケースが多い。
`process.env.CI` で条件分岐することで、CI での無駄な再ビルドを排除しつつ、ローカルでは自動ビルドを維持する
という両立が実現できる。

**対処のチェックポイント**

- `webServer` に `npm run build` 相当のコマンドを含めるときは `timeout` を必ず設定する
- CI の実行順序を確認し、`dist/` が存在するなら `webServer.command` でビルドを省略できる
- `reuseExistingServer: !process.env.CI` パターンは CI で常にフレッシュなサーバーを保証するための定石
